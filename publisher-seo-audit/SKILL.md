---
name: publisher-seo-audit
description: Vollständiger SEO-Audit-Workflow für News-Publisher (Zeitungen, Magazine, Nachrichtenportale). Analysiert Seitenstruktur, Scope und Disziplinen (S/M/L), crawlt über SF MCP oder Fallback-Crawler, kann Chrome DevTools MCP und Lighthouse einbinden. Nur automatisch prüfbare Kriterien. Use when asked to run a publisher SEO audit, check a news site's technical SEO, or audit bild.de, welt.de, or any German news publisher.
tools: Read, Edit, Write, Bash
---

# Publisher SEO Audit

Strukturierter SEO-Audit für News-Publisher mit täglich 15–300 neuen Artikeln. Publisher-SEO unterscheidet sich fundamental von klassischem SEO: kein Contentplan, Chronistenpflicht statt Löschstrategie, drei Traffic-Kanäle (Search, Discover, News), Channelizer-Inhalte, Republishing und tausende neue URLs täglich.

**Reihenfolge ist zwingend.** Phase 0 (Seitenstruktur) nie überspringen.
**Nur automatisch prüfbare Kriterien.** Manuelle Checks stehen als Checkliste am Ende.

---

## Workflow-Übersicht

| Phase | Was | Ergebnis |
|---|---|---|
| **0** | Seitenstruktur-Analyse (Sitemap-basiert) | Struktur-Übersicht → Diskussion |
| **1** | Scope-Auswahl | URL-Liste für den Audit |
| **2** | Disziplinen-Konfiguration (S / M / L / Custom) | Konfiguration bestätigt |
| **3** | Crawler-Setup (SF MCP / Chrome DevTools MCP / Fallback) | Crawl-Daten vorhanden |
| **4** | Audit-Disziplinen | Findings-Daten |
| **5** | HTML-Dashboard erzeugen | `~/Downloads/publisher-seo-audit-{domain}-{datum}.html` |

---

## Phase 0: Seitenstruktur-Analyse

### 0.1 Sitemaps entdecken

```python
import requests, re, json
from bs4 import BeautifulSoup
from urllib.parse import urlparse
from collections import Counter, defaultdict

UA = "PublisherSEOAudit/1.0 (Mozilla Firefox)"

def fetch_xml(url: str) -> BeautifulSoup | None:
    try:
        r = requests.get(url, timeout=12, headers={"User-Agent": UA})
        r.raise_for_status()
        return BeautifulSoup(r.content, "xml")
    except Exception as e:
        print(f"  ! {url}: {e}")
        return None

def discover_sitemaps(base_url: str) -> dict:
    try:
        robots = requests.get(f"{base_url}/robots.txt", timeout=8,
                              headers={"User-Agent": UA}).text
    except Exception:
        robots = ""

    sitemap_urls = re.findall(r"(?i)Sitemap:\s*(\S+)", robots)
    if not sitemap_urls:
        sitemap_urls = [
            f"{base_url}/sitemap.xml",
            f"{base_url}/sitemap-index.xml",
            f"{base_url}/sitemap-news.xml",
        ]

    sitemaps = {"index": [], "news": [], "other": [], "robots_text": robots}

    for url in sitemap_urls:
        soup = fetch_xml(url)
        if not soup:
            continue
        if soup.find("sitemapindex"):
            for loc in soup.find_all("loc"):
                child_url = loc.text.strip()
                child = fetch_xml(child_url)
                if not child:
                    continue
                sitemaps["index"].append(child_url)
                if "news" in child_url.lower() or child.find("news:news"):
                    sitemaps["news"].append({"url": child_url, "soup": child})
                else:
                    sitemaps["other"].append({"url": child_url, "soup": child})
        elif soup.find("news:news"):
            sitemaps["news"].append({"url": url, "soup": soup})
        else:
            sitemaps["other"].append({"url": url, "soup": soup})

    return sitemaps
```

### 0.2 URL-Muster analysieren

```python
def analyze_url_structure(sitemaps: dict) -> dict:
    all_urls = []
    for sm in sitemaps["news"] + sitemaps["other"]:
        soup = sm.get("soup")
        if soup:
            all_urls.extend(loc.text.strip() for loc in soup.find_all("loc"))

    PATTERNS = {
        "autor":       r"/autor[en]?/|/author/|/redakteur/",
        "thema":       r"/thema[s]?/|/topic[s]?/|/tag[s]?/|/suchwort/|/stichwort/",
        "advertorial": r"/advertorial/|/anzeige/|/sponsored/|/native-ad/|/partner-content/",
        "galerie":     r"/fotostrecke|/galerie|/bilder-|/bildergalerie",
        "video":       r"/video[s]?/|/clip[s]?/",
        "live":        r"/live[blog]?/|/ticker/|/liveticker/",
        "amp":         r"/amp/|\.amp\b|[?&]amp=1",
        "service":     r"/ratgeber/|/service/|/vergleich/",
    }

    classified = defaultdict(list)
    for url in all_urls:
        matched = False
        for typ, pat in PATTERNS.items():
            if re.search(pat, url, re.I):
                classified[typ].append(url)
                matched = True
                break
        if not matched:
            classified["artikel"].append(url)

    top_ressorts = Counter(
        urlparse(u).path.strip("/").split("/")[0]
        for u in classified["artikel"]
        if urlparse(u).path.strip("/")
    ).most_common(15)

    domains = Counter(urlparse(u).netloc for u in all_urls)

    news_count = 0
    for sm in sitemaps["news"]:
        s = sm.get("soup")
        if s:
            news_count += len(s.find_all("url"))

    return {
        "total_urls":        len(all_urls),
        "news_sitemap_count": news_count,
        "classified":        {k: len(v) for k, v in classified.items()},
        "classified_full":   dict(classified),
        "top_ressorts":      top_ressorts,
        "domains":           dict(domains),
        "all_urls":          all_urls,
    }
```

### 0.3 Struktur ausgeben — Pause + User-Diskussion

```
╔══════════════════════════════════════════════════════════╗
  Seitenstruktur — {domain}
╚══════════════════════════════════════════════════════════╝

URLs gesamt:    {total_urls}
News-Sitemap:   {news_sitemap_count} Artikel (letzte 48h)

URL-Typen:
  Artikel:        {n:>6}
  Advertorials:   {n:>6}   ← URL-basiert erkannt
  Autorseiten:    {n:>6}
  Themenseiten:   {n:>6}
  Galerien:       {n:>6}
  Videos:         {n:>6}
  Live-Ticker:    {n:>6}
  AMP-URLs:       {n:>6}
  Service/Ratg:   {n:>6}

Top-Ressorts:  /{ressort}/  {n} Artikel

Domains/Subdomains: {domains}
→ Channelizer-Verdacht: [Sub-Domains mit >5 URLs]
```

**Zwingend pausieren und auf User-Feedback zur Struktur warten.**

---

## Phase 1: Scope-Auswahl

```
Welchen Teil soll der Audit abdecken?

  [1] Aktuelle Artikel   — News-Sitemap letzte 48h ({n} URLs) → schnell, aktuell
  [2] Gesamte Site       — alle {n} URLs → empfiehlt SF MCP (ohne: max. 500)
  [3] Ressort            — z.B. /sport/ oder /politik/
  [4] Nur Autorseiten    — {n} URLs
  [5] Nur Themenseiten   — {n} URLs
  [6] Nur Advertorials   — {n} URLs
  [7] Custom             — eigene URL-Liste
```

```python
def build_scope_urls(choice: str, structure: dict, ressort: str = None, limit: int = 500) -> list[str]:
    cf = structure["classified_full"]
    if choice == "1":
        return cf.get("artikel", [])[:limit]
    elif choice == "2":
        return structure["all_urls"]
    elif choice == "3" and ressort:
        return [u for u in cf.get("artikel", []) if f"/{ressort}/" in u][:limit]
    elif choice == "4":
        return cf.get("autor", [])[:limit]
    elif choice == "5":
        return cf.get("thema", [])[:limit]
    elif choice == "6":
        return cf.get("advertorial", [])[:limit]
    else:
        return structure["all_urls"][:limit]
```

---

## Phase 2: Disziplinen-Konfiguration

| Code | Disziplin | S | M | L |
|---|---|:---:|:---:|:---:|
| `robots` | robots.txt & KI-Bots | ✓ | ✓ | ✓ |
| `news_sitemap` | News-Sitemap (Struktur + Freshness) | ✓ | ✓ | ✓ |
| `discover` | Discover-Präsenz (Artikel-Count) | ✓ | ✓ | ✓ |
| `canonical` | Canonical & Republishing | ✓ | ✓ | ✓ |
| `schema` | Strukturierte Daten | — | ✓ | ✓ |
| `crawl_traps` | Crawl-Fallen & Budget | — | ✓ | ✓ |
| `thin_content` | Thin Content / Breaking-Stubs | — | ✓ | ✓ |
| `author_pages` | Autorseiten (E-E-A-T) | — | ✓ | ✓ |
| `advertorial` | Advertorials & Werbecluster | — | ✓ | ✓ |
| `topic_pages` | Themenseiten (Freshness + Duplikate) | — | — | ✓ |
| `channelizer` | Channelizer-Bereiche | — | — | ✓ |
| `site_structure` | URL-Cluster-Struktur | — | — | ✓ |
| `editorial_transparency` | Redaktionelle Transparenz | — | — | ✓ |
| `lighthouse` | Lighthouse (Chrome DevTools MCP) | — | — | ✓ |

```python
PRESETS = {
    "S": ["robots", "news_sitemap", "discover", "canonical"],
    "M": ["robots", "news_sitemap", "discover", "canonical",
          "schema", "crawl_traps", "thin_content", "author_pages", "advertorial"],
    "L": ["robots", "news_sitemap", "discover", "canonical",
          "schema", "crawl_traps", "thin_content", "author_pages", "advertorial",
          "topic_pages", "channelizer", "site_structure", "editorial_transparency", "lighthouse"],
}
```

---

## Phase 3: Crawler-Setup

### Screaming Frog MCP (v24+)

**Frage:** "Hast du Screaming Frog SEO Spider v24+ mit Lizenz?"

Wenn ja:

```
1. SF SEO Spider öffnen → Configuration → MCP Server → Start MCP Server (Port 11435)
2. In ~/.claude/settings.json:
   { "mcpServers": { "screaming-frog-mcp-server": { "url": "http://localhost:11435/mcp" } } }
3. Claude Code neu starten
4. Test: sf_list_crawls → Antwort = aktiv ✓
```

```python
# SF MCP Crawl-Workflow
sf_crawl(crawl_url=base_url, project_name=f"audit-{domain}")
while True:
    if sf_crawl_progress()["status"] == "complete":
        break
    time.sleep(10)
internal_all  = sf_generate_bulk_export(export="Internal:All")
canonical_exp = sf_generate_bulk_export(export="Canonicals:All")
schema_exp    = sf_generate_bulk_export(export="Structured Data:All")
```

### Chrome DevTools MCP (für Lighthouse, L-Preset)

**Frage:** "Ist Chrome DevTools MCP eingerichtet?" — nur für L-Preset relevant.

```
In ~/.claude/settings.json:
{ "mcpServers": { "chrome-devtools": { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-chrome-devtools"] } } }
```

Lighthouse via CDP:
```python
# Über Chrome DevTools MCP: Lighthouse-Report für eine URL abrufen
# cdp_navigate(url=target_url)
# cdp_run_lighthouse(url=target_url, categories=["performance","seo","accessibility"])
# → JSON-Report mit LCP, FID, CLS, SEO-Checks, Image-Audit
```

### Fallback-Crawler (kein SF)

```python
import requests, re, json
from bs4 import BeautifulSoup
from concurrent.futures import ThreadPoolExecutor

UA = "PublisherSEOAudit/1.0 (Mozilla Firefox)"

def crawl_url(url: str) -> dict:
    try:
        r = requests.get(url, timeout=12, headers={"User-Agent": UA}, allow_redirects=True)
        final_url = r.url
        status    = r.status_code
        if status != 200:
            return {"url": url, "final_url": final_url, "status": status, "error": f"HTTP {status}"}

        soup = BeautifulSoup(r.content, "lxml")

        canon_tag = soup.find("link", rel="canonical")
        canonical = canon_tag["href"].strip() if canon_tag else None

        robots_content = " ".join(
            t.get("content", "") for t in soup.find_all("meta", attrs={"name": re.compile(r"^robots$", re.I)})
        ).lower()

        title_tag = soup.find("title")
        title     = title_tag.get_text(strip=True) if title_tag else ""
        h1_tag    = soup.find("h1")
        h1        = h1_tag.get_text(strip=True) if h1_tag else ""

        og_img = ""
        tag = soup.find("meta", property="og:image")
        if tag:
            og_img = tag.get("content", "")

        # JSON-LD Schema
        schema_blocks, schema_items = [], []
        for script in soup.find_all("script", type="application/ld+json"):
            try:
                schema_blocks.append(json.loads(script.string or ""))
            except Exception:
                pass
        for block in schema_blocks:
            if isinstance(block, list):
                schema_items.extend(block)
            elif isinstance(block, dict):
                schema_items.append(block)

        schema_types = [i.get("@type", "") for i in schema_items if isinstance(i, dict)]

        date_published = author_name = author_url = ""
        same_as_links  = []
        for item in schema_items:
            if not isinstance(item, dict):
                continue
            if item.get("@type") in ("NewsArticle", "Article", "ReportageNewsArticle", "LiveBlogPosting"):
                date_published = date_published or item.get("datePublished", "")
                author = item.get("author", {})
                if isinstance(author, list):
                    author = author[0] if author else {}
                if isinstance(author, dict):
                    author_name = author_name or author.get("name", "")
                    author_url  = author_url  or author.get("url", "")
                elif isinstance(author, str):
                    author_name = author_name or author
            elif item.get("@type") in ("Person", "ProfilePage"):
                same_as_links = item.get("sameAs", [])
                if isinstance(same_as_links, str):
                    same_as_links = [same_as_links]

        if not date_published:
            tag = soup.find("meta", property="article:published_time")
            if tag:
                date_published = tag.get("content", "")

        if not author_url:
            a = soup.find("a", href=re.compile(r"/autor[en]?/|/author/", re.I))
            if a:
                author_url  = a.get("href", "")
                author_name = author_name or a.get_text(strip=True)

        # Werbecluster: Data-Attribute und Meta-Tags für Programmatic Advertising
        ad_cluster = _extract_ad_cluster(soup)

        # Advertorial-Signale
        advertorial_signals = _detect_advertorial(soup, url)

        main = (soup.find("main") or soup.find("article") or
                soup.find(class_=re.compile(r"article|story|content", re.I)))
        word_count = len(main.get_text(separator=" ", strip=True).split()) if main else 0

        redirects = [resp.url for resp in r.history]

        return {
            "url":                    url,
            "final_url":              final_url,
            "status":                 status,
            "canonical":              canonical,
            "is_canonical":           not canonical or canonical == url or canonical == final_url,
            "redirects":              redirects,
            "redirect_count":         len(redirects),
            "robots_meta":            robots_content,
            "noindex":                "noindex" in robots_content,
            "nofollow":               "nofollow" in robots_content,
            "max_image_preview_large": "max-image-preview:large" in robots_content,
            "title":                  title,
            "title_length":           len(title),
            "h1":                     h1,
            "og_image":               og_img,
            "schema_types":           schema_types,
            "schema_blocks":          schema_blocks,
            "date_published":         date_published,
            "author_name":            author_name,
            "author_url":             author_url,
            "same_as_links":          same_as_links,
            "word_count":             word_count,
            "ad_cluster":             ad_cluster,
            "advertorial_signals":    advertorial_signals,
            "error":                  None,
        }
    except Exception as e:
        return {"url": url, "status": None, "error": str(e)}


def _extract_ad_cluster(soup: BeautifulSoup) -> dict:
    """Extrahiert Werbecluster-Informationen für Programmatic Advertising."""
    clusters = {}

    # GAM/DFP: data-targeting, data-adunit, data-slot
    for tag in soup.find_all(attrs={"data-targeting": True}):
        try:
            clusters["gam_targeting"] = json.loads(tag["data-targeting"])
        except Exception:
            clusters["gam_targeting_raw"] = tag["data-targeting"]

    for tag in soup.find_all(attrs={"data-adunit": True}):
        clusters["adunit"] = tag["data-adunit"]

    for tag in soup.find_all(attrs={"data-ad-unit": True}):
        clusters["adunit"] = tag["data-ad-unit"]

    # Meta-Tags für Ad-Targeting
    for tag in soup.find_all("meta"):
        name = (tag.get("name") or tag.get("property") or "").lower()
        if any(k in name for k in ["adcat", "adkws", "adtargeting", "ivw", "comscore",
                                    "kategorie", "ressort", "channel", "sectionname"]):
            clusters[name] = tag.get("content", "")

    # JavaScript-Variablen für Ad-Targeting (googletag.cmd.push o.ä.)
    for script in soup.find_all("script"):
        text = script.string or ""
        # IVW-Kennzahlen
        ivw = re.search(r'iom\.c\(({[^}]+})', text)
        if ivw:
            clusters["ivw_raw"] = ivw.group(1)
        # GAM pageTargeting
        targeting = re.search(r'pageTargeting\s*[=:]\s*({[^}]+})', text)
        if targeting:
            try:
                clusters["page_targeting"] = json.loads(targeting.group(1))
            except Exception:
                clusters["page_targeting_raw"] = targeting.group(1)
        # Kategorie/Ressort-Variable
        cat = re.search(r'(?:category|ressort|section)\s*[=:]\s*["\']([^"\']+)["\']', text, re.I)
        if cat:
            clusters.setdefault("category", cat.group(1))

    return clusters


def _detect_advertorial(soup: BeautifulSoup, url: str) -> dict:
    """Erkennt Advertorial-Signale im HTML."""
    signals = {
        "url_based":     False,
        "css_class":     False,
        "meta_tag":      False,
        "schema_type":   False,
        "disclosure_text": False,
    }

    # URL-basiert
    if re.search(r"/advertorial/|/anzeige/|/sponsored/|/native-ad/|/partner-content/", url, re.I):
        signals["url_based"] = True

    # CSS-Klassen
    body = str(soup.body) if soup.body else ""
    if re.search(r'class=["\'][^"\']*(?:advertorial|sponsored|native-ad|partner-content|werbung)[^"\']*["\']',
                 body, re.I):
        signals["css_class"] = True

    # Meta-Tags
    for tag in soup.find_all("meta"):
        content = (tag.get("content") or tag.get("name") or "").lower()
        if any(k in content for k in ["advertorial", "sponsored", "paid", "native", "werbung", "anzeige"]):
            signals["meta_tag"] = True

    # Schema: AdvertiserContentArticle
    for script in soup.find_all("script", type="application/ld+json"):
        try:
            d = json.loads(script.string or "")
            types = [d.get("@type", "")] if isinstance(d, dict) else [i.get("@type","") for i in d if isinstance(i,dict)]
            if "AdvertiserContentArticle" in types:
                signals["schema_type"] = True
        except Exception:
            pass

    # Disclosure-Text im sichtbaren HTML
    disclosure_patterns = r"\banzeige\b|\bsponsored\b|\bwerbung\b|\bnative ad\b|\bpartner content\b|\badvertorial\b"
    headings = " ".join(t.get_text(strip=True) for t in soup.find_all(["h1","h2","h3","strong","span","div"])[:20])
    if re.search(disclosure_patterns, headings, re.I):
        signals["disclosure_text"] = True

    signals["is_advertorial"] = any(signals[k] for k in ["url_based", "css_class", "meta_tag", "schema_type"])
    return signals


def crawl_parallel(urls: list[str], max_workers: int = 8) -> list[dict]:
    results = []
    with ThreadPoolExecutor(max_workers=max_workers) as ex:
        for i, result in enumerate(ex.map(crawl_url, urls)):
            results.append(result)
            if (i + 1) % 50 == 0:
                print(f"  Gecrawlt: {i+1}/{len(urls)}")
    return results
```

---

## Phase 4: Audit-Disziplinen

### D1 — robots.txt & KI-Bots `[S]`

```python
def check_robots(base_url: str) -> dict:
    TIER1 = [  # alle 5 deutschen Großverlage blockieren diese
        "CCBot", "GPTBot", "Applebot-Extended", "Bytespider",
        "Meta-ExternalAgent", "meta-externalagent",
    ]
    TIER2 = [  # Mehrheit blockiert diese
        "ClaudeBot", "anthropic-ai", "Claude-Web", "Claude-User",
        "Amazonbot", "MistralAI-User", "PetalBot",
        "cohere-ai", "cohere-training-data-crawler",
        "AI2Bot", "Ai2Bot-Dolma", "DeepSeekBot", "DeepSeek",
        "Google-Extended",  # ⚠️ Gemini-Training ≠ Googlebot
        "FacebookBot", "Devin", "NovaAct", "Operator",
        "FriendlyCrawler", "img2dataset", "Timpibot", "TurnitinBot", "iaskspider",
    ]
    RETRIEVAL_BOTS = [
        "ChatGPT-User", "OAI-SearchBot",
        "PerplexityBot", "Perplexity-User",
        "Claude-SearchBot", "DuckAssistBot",
    ]

    try:
        robots_text = requests.get(
            f"{base_url}/robots.txt", timeout=8, headers={"User-Agent": UA}
        ).text
    except Exception as e:
        return {"discipline": "robots", "error": str(e)}

    rt_lower = robots_text.lower()

    tier1_blocked  = [b for b in TIER1 if b.lower() in rt_lower]
    tier1_missing  = [b for b in TIER1 if b.lower() not in rt_lower]
    tier2_blocked  = [b for b in TIER2 if b.lower() in rt_lower]
    tier2_missing  = [b for b in TIER2 if b.lower() not in rt_lower]
    retrieval_addressed = [b for b in RETRIEVAL_BOTS if b.lower() in rt_lower]

    has_legal_44b = "44b" in robots_text or "text and data mining" in rt_lower
    has_sitemap   = bool(re.search(r"(?i)sitemap:", robots_text))

    findings = []
    if tier1_missing:
        findings.append(f"⚠️ Tier-1-KI-Bots nicht blockiert: {', '.join(tier1_missing)}")
    if tier2_missing:
        preview = ', '.join(tier2_missing[:5]) + ('…' if len(tier2_missing) > 5 else '')
        findings.append(f"{len(tier2_missing)} Tier-2-KI-Bots nicht blockiert: {preview}")
    if not has_legal_44b:
        findings.append("§44b UrhG-Hinweis fehlt")
    if not has_sitemap:
        findings.append("Kein Sitemap:-Eintrag in robots.txt")
    if not retrieval_addressed:
        findings.append("Such-Bots (ChatGPT-User, PerplexityBot) nicht behandelt — Entscheidung offen")

    score = len(tier1_blocked)*3 + len(tier2_blocked) + (2 if has_legal_44b else 0) + (1 if has_sitemap else 0)
    max_s = len(TIER1)*3 + len(TIER2) + 3

    return {
        "discipline":           "robots",
        "robots_text":          robots_text,
        "tier1_blocked":        tier1_blocked,
        "tier1_missing":        tier1_missing,
        "tier2_blocked":        tier2_blocked,
        "tier2_missing":        tier2_missing,
        "retrieval_addressed":  retrieval_addressed,
        "has_legal_44b":        has_legal_44b,
        "has_sitemap":          has_sitemap,
        "findings":             findings,
        "score_pct":            round(score / max_s * 100),
    }
```

### D2 — News-Sitemap `[S]`

```python
def check_news_sitemap(sitemaps: dict) -> dict:
    from datetime import datetime, timezone

    if not sitemaps["news"]:
        return {
            "discipline": "news_sitemap",
            "error":      "Keine News-Sitemap gefunden",
            "findings":   ["Keine News-Sitemap in robots.txt oder /sitemap-news.xml"],
            "score_pct":  0,
        }

    all_entries, findings = [], []

    for sm in sitemaps["news"]:
        soup = sm.get("soup")
        if not soup:
            continue
        for entry in soup.find_all("url"):
            loc      = entry.find("loc")
            news_tag = entry.find("news:news")
            if not loc or not news_tag:
                continue
            pub_date_tag = news_tag.find("news:publication_date")
            title_tag    = news_tag.find("news:title")
            name_tag     = news_tag.find("news:name")
            dp_str = pub_date_tag.text.strip() if pub_date_tag else ""
            try:
                dt    = datetime.fromisoformat(dp_str.replace("Z", "+00:00"))
                age_h = (datetime.now(timezone.utc) - dt).total_seconds() / 3600
            except Exception:
                dt = None; age_h = None
            all_entries.append({
                "url":      loc.text.strip(),
                "title":    title_tag.text.strip() if title_tag else "",
                "pub_name": name_tag.text.strip() if name_tag else "",
                "pub_date": dp_str,
                "age_hours": round(age_h, 1) if age_h else None,
            })

    stale    = [e for e in all_entries if e["age_hours"] and e["age_hours"] > 48]
    no_date  = [e for e in all_entries if not e["pub_date"]]
    no_title = [e for e in all_entries if not e["title"]]
    pub_names = Counter(e["pub_name"] for e in all_entries if e["pub_name"])

    if stale:
        findings.append(f"{len(stale)} Einträge älter als 48h — aus News-Sitemap entfernen")
    if len(all_entries) > 1000:
        findings.append(f"News-Sitemap hat {len(all_entries)} Einträge — Google-Limit 1.000 überschritten")
    if no_date:
        findings.append(f"{len(no_date)} Einträge ohne news:publication_date")
    if no_title:
        findings.append(f"{len(no_title)} Einträge ohne news:title")
    if len(pub_names) > 1:
        findings.append(f"Inkonsistenter Publikationsname: {dict(pub_names)}")
    if not all_entries:
        findings.append("News-Sitemap ist leer")

    return {
        "discipline": "news_sitemap",
        "total":      len(all_entries),
        "stale":      stale[:10],
        "no_date":    no_date[:10],
        "pub_names":  dict(pub_names),
        "findings":   findings,
        "preview":    all_entries[:10],
        "score_pct":  max(0, 100 - len(stale)*4 - len(no_date)*8 - (30 if not all_entries else 0)),
    }
```

### D3 — Discover-Präsenz (Artikel-Count) `[S]`

Der Audit zählt: von den gestrigen Artikeln (aus News-Sitemap) — wie viele haben alle Discover-Voraussetzungen erfüllt? Diese Zahl ist der Trend-Indikator: Sinkt sie von Audit zu Audit, gibt es Handlungsbedarf.

```python
def check_discover(crawl_results: list[dict], sitemaps: dict) -> dict:
    from datetime import datetime, timezone, timedelta

    yesterday = (datetime.now(timezone.utc) - timedelta(days=1)).date()

    # Artikel vom gestrigen Tag aus News-Sitemap
    news_yesterday = []
    for sm in sitemaps["news"]:
        soup = sm.get("soup")
        if not soup:
            continue
        for entry in soup.find_all("url"):
            loc      = entry.find("loc")
            news_tag = entry.find("news:news")
            if not loc or not news_tag:
                continue
            dp_tag = news_tag.find("news:publication_date")
            if not dp_tag:
                continue
            try:
                dt = datetime.fromisoformat(dp_tag.text.strip().replace("Z","+00:00"))
                if dt.date() == yesterday:
                    news_yesterday.append(loc.text.strip())
            except Exception:
                pass

    # Gecrawlte Daten auf gestrige Artikel einschränken
    yesterday_set = set(news_yesterday)
    yesterday_crawled = [
        r for r in crawl_results
        if r.get("url") in yesterday_set and r.get("status") == 200
    ]

    # Fallback: wenn gestrige Artikel nicht im Scope-Crawl waren, gezielt nachholen
    if not yesterday_crawled and news_yesterday:
        print(f"  Discover: {len(news_yesterday)} gestrige Artikel nicht im Scope-Crawl — Stichproben-Crawl (max 30)...")
        extra = crawl_parallel(news_yesterday[:30], max_workers=5)
        yesterday_crawled = [r for r in extra if r.get("status") == 200]

    # Discover-Voraussetzungen prüfen
    discover_ready = [
        r for r in yesterday_crawled
        if r.get("max_image_preview_large") and r.get("og_image") and r.get("date_published")
    ]

    missing_mip  = [r for r in yesterday_crawled if not r.get("max_image_preview_large")]
    missing_img  = [r for r in yesterday_crawled if not r.get("og_image")]
    short_title  = [r for r in yesterday_crawled if r.get("title_length", 0) < 30]
    long_title   = [r for r in yesterday_crawled if r.get("title_length", 0) > 70]

    n = len(yesterday_crawled)

    findings = []
    if n == 0 and not news_yesterday:
        findings.append("Keine gestrigen Artikel in News-Sitemap gefunden — Sitemap prüfen")
    elif n == 0:
        findings.append(f"Gestrige Artikel in Sitemap ({len(news_yesterday)}), aber Crawl fehlgeschlagen — Netzwerk prüfen")
    else:
        pct = round(len(discover_ready)/n*100)
        findings.append(f"{len(discover_ready)}/{n} gestrige Artikel ({pct}%) Discover-ready")
        if missing_mip:
            findings.append(f"⚠️ max-image-preview:large fehlt auf {len(missing_mip)}/{n} — kritisch")
        if missing_img:
            findings.append(f"og:image fehlt auf {len(missing_img)}/{n}")
        if short_title:
            findings.append(f"{len(short_title)} Headlines <30 Zeichen")
        if long_title:
            findings.append(f"{len(long_title)} Headlines >70 Zeichen")

    return {
        "discipline":         "discover",
        "news_yesterday":     len(news_yesterday),
        "crawled_yesterday":  n,
        "discover_ready":     len(discover_ready),
        "discover_ready_pct": round(len(discover_ready)/max(n,1)*100),
        "missing_mip":        len(missing_mip),
        "missing_img":        len(missing_img),
        "findings":           findings,
        "detail":             [r for r in yesterday_crawled if r not in discover_ready][:20],
        "score_pct":          round(len(discover_ready)/max(n,1)*100),
    }
```

### D4 — Canonical & Republishing `[S]`

```python
def check_canonical(crawl_results: list[dict]) -> dict:
    arts = [r for r in crawl_results if r.get("status") == 200]

    non_canonical   = [a for a in arts if not a.get("is_canonical") and a.get("canonical")]
    no_canonical    = [a for a in arts if not a.get("canonical")]
    redirect_chains = [a for a in arts if a.get("redirect_count", 0) > 1]
    noindex_scope   = [a for a in arts if a.get("noindex")]

    republishing_candidates = []
    for a in arts:
        if a.get("canonical") and a.get("canonical") != a["url"]:
            orig_path  = urlparse(a["url"]).path
            canon_path = urlparse(a["canonical"]).path
            if orig_path != canon_path:
                republishing_candidates.append({"url": a["url"], "canonical": a["canonical"]})

    cross_domain = [
        rc for rc in republishing_candidates
        if urlparse(rc["url"]).netloc != urlparse(rc["canonical"]).netloc
    ]

    findings = []
    if non_canonical:
        findings.append(f"{len(non_canonical)} URLs mit Self-Canonical-Abweichung")
    if no_canonical:
        findings.append(f"{len(no_canonical)} URLs ohne Canonical-Tag")
    if redirect_chains:
        findings.append(f"{len(redirect_chains)} Redirect-Chains >1 Hop")
    if noindex_scope:
        findings.append(f"{len(noindex_scope)} noindex-URLs im Scope")
    if cross_domain:
        findings.append(f"{len(cross_domain)} Cross-Domain-Canonicals (Domain-übergreifendes Republishing)")

    return {
        "discipline":              "canonical",
        "non_canonical":           non_canonical[:15],
        "no_canonical":            no_canonical[:10],
        "redirect_chains":         redirect_chains[:10],
        "noindex_scope":           noindex_scope[:10],
        "republishing_candidates": republishing_candidates[:15],
        "cross_domain":            cross_domain[:10],
        "findings":                findings,
        "score_pct":               max(0, 100 - len(non_canonical)*3 - len(no_canonical)*2 - len(redirect_chains)*2),
    }
```

### D5 — Strukturierte Daten `[M]`

```python
def check_schema(crawl_results: list[dict]) -> dict:
    REQUIRED = ["headline", "datePublished", "author", "image", "publisher"]

    arts = [r for r in crawl_results if r.get("status") == 200]
    n    = len(arts)
    if not n:
        return {"discipline": "schema", "error": "Keine gecrawlten Seiten", "findings": []}

    no_schema, no_news_article, live_blogs = [], [], []
    findings, detail = [], []

    for a in arts:
        types  = a.get("schema_types", [])
        blocks = a.get("schema_blocks", [])

        items = []
        for b in blocks:
            if isinstance(b, list):
                items.extend(b)
            elif isinstance(b, dict):
                items.append(b)

        if not types:
            no_schema.append(a["url"])
            continue

        article = next((i for i in items if isinstance(i, dict) and
                        i.get("@type") in ("NewsArticle","Article","ReportageNewsArticle")), None)

        if not article:
            no_news_article.append(a["url"])
            continue

        issues         = []
        missing_fields = [f for f in REQUIRED if not article.get(f)]
        if missing_fields:
            issues.append(f"Pflichtfelder fehlen: {', '.join(missing_fields)}")

        author = article.get("author")
        if isinstance(author, str):
            issues.append("author als String — kein Person-Objekt")
        elif isinstance(author, dict) and not author.get("url"):
            issues.append("author.url fehlt")
        elif isinstance(author, list):
            if any(isinstance(au, dict) and not au.get("url") for au in author):
                issues.append("Mindestens ein author ohne url")

        # Bildgröße aus Schema (kein Bild-Download nötig)
        image    = article.get("image", [])
        img_list = image if isinstance(image, list) else [image]
        has_large = any(
            isinstance(i, dict) and int(i.get("width", 0)) >= 1200
            for i in img_list if isinstance(i, dict)
        )
        if img_list and not has_large:
            issues.append("Kein Bild ≥1200px im Schema (width-Feld)")

        if article.get("isAccessibleForFree") is False and not article.get("hasPart"):
            issues.append("isAccessibleForFree=false ohne hasPart (Paywall-Markup unvollständig)")

        if "LiveBlogPosting" in types:
            live_blogs.append(a["url"])

        if issues:
            detail.append({"url": a["url"], "title": a.get("title",""), "issues": issues})

    if no_schema:
        findings.append(f"⚠️ {len(no_schema)}/{n} Seiten ohne JSON-LD Schema")
    if no_news_article:
        findings.append(f"{len(no_news_article)}/{n} Seiten ohne NewsArticle-Schema")
    if detail:
        findings.append(f"{len(detail)}/{n} NewsArticle-Schemas mit Fehlern")
    if live_blogs:
        findings.append(f"{len(live_blogs)} LiveBlogPosting gefunden (Live-Button möglich)")

    return {
        "discipline":      "schema",
        "total":           n,
        "no_schema":       no_schema[:15],
        "no_news_article": no_news_article[:15],
        "live_blogs":      live_blogs[:5],
        "detail":          detail[:30],
        "findings":        findings,
        "score_pct":       max(0, 100 - int(len(no_schema)/n*50) - int(len(detail)/n*30)),
    }
```

### D6 — Crawl-Fallen & Budget `[M]`

```python
def check_crawl_traps(sitemaps: dict, base_url: str, structure: dict) -> dict:
    all_urls = structure["all_urls"]
    robots   = sitemaps.get("robots_text", "").lower()

    date_arch  = [u for u in all_urls if re.search(r"/\d{4}/\d{2}(/\d{2})?/?$", urlparse(u).path)]
    param_urls = [u for u in all_urls if "?" in u and
                  any(p in u.lower() for p in ["sort=","filter=","page=","seite=","p=","offset="])]
    tag_pages  = structure["classified_full"].get("thema", [])
    paginated  = [u for u in all_urls if re.search(r"/seite/\d+|/page/\d+|\?p=\d+", u, re.I)]

    search_blocked            = "disallow: /suche" in robots or "disallow: /search" in robots
    archive_blocked           = bool(re.search(r"disallow:\s*/\d{4}", robots))
    infinite_scroll_blocked   = bool(re.search(r"disallow:.*(/load-more|/infinite|/ajax)", robots))

    findings = []
    if date_arch and not archive_blocked:
        findings.append(f"{len(date_arch)} Kalender-Archiv-URLs in Sitemap, nicht in robots.txt blockiert")
    elif date_arch:
        findings.append(f"{len(date_arch)} Kalender-Archiv-URLs in Sitemap — robots.txt blockiert, aber noch in Sitemap")
    if param_urls:
        findings.append(f"{len(param_urls)} URL-Parameter-Varianten in Sitemap")
    if len(tag_pages) > 500:
        findings.append(f"{len(tag_pages)} Tag-/Themenseiten — viele davon wahrscheinlich dünn")
    if not search_blocked:
        findings.append("Suchseiten nicht per robots.txt blockiert")
    if paginated:
        findings.append(f"{len(paginated)} paginierte URLs in Sitemaps")
    if not infinite_scroll_blocked:
        findings.append("Infinite-Scroll-Endpunkte nicht explizit in robots.txt blockiert")

    return {
        "discipline":       "crawl_traps",
        "date_arch_count":  len(date_arch),
        "date_archives":    date_arch[:5],
        "param_urls_count": len(param_urls),
        "tag_pages_count":  len(tag_pages),
        "paginated_count":  len(paginated),
        "search_blocked":   search_blocked,
        "archive_blocked":  archive_blocked,
        "findings":         findings,
        "score_pct":        max(0, 100 - len(findings) * 14),
    }
```

### D7 — Thin Content & Breaking-News-Stubs `[M]`

```python
def check_thin_content(crawl_results: list[dict]) -> dict:
    from datetime import datetime, timezone

    arts = [r for r in crawl_results if r.get("status") == 200]
    now  = datetime.now(timezone.utc)
    n    = len(arts)

    stubs, no_date = [], []

    for a in arts:
        if not a.get("date_published"):
            no_date.append(a["url"])

        wc = a.get("word_count", 0)
        if wc < 150:
            age_h = None
            dp = a.get("date_published", "")
            if dp:
                try:
                    age_h = (now - datetime.fromisoformat(dp.replace("Z","+00:00"))).total_seconds()/3600
                except Exception:
                    pass
            stubs.append({
                "url":      a["url"],
                "title":    a.get("title",""),
                "words":    wc,
                "age_hours": round(age_h, 1) if age_h else None,
                "old_stub": bool(age_h and age_h > 48),
            })

    old_stubs = [s for s in stubs if s["old_stub"]]

    findings = []
    if old_stubs:
        findings.append(f"⚠️ {len(old_stubs)} veraltete Stubs (>48h, <150 Wörter)")
    if len(stubs) > n * 0.3:
        findings.append(f"{len(stubs)}/{n} Artikel unter 150 Wörtern ({round(len(stubs)/n*100)}%)")
    if no_date:
        findings.append(f"{len(no_date)} Artikel ohne erkennbares Publikationsdatum")

    return {
        "discipline": "thin_content",
        "total":      n,
        "stubs":      stubs[:20],
        "old_stubs":  old_stubs,
        "no_date":    no_date[:15],
        "findings":   findings,
        "score_pct":  max(0, 100 - len(old_stubs)*3 - int(len(no_date)/max(n,1)*30)),
    }
```

### D8 — Autorseiten (E-E-A-T) `[M]`

```python
def check_author_pages(crawl_results: list[dict], structure: dict) -> dict:
    author_urls = list({
        r["author_url"] for r in crawl_results
        if r.get("author_url") and re.search(r"/autor[en]?/|/author/", r.get("author_url",""), re.I)
    })
    if not author_urls:
        author_urls = structure["classified_full"].get("autor", [])

    articles_without_author = [
        r["url"] for r in crawl_results
        if r.get("status") == 200 and not r.get("author_name") and not r.get("author_url")
    ]

    findings, page_issues = [], []

    if not author_urls:
        findings.append("⚠️ Keine Autorseiten gefunden")
        return {
            "discipline":              "author_pages",
            "author_pages_found":      0,
            "articles_without_author": articles_without_author[:15],
            "findings":                findings,
            "score_pct":               0,
        }

    sample  = author_urls[:30]
    crawled = crawl_parallel(sample, max_workers=5)

    for page in crawled:
        if page.get("status") != 200:
            page_issues.append({"url": page["url"], "issues": [f"HTTP {page.get('status','?')}"]})
            continue
        issues = []
        types  = page.get("schema_types", [])
        if "Person" not in types and "ProfilePage" not in types:
            issues.append("Person/ProfilePage-Schema fehlt")
        if not page.get("h1"):
            issues.append("Kein H1-Tag")
        if page.get("word_count", 0) < 50:
            issues.append(f"Sehr dünn ({page.get('word_count',0)} Wörter)")
        if page.get("noindex"):
            issues.append("⚠️ noindex — Autorseite nicht indexiert!")
        if not page.get("same_as_links"):
            issues.append("sameAs-Links fehlen (LinkedIn, Twitter etc.)")
        if issues:
            page_issues.append({"url": page["url"], "issues": issues})

    noindex_author = [p for p in page_issues if any("noindex" in i for i in p["issues"])]

    if articles_without_author:
        findings.append(f"{len(articles_without_author)} Artikel ohne Autorenangabe")
    if page_issues:
        findings.append(f"{len(page_issues)}/{len(sample)} Autorseiten mit E-E-A-T-Lücken")
    if noindex_author:
        findings.append(f"⚠️ {len(noindex_author)} Autorseiten sind noindex")

    return {
        "discipline":               "author_pages",
        "author_pages_found":       len(author_urls),
        "sampled":                  len(sample),
        "page_issues":              page_issues,
        "articles_without_author":  articles_without_author[:15],
        "findings":                 findings,
        "score_pct":                max(0, 100 - int(len(page_issues)/max(len(sample),1)*50) - (20 if articles_without_author else 0)),
    }
```

### D9 — Advertorials & Werbecluster `[M]`

```python
def check_advertorial(crawl_results: list[dict], structure: dict, sitemaps: dict) -> dict:
    """
    Prüft Advertorials auf korrektes SEO-Handling und extrahiert
    Werbecluster-Daten für Programmatic-Advertising-Transparenz.
    """
    all_results = [r for r in crawl_results if r.get("status") == 200]
    n = len(all_results)

    # Advertorials identifizieren
    identified_advertorials = [
        r for r in all_results
        if r.get("advertorial_signals", {}).get("is_advertorial")
    ]

    # URL-basiert erkannte Advertorials aus Sitemap-Analyse
    sitemap_advertorials = structure["classified_full"].get("advertorial", [])

    all_advertorial_urls = set(
        [r["url"] for r in identified_advertorials] + sitemap_advertorials
    )

    # News-Sitemap-URLs sammeln für Abgleich
    news_sitemap_urls = set()
    for sm in sitemaps.get("news", []):
        soup = sm.get("soup")
        if soup:
            news_sitemap_urls.update(loc.text.strip() for loc in soup.find_all("loc"))

    # Probleme bei Advertorials
    advertorial_issues = []
    for r in identified_advertorials:
        issues = []
        signals = r.get("advertorial_signals", {})

        # Advertorial sollte noindex sein oder canonical auf Original zeigen
        if not r.get("noindex") and r.get("is_canonical"):
            issues.append("Advertorial ist indexiert und hat Self-Canonical — noindex prüfen")

        # Schema-Typ korrekt? Advertorials sollten nicht NewsArticle verwenden
        if "NewsArticle" in r.get("schema_types", []):
            issues.append("⚠️ Advertorial verwendet NewsArticle-Schema — sollte AdvertiserContentArticle sein")

        # Disclosure vorhanden?
        if not signals.get("disclosure_text") and not signals.get("css_class"):
            issues.append("Keine sichtbare Kennzeichnung als Werbung erkannt")

        # In News-Sitemap enthalten? (wäre kritischer Fehler)
        if r["url"] in news_sitemap_urls:
            issues.append("⚠️ Advertorial in News-Sitemap — Google News darf das nicht indexieren")

        if issues:
            advertorial_issues.append({"url": r["url"], "issues": issues})

    # Sitemap-Advertorials in News-Sitemap
    sitemap_adverts_in_news = [u for u in sitemap_advertorials if u in news_sitemap_urls]

    # Werbecluster-Übersicht
    cluster_data = {}
    for r in all_results:
        ac = r.get("ad_cluster", {})
        if not ac:
            continue
        cat = ac.get("category") or ac.get("adunit") or ""
        if cat:
            cluster_data[cat] = cluster_data.get(cat, 0) + 1

    # Werbecluster-Konsistenz: Artikel ohne Cluster-Zuweisung
    no_cluster = [
        r["url"] for r in all_results
        if not r.get("ad_cluster") and r["url"] not in all_advertorial_urls
    ]

    findings = []
    if sitemap_advertorials:
        findings.append(f"{len(sitemap_advertorials)} Advertorial-URLs in Sitemap erkannt (URL-basiert)")
    if sitemap_adverts_in_news:
        findings.append(f"⚠️ {len(sitemap_adverts_in_news)} Advertorials in News-Sitemap — sofort entfernen")
    if advertorial_issues:
        findings.append(f"⚠️ {len(advertorial_issues)} Advertorials mit SEO-Handling-Problemen")
    if cluster_data:
        top_clusters = sorted(cluster_data.items(), key=lambda x: -x[1])[:10]
        findings.append(f"Werbecluster erkannt: {', '.join(f'{k} ({v})' for k,v in top_clusters[:5])}")
    if no_cluster and len(no_cluster) > n * 0.5:
        findings.append(f"{len(no_cluster)}/{n} Artikel ohne erkennbare Werbecluster-Zuweisung im HTML")

    return {
        "discipline":             "advertorial",
        "identified_count":       len(all_advertorial_urls),
        "sitemap_count":          len(sitemap_advertorials),
        "advertorial_issues":     advertorial_issues[:15],
        "cluster_data":           dict(sorted(cluster_data.items(), key=lambda x: -x[1])[:20]),
        "no_cluster_count":       len(no_cluster),
        "findings":               findings,
        "score_pct":              max(0, 100 - len(advertorial_issues)*10),
    }
```

### D10 — Themenseiten (Freshness + Duplikate) `[L]`

```python
def check_topic_pages(structure: dict, sitemaps: dict) -> dict:
    from datetime import datetime, timezone
    from difflib import SequenceMatcher

    topic_urls = structure["classified_full"].get("thema", [])
    if not topic_urls:
        return {"discipline": "topic_pages", "error": "Keine Themenseiten gefunden", "findings": []}

    # Artikel-URLs mit Datum aus allen Sitemaps sammeln
    article_dates: dict[str, str] = {}
    for sm in sitemaps["news"] + sitemaps["other"]:
        soup = sm.get("soup")
        if not soup:
            continue
        for entry in soup.find_all("url"):
            loc = entry.find("loc")
            if not loc:
                continue
            url = loc.text.strip()
            news_tag = entry.find("news:news")
            if news_tag:
                dp = news_tag.find("news:publication_date")
                if dp:
                    article_dates[url] = dp.text.strip()
            else:
                lm = entry.find("lastmod")
                if lm:
                    article_dates[url] = lm.text.strip()

    now = datetime.now(timezone.utc)
    STALE_DAYS = 30

    topic_freshness = []
    for topic_url in topic_urls:
        topic_slug = urlparse(topic_url).path.strip("/").split("/")[-1].lower()
        if not topic_slug or len(topic_slug) < 4:
            continue

        # Top-3-Artikel zu diesem Thema: neueste Artikel mit Slug im URL
        matching = sorted(
            [(u, d) for u, d in article_dates.items() if topic_slug in u.lower()],
            key=lambda x: x[1],
            reverse=True,
        )[:3]

        last_date_str = matching[0][1] if matching else ""
        age_days      = None
        if last_date_str:
            try:
                dt       = datetime.fromisoformat(last_date_str.replace("Z", "+00:00"))
                age_days = (now - dt).days
            except Exception:
                pass

        topic_freshness.append({
            "url":          topic_url,
            "slug":         topic_slug,
            "top3_articles": [{"url": u, "date": d} for u, d in matching],
            "last_date":    last_date_str,
            "age_days":     age_days,
            "stale":        bool(age_days is not None and age_days > STALE_DAYS),
            "no_articles":  len(matching) == 0,
        })

    stale_topics      = [t for t in topic_freshness if t["stale"]]
    no_article_topics = [t for t in topic_freshness if t["no_articles"]]

    # Duplikat-Erkennung über Slug-Ähnlichkeit
    slugs       = [t["slug"] for t in topic_freshness if t.get("slug") and len(t["slug"]) >= 4]
    slug_to_url = {t["slug"]: t["url"] for t in topic_freshness}
    duplicates, checked = [], set()

    for i, s1 in enumerate(slugs):
        for s2 in slugs[i+1:]:
            pair = tuple(sorted([s1, s2]))
            if pair in checked:
                continue
            checked.add(pair)
            sim = SequenceMatcher(None, s1, s2).ratio()
            if sim >= 0.80 and s1 != s2:
                duplicates.append({
                    "slug1": s1, "url1": slug_to_url.get(s1,""),
                    "slug2": s2, "url2": slug_to_url.get(s2,""),
                    "similarity": round(sim, 2),
                })

    findings = []
    if stale_topics:
        findings.append(f"⚠️ {len(stale_topics)} Themenseiten ohne frischen Content >{STALE_DAYS} Tage — deindexieren oder konsolidieren")
    if no_article_topics:
        findings.append(f"{len(no_article_topics)} Themenseiten ohne zuordenbare Artikel in Sitemap")
    if duplicates:
        findings.append(f"{len(duplicates)} mögliche Duplikat-Paare (Slug-Ähnlichkeit ≥80%)")

    return {
        "discipline":        "topic_pages",
        "total":             len(topic_urls),
        "stale_topics":      stale_topics[:15],
        "no_article_topics": no_article_topics[:10],
        "duplicates":        duplicates[:20],
        "freshness_sample":  topic_freshness[:20],
        "findings":          findings,
        "score_pct":         max(0, 100 - int(len(stale_topics)/max(len(topic_urls),1)*60) - (20 if duplicates else 0)),
    }
```

### D11 — Channelizer-Bereiche `[L]`

```python
def check_channelizer(structure: dict, base_url: str) -> dict:
    base_domain = urlparse(base_url).netloc.replace("www.", "")
    domains     = structure.get("domains", {})
    all_urls    = structure.get("all_urls", [])

    suspicious_subdomains = {
        netloc: count
        for netloc, count in domains.items()
        if netloc != base_domain
        and netloc.endswith("." + base_domain)
        and count > 5
    }

    CHANNELIZER_PATTERNS = [
        r"/deals/", r"/shopping/", r"/gutscheine?/", r"/produkte?/",
        r"/preisvergleich/", r"/anzeige/", r"/sponsored/", r"/partner/",
        r"/immobilien?/", r"/jobs?/", r"/finanzen?/",
    ]
    channelizer_paths = defaultdict(int)
    for url in all_urls:
        path = urlparse(url).path.lower()
        for pat in CHANNELIZER_PATTERNS:
            if re.search(pat, path):
                seg = re.search(pat, path).group(0).strip("/")
                channelizer_paths[seg] += 1

    findings = []
    if suspicious_subdomains:
        for sd, count in suspicious_subdomains.items():
            findings.append(f"Sub-Domain {sd} ({count} URLs) — Channelizer oder eigenständige Redaktion?")
    for seg, count in sorted(channelizer_paths.items(), key=lambda x: -x[1])[:5]:
        findings.append(f"/{seg}/ ({count} URLs) — Channelizer-Bereich erkannt")

    return {
        "discipline":           "channelizer",
        "suspicious_subdomains": suspicious_subdomains,
        "channelizer_paths":    dict(channelizer_paths),
        "findings":             findings,
        "score_pct":            None,
    }
```

### D12 — URL-Cluster-Struktur `[L]`

```python
def check_site_structure(crawl_results: list[dict], structure: dict) -> dict:
    all_urls = [r["url"] for r in crawl_results if r.get("status") == 200]

    clusters = defaultdict(list)
    for url in all_urls:
        seg = urlparse(url).path.strip("/").split("/")[0] or "root"
        clusters[seg].append(url)

    cluster_summary = sorted(
        [{"cluster": k, "count": len(v), "sample": v[:2]} for k, v in clusters.items()],
        key=lambda x: -x["count"]
    )
    tiny_clusters = [c for c in cluster_summary if c["count"] < 5 and c["cluster"] != "root"]

    depth_counter = Counter(len(urlparse(u).path.strip("/").split("/")) for u in all_urls)
    deep_urls     = [u for u in all_urls if len(urlparse(u).path.strip("/").split("/")) > 5]

    findings = []
    if len(clusters) > 30:
        findings.append(f"{len(clusters)} Pfad-Cluster — stark fragmentierte Silo-Struktur")
    if tiny_clusters:
        findings.append(f"{len(tiny_clusters)} Cluster mit <5 URLs — isolierte Bereiche")
    if deep_urls:
        findings.append(f"{len(deep_urls)} URLs mit >5 Pfad-Ebenen")

    return {
        "discipline":      "site_structure",
        "cluster_summary": cluster_summary[:20],
        "total_clusters":  len(clusters),
        "tiny_clusters":   tiny_clusters,
        "depth_dist":      dict(depth_counter),
        "deep_urls":       deep_urls[:10],
        "findings":        findings,
        "score_pct":       max(0, 100 - len(tiny_clusters)*3 - (20 if len(clusters)>30 else 0)),
    }
```

### D13 — Redaktionelle Transparenz `[L]`

```python
def check_editorial_transparency(base_url: str) -> dict:
    """
    Prüft automatisch erkennbare redaktionelle Transparenz-Signale:
    Impressum, Datenschutz, Newsroom-/Publisher-Prinzipien-Seite.
    """
    TRANSPARENCY_PATHS = [
        # Impressum / Legal
        ("/impressum", "impressum"),
        ("/impressum/", "impressum"),
        # Datenschutz
        ("/datenschutz", "datenschutz"),
        ("/datenschutz/", "datenschutz"),
        ("/datenschutzerklaerung", "datenschutz"),
        # Newsroom / Publisher-Prinzipien / "So arbeiten wir"
        ("/newsroom", "newsroom"),
        ("/so-arbeiten-wir", "publisher_prinzipien"),
        ("/wie-wir-arbeiten", "publisher_prinzipien"),
        ("/publisher-prinzipien", "publisher_prinzipien"),
        ("/redaktionsstatut", "publisher_prinzipien"),
        ("/leitlinien", "publisher_prinzipien"),
        ("/prinzipien", "publisher_prinzipien"),
        ("/ueber-uns/so-arbeiten-wir", "publisher_prinzipien"),
        ("/ueber-uns/redaktion", "publisher_prinzipien"),
        # Kontakt
        ("/kontakt", "kontakt"),
        # Korrekturen
        ("/korrekturen", "korrekturen"),
        ("/fehlermeldung", "korrekturen"),
    ]

    found, missing = {}, []

    for path, category in TRANSPARENCY_PATHS:
        url = base_url.rstrip("/") + path
        try:
            r = requests.head(url, timeout=5, headers={"User-Agent": UA}, allow_redirects=True)
            if r.status_code == 200:
                found[category] = found.get(category) or url
        except Exception:
            pass

    required = {"impressum", "datenschutz", "publisher_prinzipien"}
    missing  = [k for k in required if k not in found]

    # Schema: NewsMediaOrganization auf der Startseite?
    try:
        r = requests.get(base_url, timeout=8, headers={"User-Agent": UA})
        soup = BeautifulSoup(r.content, "lxml")
        schema_types = []
        for script in soup.find_all("script", type="application/ld+json"):
            try:
                d = json.loads(script.string or "")
                items = d if isinstance(d, list) else [d]
                schema_types.extend(i.get("@type","") for i in items if isinstance(i, dict))
            except Exception:
                pass
        has_news_org = "NewsMediaOrganization" in schema_types
    except Exception:
        has_news_org = False

    findings = []
    if "impressum" in missing:
        findings.append("⚠️ Impressum nicht unter Standard-URL gefunden")
    if "publisher_prinzipien" in missing:
        findings.append("Publisher-Prinzipien / 'So arbeiten wir'-Seite nicht gefunden")
    if not has_news_org:
        findings.append("NewsMediaOrganization-Schema auf Startseite fehlt")

    return {
        "discipline":     "editorial_transparency",
        "found":          found,
        "missing":        missing,
        "has_news_org":   has_news_org,
        "findings":       findings,
        "score_pct":      max(0, 100 - len(missing)*25 - (15 if not has_news_org else 0)),
    }
```

### D14 — Lighthouse (Chrome DevTools MCP) `[L]`

```python
def check_lighthouse(scope_urls: list[str], max_pages: int = 5) -> dict:
    """
    Führt Lighthouse-Checks über Chrome DevTools MCP aus.
    Erfordert: Chrome DevTools MCP eingerichtet (siehe Phase 3).

    Wichtig: CWV sind für Publisher kein primärer Rankingfaktor.
    Lighthouse wird genutzt für: Bildoptimierung, SEO-Checks, Accessibility.
    Maximal {max_pages} Seiten (Stichprobe).
    """
    # Prüfen ob Chrome DevTools MCP verfügbar ist — Verbindungstest
    mcp_available = False
    try:
        # cdp_navigate(url="about:blank")  # wirft Exception wenn MCP nicht verbunden
        # mcp_available = True
        pass  # MCP-Tool-Namen vom jeweiligen Setup abhängig — hier manuell aktivieren
    except Exception:
        pass

    if not mcp_available:
        return {
            "discipline": "lighthouse",
            "sampled":    0,
            "findings":   [
                "Chrome DevTools MCP nicht verbunden — Lighthouse-Check übersprungen. "
                "MCP einrichten (siehe Phase 3) und Audit wiederholen."
            ],
            "score_pct":  None,
        }

    sample   = scope_urls[:max_pages]
    results  = []

    for url in sample:
        try:
            # Chrome DevTools MCP: Lighthouse-Report abrufen
            # cdp_navigate(url=url)
            # report = cdp_run_lighthouse(
            #     url=url,
            #     categories=["seo", "performance"],
            #     settings={"formFactor": "mobile", "throttlingMethod": "simulate"}
            # )
            report = {}  # Platzhalter: durch tatsächlichen MCP-Aufruf ersetzen

            seo_score  = report.get("categories", {}).get("seo",         {}).get("score", 0) * 100
            perf_score = report.get("categories", {}).get("performance", {}).get("score", 0) * 100

            audits = report.get("audits", {})
            img_issues = [
                k for k, v in audits.items()
                if "image" in k and v.get("score", 1) < 0.9
            ]
            seo_issues = [
                {"id": k, "title": v.get("title","")}
                for k, v in audits.items()
                if v.get("score", 1) < 0.9 and v.get("weight", 0) > 0
                and any(cat in k for cat in ["meta","canonical","robots","hreflang","structured"])
            ]

            results.append({
                "url":        url,
                "seo_score":  round(seo_score),
                "perf_score": round(perf_score),
                "img_issues": img_issues,
                "seo_issues": seo_issues,
            })
        except Exception as e:
            results.append({"url": url, "error": str(e)})

    avg_seo  = round(sum(r.get("seo_score", 0)  for r in results) / max(len(results), 1))
    avg_perf = round(sum(r.get("perf_score", 0) for r in results) / max(len(results), 1))

    findings = []
    img_problems = [r for r in results if r.get("img_issues")]
    if img_problems:
        findings.append(f"Bildoptimierungs-Issues auf {len(img_problems)}/{len(results)} Seiten")
    if avg_seo < 80:
        findings.append(f"Durchschn. Lighthouse SEO-Score: {avg_seo}/100")

    return {
        "discipline": "lighthouse",
        "sampled":    len(results),
        "avg_seo":    avg_seo,
        "avg_perf":   avg_perf,
        "results":    results,
        "findings":   findings,
        "score_pct":  avg_seo if results else None,
    }
```

---

## Phase 5: HTML-Dashboard erzeugen

```python
def generate_dashboard(results: dict, meta: dict, crawl_data: list[dict]) -> str:
    import os
    from datetime import date

    domain  = meta.get("domain", "unknown")
    scope   = meta.get("scope_label", "")
    today   = date.today().isoformat()
    out     = os.path.expanduser(f"~/Downloads/publisher-seo-audit-{domain}-{today}.html")

    scores = [v.get("score_pct") for v in results.values()
              if isinstance(v, dict) and v.get("score_pct") is not None]
    overall = round(sum(scores) / len(scores)) if scores else 0
    color   = "#22c55e" if overall >= 70 else "#f59e0b" if overall >= 40 else "#ef4444"

    DISC_LABELS = {
        "robots":                 "robots.txt & KI-Bots",
        "news_sitemap":           "News-Sitemap",
        "discover":               "Discover-Präsenz",
        "canonical":              "Canonical & Republishing",
        "schema":                 "Strukturierte Daten",
        "crawl_traps":            "Crawl-Fallen",
        "thin_content":           "Thin Content",
        "author_pages":           "Autorseiten (E-E-A-T)",
        "advertorial":            "Advertorials & Werbecluster",
        "topic_pages":            "Themenseiten",
        "channelizer":            "Channelizer",
        "site_structure":         "URL-Cluster-Struktur",
        "editorial_transparency": "Redaktionelle Transparenz",
        "lighthouse":             "Lighthouse",
    }

    tiles_html = ""
    for code, label in DISC_LABELS.items():
        if code not in results:
            continue
        r  = results[code]
        sc = r.get("score_pct")
        if r.get("error"):
            tiles_html += f'<div class="tile tile-err"><div class="tile-label">{label}</div><div class="tile-score">—</div></div>'
        elif sc is None:
            tiles_html += f'<div class="tile tile-na"><div class="tile-label">{label}</div><div class="tile-score">?</div></div>'
        else:
            cls = "tile-ok" if sc >= 70 else "tile-warn" if sc >= 40 else "tile-crit"
            tiles_html += f'<div class="tile {cls}"><div class="tile-label">{label}</div><div class="tile-score">{sc}%</div></div>'

    all_findings_html = ""
    for code, r in results.items():
        if not isinstance(r, dict):
            continue
        label = DISC_LABELS.get(code, code)
        for f in r.get("findings", []):
            all_findings_html += f'<tr><td class="td-disc">{label}</td><td>{f}</td></tr>'

    crawl_json   = json.dumps(crawl_data,  ensure_ascii=False, default=str)
    results_json = json.dumps(results,     ensure_ascii=False, default=str)

    html = f"""<!DOCTYPE html>
<html lang="de">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>Publisher SEO Audit — {domain} — {today}</title>
<style>
  :root {{
    --bg:#F3F2EF;--surface:#fff;--border:#DDD9D2;--text:#1B1A17;--text2:#6A6660;
  }}
  @media(prefers-color-scheme:dark){{:root:not([data-theme="light"]){{
    --bg:#131210;--surface:#1D1C1A;--border:#2C2A27;--text:#E6E3DC;--text2:#A09C95;
  }}}}
  [data-theme="dark"]{{--bg:#131210;--surface:#1D1C1A;--border:#2C2A27;--text:#E6E3DC;--text2:#A09C95;}}
  *{{box-sizing:border-box;margin:0;padding:0}}
  body{{font-family:system-ui,sans-serif;background:var(--bg);color:var(--text)}}
  .hdr{{background:var(--surface);border-bottom:1px solid var(--border);padding:22px 32px 16px}}
  .hdr-domain{{font-size:19px;font-weight:700;margin-bottom:3px}}
  .hdr-meta{{font-size:12.5px;color:var(--text2)}}
  .overall{{display:inline-block;margin-left:12px;font-size:21px;font-weight:700;color:{color}}}
  .main{{max-width:1100px;margin:0 auto;padding:22px 32px}}
  h2{{font-size:12.5px;font-weight:700;letter-spacing:.07em;text-transform:uppercase;
      color:var(--text2);margin:26px 0 10px}}
  .tiles{{display:grid;grid-template-columns:repeat(auto-fill,minmax(150px,1fr));gap:8px;margin-bottom:26px}}
  .tile{{background:var(--surface);border:1px solid var(--border);border-radius:6px;
         padding:11px;border-left:4px solid var(--border)}}
  .tile-ok{{border-left-color:#16A34A}}.tile-warn{{border-left-color:#D97706}}
  .tile-crit{{border-left-color:#C42B19}}.tile-err,.tile-na{{border-left-color:#aaa}}
  .tile-label{{font-size:10.5px;color:var(--text2);margin-bottom:4px;line-height:1.3}}
  .tile-score{{font-size:19px;font-weight:700}}
  .tile-ok .tile-score{{color:#16A34A}}.tile-warn .tile-score{{color:#D97706}}
  .tile-crit .tile-score{{color:#C42B19}}
  table{{width:100%;border-collapse:collapse;font-size:12.5px}}
  th{{text-align:left;padding:6px 10px;border-bottom:2px solid var(--border);
      font-size:10px;letter-spacing:.09em;text-transform:uppercase;color:var(--text2)}}
  td{{padding:6px 10px;border-bottom:1px solid var(--border);vertical-align:top}}
  .td-disc{{color:var(--text2);font-size:10.5px;white-space:nowrap;width:150px}}
  tr:hover td{{background:var(--bg)}}
  .tab-bar{{display:flex;gap:4px;border-bottom:2px solid var(--border);margin-bottom:16px}}
  .tab{{padding:7px 13px;font-size:12px;font-weight:500;cursor:pointer;
        border-bottom:2px solid transparent;margin-bottom:-2px;color:var(--text2)}}
  .tab.active{{color:var(--text);border-bottom-color:var(--text)}}
  .tab-content{{display:none}}.tab-content.active{{display:block}}
  pre{{font-size:10.5px;background:var(--bg);padding:11px;border-radius:5px;
       overflow-x:auto;white-space:pre-wrap;max-height:360px;overflow-y:auto}}
  input[type=search]{{width:100%;padding:6px 10px;border:1px solid var(--border);
                      border-radius:5px;background:var(--surface);color:var(--text);
                      font-size:12px;margin-bottom:9px}}
</style>
</head>
<body>
<div class="hdr">
  <div class="hdr-domain">{domain}<span class="overall">{overall}%</span></div>
  <div class="hdr-meta">Publisher SEO Audit · {today} · Scope: {scope} · {len(crawl_data)} URLs gecrawlt</div>
</div>
<div class="main">
  <div class="tab-bar">
    <div class="tab active" onclick="switchTab('overview',this)">Übersicht</div>
    <div class="tab" onclick="switchTab('findings',this)">Alle Findings</div>
    <div class="tab" onclick="switchTab('robots_tab',this)">robots.txt</div>
    <div class="tab" onclick="switchTab('werbecluster',this)">Werbecluster</div>
    <div class="tab" onclick="switchTab('crawl_tab',this)">Crawl-Daten</div>
  </div>
  <div id="overview" class="tab-content active">
    <h2>Score nach Disziplin</h2>
    <div class="tiles">{tiles_html}</div>
  </div>
  <div id="findings" class="tab-content">
    <h2>Alle Findings</h2>
    <table><thead><tr><th>Disziplin</th><th>Finding</th></tr></thead>
    <tbody>{all_findings_html}</tbody></table>
  </div>
  <div id="robots_tab" class="tab-content">
    <h2>robots.txt</h2><pre id="robots-pre"></pre>
  </div>
  <div id="werbecluster" class="tab-content">
    <h2>Werbecluster-Übersicht</h2>
    <table id="cluster-table"><thead><tr><th>Cluster/Kategorie</th><th>Anzahl URLs</th></tr></thead>
    <tbody id="cluster-tbody"></tbody></table>
  </div>
  <div id="crawl_tab" class="tab-content">
    <h2>Gecrawlte URLs</h2>
    <input type="search" placeholder="URL oder Titel suchen…" oninput="filterCrawl(this.value)">
    <table><thead><tr>
      <th>URL</th><th>St.</th><th>Titel</th><th>Schema</th>
      <th>MIP</th><th>Datum</th><th>Wörter</th><th>noindex</th><th>Advertorial</th>
    </tr></thead><tbody id="crawl-tbody"></tbody></table>
  </div>
</div>
<script id="crawl-data" type="application/json">{crawl_json}</script>
<script id="audit-results" type="application/json">{results_json}</script>
<script>
const CRAWL   = JSON.parse(document.getElementById('crawl-data').textContent);
const RESULTS = JSON.parse(document.getElementById('audit-results').textContent);

function switchTab(id, el) {{
  document.querySelectorAll('.tab-content').forEach(e => e.classList.remove('active'));
  document.querySelectorAll('.tab').forEach(e => e.classList.remove('active'));
  document.getElementById(id).classList.add('active');
  el.classList.add('active');
  if (id === 'robots_tab') renderRobots();
  if (id === 'werbecluster') renderCluster();
  if (id === 'crawl_tab' && !document.getElementById('crawl-tbody').children.length) renderCrawl(CRAWL);
}}

function renderRobots() {{
  const r = RESULTS.robots;
  document.getElementById('robots-pre').textContent = r ? (r.robots_text||'(nicht verfügbar)') : '(nicht geprüft)';
}}

function renderCluster() {{
  const adv = RESULTS.advertorial;
  if (!adv) return;
  const data = adv.cluster_data || {{}};
  document.getElementById('cluster-tbody').innerHTML =
    Object.entries(data).sort((a,b)=>b[1]-a[1]).map(([k,v]) =>
      `<tr><td>${{k}}</td><td>${{v}}</td></tr>`).join('');
}}

function renderCrawl(data) {{
  document.getElementById('crawl-tbody').innerHTML = data.slice(0,500).map(r => {{
    const adv = r.advertorial_signals || {{}};
    return `<tr>
      <td style="max-width:240px;overflow:hidden;text-overflow:ellipsis;white-space:nowrap">
        <a href="${{r.url}}" target="_blank" style="color:inherit;font-size:10.5px">${{r.url}}</a></td>
      <td style="color:${{r.status===200?'inherit':'#C42B19'}}">${{r.status||'err'}}</td>
      <td style="max-width:190px;overflow:hidden;text-overflow:ellipsis;white-space:nowrap;font-size:10.5px">${{r.title||''}}</td>
      <td style="font-size:10px">${{(r.schema_types||[]).join(', ')}}</td>
      <td style="text-align:center">${{r.max_image_preview_large?'✓':'✗'}}</td>
      <td style="font-size:10px">${{r.date_published||'—'}}</td>
      <td style="text-align:right">${{r.word_count||0}}</td>
      <td style="text-align:center;color:${{r.noindex?'#C42B19':'inherit'}}">${{r.noindex?'✗':''}}</td>
      <td style="text-align:center;color:${{adv.is_advertorial?'#D97706':'inherit'}}">${{adv.is_advertorial?'Adv':''}}</td>
    </tr>`;
  }}).join('');
}}

function filterCrawl(q) {{
  document.querySelectorAll('#crawl-tbody tr').forEach(row => {{
    row.style.display = row.textContent.toLowerCase().includes(q.toLowerCase()) ? '' : 'none';
  }});
}}
</script>
</body>
</html>"""

    with open(out, "w", encoding="utf-8") as f:
        f.write(html)
    return out
```

---

## Phase 6: Einstiegspunkt

```python
def run_audit(base_url: str):
    from urllib.parse import urlparse
    domain = urlparse(base_url).netloc.replace("www.", "")
    print(f"\n{'='*60}\n  Publisher SEO Audit — {domain}\n{'='*60}\n")

    # Phase 0 — immer zuerst
    sitemaps  = discover_sitemaps(base_url)
    structure = analyze_url_structure(sitemaps)
    # → Struktur ausgeben (Phase 0.3), User-Feedback einholen

    # Phase 1 — scope_choice vom User
    # scope_urls = build_scope_urls(scope_choice, structure)

    # Phase 2 — preset vom User
    # disciplines = PRESETS.get(preset, PRESETS["M"])

    # Phase 3 — crawl_data
    # use_sf = ...  # SF MCP verfügbar?
    # crawl_data = crawl_parallel(scope_urls, max_workers=8)

    results = {}
    # if "robots"                 in disciplines: results["robots"]                 = check_robots(base_url)
    # if "news_sitemap"           in disciplines: results["news_sitemap"]           = check_news_sitemap(sitemaps)
    # if "discover"               in disciplines: results["discover"]               = check_discover(crawl_data, sitemaps)
    # if "canonical"              in disciplines: results["canonical"]              = check_canonical(crawl_data)
    # if "schema"                 in disciplines: results["schema"]                 = check_schema(crawl_data)
    # if "crawl_traps"            in disciplines: results["crawl_traps"]            = check_crawl_traps(sitemaps, base_url, structure)
    # if "thin_content"           in disciplines: results["thin_content"]           = check_thin_content(crawl_data)
    # if "author_pages"           in disciplines: results["author_pages"]           = check_author_pages(crawl_data, structure)
    # if "advertorial"            in disciplines: results["advertorial"]            = check_advertorial(crawl_data, structure, sitemaps)
    # if "topic_pages"            in disciplines: results["topic_pages"]            = check_topic_pages(structure, sitemaps)
    # if "channelizer"            in disciplines: results["channelizer"]            = check_channelizer(structure, base_url)
    # if "site_structure"         in disciplines: results["site_structure"]         = check_site_structure(crawl_data, structure)
    # if "editorial_transparency" in disciplines: results["editorial_transparency"] = check_editorial_transparency(base_url)
    # if "lighthouse"             in disciplines: results["lighthouse"]             = check_lighthouse(scope_urls)

    out = generate_dashboard(results, {"domain": domain, "scope_label": "custom"}, crawl_data)
    print(f"\n✓ Dashboard: {out}")
    return out
```

---

## Abhängigkeiten

| Paket | Zweck |
|---|---|
| `requests` | HTTP |
| `beautifulsoup4` + `lxml` | HTML + XML Parsing |
| `difflib` | Slug-Ähnlichkeit (stdlib) |
| `concurrent.futures` | Parallel-Crawl (stdlib) |

**MCP-Server (optional):**
- Screaming Frog SEO Spider v24+, Lizenz erforderlich (£199/yr)
- Chrome DevTools MCP: `npx @modelcontextprotocol/server-chrome-devtools` (für Lighthouse-Checks im L-Preset)

---

## Händisch prüfen (nach dem automatischen Audit)

Diese Punkte erfordern redaktionelles Urteilsvermögen oder sind nicht automatisch bewertbar:

1. **Hero-Image-Qualität**: og:image wirklich ≥1200px und 16:9? (Schema liefert nur width-Feld falls vorhanden; Lighthouse prüft Bildoptimierung, aber nicht redaktionelle Qualität)
2. **Autorenbiografie-Qualität**: Expertise, aktueller Lebenslauf, Foto, korrekte sameAs-Links?
3. **Themenseiten-Einleiter**: Redaktioneller Intro-Text vorhanden, aktuell, dem Thema angemessen?
4. **Zusammenführung ähnlicher Themenseiten**: Slug-Ähnlichkeit ist ein Signal, ob Inhalte inhaltlich zusammenführbar sind, erfordert Sichtprüfung
5. **Advertorial-Kennzeichnung vollständig**: Automatisch erkannte Advertorials — ist die Kennzeichnung für User ausreichend klar und rechtskonform?
6. **Werbecluster-Zuweisung korrekt**: Entspricht das erkannte Ad-Cluster dem redaktionellen Kontext? Fehlzuweisungen in DFP/GAM nur manuell korrigierbar
7. **Channelizer-Qualität**: Entsprechen Inhalte in Channelizer-Bereichen den Qualitätsstandards? Crawl-/Index-Behandlung dokumentiert?
8. **Republishing-Mehrwert**: Haben republizierte Artikel (Cross-Domain-Canonicals) erkennbaren Mehrwert?
9. **Orphan Pages vollständig**: Cluster-Analyse zeigt isolierte Bereiche — vollständige Orphan-Analyse über SF MCP `sf_generate_bulk_export("Internal:All")` oder manuelle Crawl-Graph-Analyse
10. **Breaking-News-Stub-Entscheidung**: Veraltete Stubs markiert — noindex oder ausbauen ist eine redaktionelle Einzelfallentscheidung
11. **Such-Bots-Strategie**: Ob ChatGPT-User, PerplexityBot etc. erlaubt oder blockiert sein sollen — strategische Verlagsentscheidung (Traffic vs. Urheberrecht)
12. **Interne Verlinkung**: Qualität und Relevanz interner Links (welche Artikel sollten miteinander verlinkt sein) — nur über separaten Internal-Linking-Audit oder manuell
