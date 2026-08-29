---
name: discover-report-analyzer
description: Analysiert einen Google Search Console Discover Performance Export (XLSX). Prüft auf das 1000-Zeilen-Limit, berechnet Indexierungsrate via Sitemap, analysiert Ressort-Performance (3 Stufen), Headline-Muster (Fragen, Entity-Prefix, Länge, Zahlen, Sentiment), CTR-Ausreißer, Long-tail-Performer und erzeugt ein HTML-Report mit Actionable Insights. Use when asked to analyse a GSC Discover export, check Discover performance, or run a Discover audit.
tools: Read, Edit, Write, Bash
---

# Discover Report Analyzer

Strukturierte Analyse eines GSC Discover Performance Exports. Der Report beginnt immer mit **Actionable Insights** — konkreten Handlungsempfehlungen basierend auf den tatsächlichen Zahlen.

---

## Inputformat

### Was der User hochladen muss

Der User lädt **eine XLSX-Datei** hoch — den GSC Discover Performance Export. Diese Datei enthält automatisch alle nötigen Sheets.

So exportieren aus GSC:
1. [search.google.com/search-console](https://search.google.com/search-console) öffnen
2. Links im Menü → **Suchergebnisse** → oben den Tab **Discover** wählen
3. Zeitraum wählen: **Letzte 3 Monate** (für Tagesrate) oder **Letzte 16 Monate** (für Wochenrate)
4. Seitenfilter: für vollständige Site **kein Filter** setzen — wenn der Report 1000 Zeilen hat, einen Ressort-Filter hinzufügen (z.B. `/sport/`, `/leben-wissen/`)
5. Oben rechts: **Exportieren** → **Excel (.xlsx)**

Die Datei enthält diese Sheets:
| Sheet | Inhalt |
|---|---|
| `Diagramm` | Tägliche Daten: Datum, Klicks, Impressionen, CTR |
| `Seiten` | URL-Daten: Seite, Klicks, Impressionen, CTR |
| `Länder` | Länderdaten |
| `Darstellung in Discover` | Discover-Erscheinungstypen (News Showcase etc.) |
| `Filter` | Report-Metadaten: Zeitraum, angewendete Filter |

---

## Workflow

### Phase 0: Datei einlesen und parsen

```python
import openpyxl
import pandas as pd
from urllib.parse import urlparse
from collections import defaultdict, Counter
from datetime import datetime, timedelta
import re, json, os

def parse_gsc_discover_export(filepath: str) -> dict:
    """Liest alle relevanten Sheets aus dem GSC Discover XLSX."""
    wb = openpyxl.load_workbook(filepath, read_only=True, data_only=True)
    
    def sheet_to_df(sheetname):
        if sheetname not in wb.sheetnames:
            return None
        ws = wb[sheetname]
        rows = list(ws.iter_rows(values_only=True))
        if len(rows) < 2:
            return None
        headers = [str(h) if h else f"col_{i}" for i, h in enumerate(rows[0])]
        return pd.DataFrame(rows[1:], columns=headers)
    
    # Seiten-Sheet: Spalte 0 heißt "Die häufigsten Seiten"
    df_pages = sheet_to_df("Seiten")
    if df_pages is not None:
        df_pages.columns = ["url", "clicks", "impressions", "ctr"]
        df_pages = df_pages.dropna(subset=["url"])
        df_pages["clicks"] = pd.to_numeric(df_pages["clicks"], errors="coerce").fillna(0).astype(int)
        df_pages["impressions"] = pd.to_numeric(df_pages["impressions"], errors="coerce").fillna(0).astype(int)
        df_pages["ctr"] = pd.to_numeric(df_pages["ctr"], errors="coerce").fillna(0)
    
    # Diagramm-Sheet: Datum, Klicks, Impressionen, CTR
    df_dates = sheet_to_df("Diagramm")
    if df_dates is not None:
        df_dates.columns = ["date", "clicks", "impressions", "ctr"]
        df_dates["date"] = pd.to_datetime(df_dates["date"], errors="coerce")
        df_dates = df_dates.dropna(subset=["date"])
        df_dates["clicks"] = pd.to_numeric(df_dates["clicks"], errors="coerce").fillna(0).astype(int)
        df_dates["impressions"] = pd.to_numeric(df_dates["impressions"], errors="coerce").fillna(0).astype(int)
    
    # Länder-Sheet
    df_countries = sheet_to_df("Länder")
    if df_countries is not None:
        df_countries.columns = ["country", "clicks", "impressions", "ctr"]
    
    # Darstellung in Discover
    df_appearance = sheet_to_df("Darstellung in Discover")
    
    # Filter-Sheet → Metadaten
    meta = {}
    if "Filter" in wb.sheetnames:
        ws = wb["Filter"]
        for row in ws.iter_rows(values_only=True):
            if row[0] and row[1]:
                meta[str(row[0]).strip()] = str(row[1]).strip()
    
    return {
        "pages": df_pages,
        "dates": df_dates,
        "countries": df_countries,
        "appearance": df_appearance,
        "meta": meta,
        "filepath": filepath,
    }
```

### Phase 1: 1000-Zeilen-Guard

```python
def check_row_limit(data: dict) -> dict:
    """Erkennt truncated Export und gibt konkrete Filter-Empfehlung."""
    df = data["pages"]
    if df is None:
        return {"truncated": False, "row_count": 0, "warning": "Seiten-Sheet nicht gefunden"}
    
    row_count = len(df)
    truncated = row_count >= 1000
    
    if not truncated:
        return {"truncated": False, "row_count": row_count}
    
    # Aktuellen Filter aus Meta ermitteln
    page_filter = data["meta"].get("Seite", "")
    
    # Top-Ressorts aus den vorhandenen URLs berechnen
    sections = []
    for url in df["url"].head(200):
        try:
            path = urlparse(url).path.strip("/")
            seg = path.split("/")[0] if path else ""
            if seg:
                sections.append(seg)
        except Exception:
            pass
    
    top_sections = Counter(sections).most_common(8)
    
    # Filter-Empfehlungen
    suggestions = []
    for sec, count in top_sections:
        pct = round(count / 200 * 100)
        suggestions.append({
            "section": f"/{sec}/",
            "estimated_share": pct,
            "gsc_filter": f"+{sec}",
        })
    
    return {
        "truncated": True,
        "row_count": row_count,
        "current_filter": page_filter,
        "suggestions": suggestions,
        "warning": (
            f"⚠️ Der Export hat {row_count} Zeilen — das ist das GSC-Limit. "
            f"Die Daten sind unvollständig und nicht repräsentativ. "
            f"Analysiere stattdessen einzelne Bereiche der Site und filtere in GSC auf einen dieser Pfade:"
        ),
    }
```

### Phase 2: Report-Metadaten und Zeitraum

```python
def detect_report_scope(data: dict) -> dict:
    """Ermittelt Zeitraum, Domain und ob daily/weekly Analyse sinnvoll ist."""
    meta = data["meta"]
    df_dates = data["dates"]
    df_pages = data["pages"]
    
    # Zeitraum aus Filter-Sheet
    date_range_raw = meta.get("Datum", "")
    
    # Zeitraum aus Diagramm-Daten berechnen
    if df_dates is not None and len(df_dates) > 0:
        min_date = df_dates["date"].min()
        max_date = df_dates["date"].max()
        days = (max_date - min_date).days
    else:
        days = 0
        min_date = max_date = None
    
    # Domain aus URLs extrahieren
    domain = ""
    if df_pages is not None and len(df_pages) > 0:
        try:
            domain = urlparse(df_pages["url"].iloc[0]).netloc.replace("www.", "")
        except Exception:
            pass
    
    # Aggregations-Granularität: <120 Tage → täglich, sonst wöchentlich
    granularity = "daily" if days <= 120 else "weekly"
    
    return {
        "domain": domain,
        "date_range_raw": date_range_raw,
        "min_date": min_date,
        "max_date": max_date,
        "days": days,
        "granularity": granularity,  # "daily" oder "weekly"
        "page_filter": meta.get("Seite", ""),
    }
```

### Phase 3: Sitemap laden für Indexierungsrate

```python
import requests
from bs4 import BeautifulSoup

UA = "DiscoverAnalyzer/1.0"

def fetch_articles_from_sitemaps(domain: str, scope: dict) -> dict:
    """
    Lädt die relevanten Sitemaps für den Report-Zeitraum und gibt Artikel
    mit Publikationsdatum zurück.
    
    Strategie:
    - täglich (≤120 Tage): News-Sitemap + monatliche Sitemaps
    - wöchentlich (>120 Tage): Monatliche Sitemaps der letzten 12–16 Monate
    """
    base_url = f"https://www.{domain}" if not domain.startswith("http") else domain
    
    # robots.txt → Sitemaps
    try:
        robots = requests.get(f"{base_url}/robots.txt", timeout=8, headers={"User-Agent": UA}).text
        sitemap_urls = re.findall(r"(?i)Sitemap:\s*(\S+)", robots)
    except Exception:
        sitemap_urls = [f"{base_url}/sitemap.xml", f"{base_url}/sitemap-index.xml"]
    
    articles = {}  # url → {"pub_date": datetime, "title": str}
    
    def parse_sitemap(url):
        try:
            r = requests.get(url, timeout=12, headers={"User-Agent": UA})
            soup = BeautifulSoup(r.content, "xml")
            
            # Sitemap-Index → rekursiv laden
            if soup.find("sitemapindex"):
                for loc in soup.find_all("loc"):
                    child_url = loc.text.strip()
                    # Nur relevante Sitemaps laden (news oder monatliche Artikel-Sitemaps)
                    if _is_relevant_sitemap(child_url, scope):
                        parse_sitemap(child_url)
                return
            
            # URL-Set
            for entry in soup.find_all("url"):
                loc = entry.find("loc")
                if not loc:
                    continue
                art_url = loc.text.strip()
                
                # Datum aus news:publication_date oder lastmod
                pub_date = None
                news_tag = entry.find("news:publication_date")
                if news_tag:
                    try:
                        pub_date = datetime.fromisoformat(news_tag.text.strip().replace("Z", "+00:00"))
                    except Exception:
                        pass
                if not pub_date:
                    lastmod = entry.find("lastmod")
                    if lastmod:
                        try:
                            pub_date = datetime.fromisoformat(lastmod.text.strip().replace("Z", "+00:00"))
                        except Exception:
                            pass
                
                title_tag = entry.find("news:title")
                title = title_tag.text.strip() if title_tag else ""
                
                if pub_date:
                    articles[art_url] = {"pub_date": pub_date.date(), "title": title}
        except Exception as e:
            print(f"  ! Sitemap {url}: {e}")
    
    def _is_relevant_sitemap(url, scope):
        url_lower = url.lower()
        # News-Sitemaps immer
        if "news" in url_lower:
            return True
        # Monatliche Artikel-Sitemaps im relevanten Zeitraum
        if scope.get("min_date"):
            min_dt = scope["min_date"]
            max_dt = scope["max_date"] or datetime.now()
            # Prüfe ob Jahr/Monat im Pfad im Zeitraum liegt
            year_month = re.search(r"(\d{4})[_\-/](\d{2})", url)
            if year_month:
                y, m = int(year_month.group(1)), int(year_month.group(2))
                sm_date = datetime(y, m, 1)
                if min_dt.replace(tzinfo=None) - timedelta(days=31) <= sm_date <= (max_dt if hasattr(max_dt, 'year') else datetime.now()):
                    return True
        # Generische Artikel-Sitemaps (ohne Datum im Namen) bei kurzen Zeiträumen
        if scope.get("days", 0) <= 120:
            return True
        return False
    
    for sm_url in sitemap_urls:
        parse_sitemap(sm_url)
    
    # Filtern auf Report-Zeitraum
    if scope.get("min_date") and scope.get("max_date"):
        min_d = scope["min_date"].date() if hasattr(scope["min_date"], "date") else scope["min_date"]
        max_d = scope["max_date"].date() if hasattr(scope["max_date"], "date") else scope["max_date"]
        # Etwas Puffer: 14 Tage vor min_date, da Sitemap-Fetch oft verzögert
        articles = {
            url: v for url, v in articles.items()
            if min_d - timedelta(days=14) <= v["pub_date"] <= max_d
        }
    
    return articles
```

### Phase 4: Indexierungsrate berechnen

```python
def calculate_indexation_rate(data: dict, scope: dict, sitemap_articles: dict) -> dict:
    """
    Für jeden Zeitraum (Tag/Woche):
    - Anzahl publizierter Artikel (aus Sitemap)
    - Anzahl davon mit Discover-Impressionen (aus Seiten-Sheet)
    - Rate = discovered / published
    
    Gibt auch eine "Verzögerungs-Analyse" aus: wann tauchen Artikel in Discover auf
    (sofern Datumsdaten genug Granularität haben).
    """
    df_pages = data["pages"]
    if df_pages is None or not sitemap_articles:
        return {"error": "Keine Daten für Indexierungsraten-Analyse"}
    
    granularity = scope["granularity"]
    discover_urls = set(df_pages["url"].tolist())
    
    # Sitemap-Artikel nach Zeiteinheit gruppieren
    if granularity == "daily":
        groups = defaultdict(list)
        for url, info in sitemap_articles.items():
            groups[str(info["pub_date"])].append(url)
    else:
        # Wöchentlich: ISO-Wochennummer
        groups = defaultdict(list)
        for url, info in sitemap_articles.items():
            d = info["pub_date"]
            week_key = f"{d.isocalendar()[0]}-W{d.isocalendar()[1]:02d}"
            groups[week_key].append(url)
    
    periods = []
    for period_key in sorted(groups.keys()):
        published_urls = groups[period_key]
        discovered = [u for u in published_urls if u in discover_urls]
        rate = len(discovered) / len(published_urls) if published_urls else 0
        periods.append({
            "period": period_key,
            "published": len(published_urls),
            "discovered": len(discovered),
            "rate": round(rate * 100, 1),
        })
    
    # Durchschnittliche Rate (über Perioden mit mind. 5 Artikeln)
    valid = [p for p in periods if p["published"] >= 5]
    avg_rate = round(sum(p["rate"] for p in valid) / len(valid), 1) if valid else 0
    
    return {
        "granularity": granularity,
        "periods": periods,
        "avg_rate": avg_rate,
        "total_sitemap_articles": len(sitemap_articles),
        "total_discovered": len([u for u in sitemap_articles if u in discover_urls]),
    }
```

### Phase 5: Ressort-Performance-Analyse

```python
def analyze_sections(data: dict, scope: dict) -> dict:
    """
    Gruppiert URLs nach Ressort (Pfad-Segment 1), berechnet Metriken
    und klassifiziert in 3 Stufen: 🔴 schwach / 🟡 mittel / 🟢 stark.
    """
    df = data["pages"]
    if df is None:
        return {"error": "Keine Seiten-Daten"}
    
    page_filter = scope.get("page_filter", "")
    
    sections = defaultdict(lambda: {"urls": [], "clicks": 0, "impressions": 0})
    
    for _, row in df.iterrows():
        url = row["url"]
        try:
            path = urlparse(url).path.strip("/")
            seg = path.split("/")[0] if path else "/"
            if not seg:
                seg = "/"
        except Exception:
            seg = "unknown"
        sections[seg]["urls"].append(url)
        sections[seg]["clicks"] += int(row["clicks"] or 0)
        sections[seg]["impressions"] += int(row["impressions"] or 0)
    
    results = []
    for seg, vals in sections.items():
        n = len(vals["urls"])
        clicks = vals["clicks"]
        impressions = vals["impressions"]
        avg_clicks = round(clicks / n, 1) if n else 0
        avg_imp = round(impressions / n, 1) if n else 0
        avg_ctr = round(clicks / impressions * 100, 2) if impressions else 0
        results.append({
            "section": f"/{seg}/",
            "article_count": n,
            "total_clicks": clicks,
            "total_impressions": impressions,
            "avg_clicks_per_article": avg_clicks,
            "avg_impressions_per_article": avg_imp,
            "avg_ctr": avg_ctr,
        })
    
    results.sort(key=lambda x: -x["total_impressions"])
    
    # Baseline: gewichteter Durchschnitt avg_clicks_per_article
    total_urls = sum(r["article_count"] for r in results)
    total_clicks = sum(r["total_clicks"] for r in results)
    baseline_clicks = round(total_clicks / total_urls, 1) if total_urls else 0
    
    # 3-Stufen-Klassifikation basierend auf avg_clicks_per_article vs. Baseline
    for r in results:
        ratio = r["avg_clicks_per_article"] / baseline_clicks if baseline_clicks else 0
        if ratio >= 1.5:
            r["tier"] = "strong"       # 🟢 ≥1.5× Baseline
            r["tier_label"] = "🟢 Stark"
        elif ratio >= 0.6:
            r["tier"] = "medium"       # 🟡 0.6–1.5×
            r["tier_label"] = "🟡 Mittel"
        else:
            r["tier"] = "weak"         # 🔴 <0.6×
            r["tier_label"] = "🔴 Schwach"
        r["baseline_ratio"] = round(ratio, 2)
    
    # Aktionsfähige Insights vorbereiten
    strong = [r for r in results if r["tier"] == "strong" and r["article_count"] >= 5]
    weak   = [r for r in results if r["tier"] == "weak"   and r["article_count"] >= 5]
    # Ressorts die WENIG publizieren aber jeder Artikel performt super
    hidden_gems = [r for r in results if r["tier"] == "strong" and r["article_count"] < 10]
    
    return {
        "sections": results,
        "baseline_clicks_per_article": baseline_clicks,
        "strong_sections": strong[:5],
        "weak_sections": weak[:5],
        "hidden_gems": hidden_gems[:3],
        "total_urls": total_urls,
    }
```

### Phase 6: Headline-Analyse (og:title scrapen)

```python
from concurrent.futures import ThreadPoolExecutor

def scrape_og_titles(df_pages: pd.DataFrame, max_urls: int = 150) -> dict:
    """
    Scrapt og:title aus den Top-URLs (nach Impressionen sortiert).
    Begrenze auf 150 URLs, rate-limitted mit ThreadPoolExecutor.
    """
    top_urls = df_pages.nlargest(max_urls, "impressions")["url"].tolist()
    
    def fetch_title(url):
        try:
            r = requests.get(url, timeout=10, headers={"User-Agent": UA}, allow_redirects=True)
            if r.status_code != 200:
                return {"url": url, "title": None, "status": r.status_code}
            soup = BeautifulSoup(r.content, "lxml")
            og_title = soup.find("meta", property="og:title")
            title = og_title["content"].strip() if og_title else None
            if not title:
                title_tag = soup.find("title")
                title = title_tag.get_text(strip=True) if title_tag else None
            return {"url": url, "title": title, "status": r.status_code}
        except Exception as e:
            return {"url": url, "title": None, "error": str(e)}
    
    titles = []
    with ThreadPoolExecutor(max_workers=6) as ex:
        futures = list(ex.map(fetch_title, top_urls))
        titles.extend(futures)
    
    return {url_data["url"]: url_data for url_data in titles if url_data.get("title")}


def analyze_headlines(df_pages: pd.DataFrame, title_map: dict) -> dict:
    """
    Verbindet Titles mit Performance-Daten und analysiert Muster.
    """
    enriched = []
    for _, row in df_pages.iterrows():
        url = row["url"]
        title_data = title_map.get(url)
        title = title_data["title"] if title_data else None
        enriched.append({
            "url": url,
            "title": title,
            "clicks": int(row["clicks"] or 0),
            "impressions": int(row["impressions"] or 0),
            "ctr": float(row["ctr"] or 0),
        })
    
    # Nur URLs mit Title
    with_titles = [e for e in enriched if e["title"]]
    if not with_titles:
        return {"error": "Keine og:titles scrapen konnten"}
    
    total_impressions = sum(e["impressions"] for e in with_titles)
    
    def segment_avg(items, key_fn, label_fn):
        """Berechnet Durchschnittswerte für Segmente."""
        groups = defaultdict(lambda: {"count": 0, "clicks": 0, "impressions": 0})
        for item in items:
            key = key_fn(item["title"])
            groups[key]["count"] += 1
            groups[key]["clicks"] += item["clicks"]
            groups[key]["impressions"] += item["impressions"]
        return [
            {
                "label": label_fn(k),
                "count": v["count"],
                "avg_clicks": round(v["clicks"] / v["count"], 1),
                "avg_impressions": round(v["impressions"] / v["count"], 1),
            }
            for k, v in groups.items()
        ]
    
    # --- Analyse 1: Ist es eine Frage? ---
    def is_question(t):
        return t.strip().endswith("?")
    
    q_yes = [e for e in with_titles if is_question(e["title"])]
    q_no  = [e for e in with_titles if not is_question(e["title"])]
    
    def avg_imp(lst):
        return round(sum(x["impressions"] for x in lst) / len(lst), 0) if lst else 0
    def avg_clk(lst):
        return round(sum(x["clicks"] for x in lst) / len(lst), 1) if lst else 0
    
    questions = {
        "question_count": len(q_yes),
        "non_question_count": len(q_no),
        "question_avg_impressions": avg_imp(q_yes),
        "non_question_avg_impressions": avg_imp(q_no),
        "question_avg_clicks": avg_clk(q_yes),
        "non_question_avg_clicks": avg_clk(q_no),
        "winner": "Fragen" if avg_imp(q_yes) > avg_imp(q_no) else "Keine Fragen",
        "diff_pct": round((avg_imp(q_yes) - avg_imp(q_no)) / max(avg_imp(q_no), 1) * 100, 1),
    }
    
    # --- Analyse 2: Entity-Prefix (Wort vor Doppelpunkt) ---
    def has_entity_prefix(t):
        return bool(re.match(r'^[^:]{2,30}:\s', t))
    
    ep_yes = [e for e in with_titles if has_entity_prefix(e["title"])]
    ep_no  = [e for e in with_titles if not has_entity_prefix(e["title"])]
    
    entity_prefix = {
        "entity_count": len(ep_yes),
        "non_entity_count": len(ep_no),
        "entity_avg_impressions": avg_imp(ep_yes),
        "non_entity_avg_impressions": avg_imp(ep_no),
        "entity_avg_clicks": avg_clk(ep_yes),
        "winner": "Entity-Prefix" if avg_imp(ep_yes) > avg_imp(ep_no) else "Kein Prefix",
        "diff_pct": round((avg_imp(ep_yes) - avg_imp(ep_no)) / max(avg_imp(ep_no), 1) * 100, 1),
        "top_entities": Counter(
            re.match(r'^([^:]{2,30}):', t["title"]).group(1) 
            for t in ep_yes 
            if re.match(r'^([^:]{2,30}):', t["title"])
        ).most_common(8),
    }
    
    # --- Analyse 3: Headline-Länge ---
    def length_bucket(t):
        n = len(t)
        if n < 50:    return "<50"
        if n <= 70:   return "50–70"
        if n <= 100:  return "71–100"
        return ">100"
    
    length_groups = defaultdict(list)
    for e in with_titles:
        bucket = length_bucket(e["title"])
        length_groups[bucket].append(e)
    
    length_analysis = []
    for bucket in ["<50", "50–70", "71–100", ">100"]:
        items = length_groups.get(bucket, [])
        length_analysis.append({
            "bucket": bucket,
            "count": len(items),
            "avg_impressions": avg_imp(items),
            "avg_clicks": avg_clk(items),
        })
    
    best_length_bucket = max(length_analysis, key=lambda x: x["avg_impressions"])
    over_100 = [e for e in with_titles if len(e["title"]) > 100]
    
    # --- Analyse 4: Zahlen im Titel ---
    def has_number(t):
        return bool(re.search(r'\b\d+\b', t))
    
    num_yes = [e for e in with_titles if has_number(e["title"])]
    num_no  = [e for e in with_titles if not has_number(e["title"])]
    
    numbers = {
        "with_number_count": len(num_yes),
        "without_number_count": len(num_no),
        "with_number_avg_impressions": avg_imp(num_yes),
        "without_number_avg_impressions": avg_imp(num_no),
        "winner": "Mit Zahl" if avg_imp(num_yes) > avg_imp(num_no) else "Ohne Zahl",
        "diff_pct": round((avg_imp(num_yes) - avg_imp(num_no)) / max(avg_imp(num_no), 1) * 100, 1),
    }
    
    # --- Analyse 5: Sentiment-Signalwörter ---
    SENTIMENT_WORDS = [
        "exklusiv", "schockierend", "endlich", "warnung", "verboten", "verbote",
        "kult", "wahnsinn", "skandal", "jetzt", "aktuell", "dringend",
        "geheimnis", "enthüllt", "verrät", "so läuft", "niemand weiß",
        "so einfach", "so geht", "das steckt", "das ist der grund",
        "deshalb", "darum", "weil", "denn", "achtung",
    ]
    
    def has_sentiment(t):
        t_lower = t.lower()
        return any(w in t_lower for w in SENTIMENT_WORDS)
    
    sent_yes = [e for e in with_titles if has_sentiment(e["title"])]
    sent_no  = [e for e in with_titles if not has_sentiment(e["title"])]
    
    sentiment = {
        "with_signal_count": len(sent_yes),
        "without_signal_count": len(sent_no),
        "with_signal_avg_impressions": avg_imp(sent_yes),
        "without_signal_avg_impressions": avg_imp(sent_no),
        "winner": "Mit Signal" if avg_imp(sent_yes) > avg_imp(sent_no) else "Ohne Signal",
        "diff_pct": round((avg_imp(sent_yes) - avg_imp(sent_no)) / max(avg_imp(sent_no), 1) * 100, 1),
        "top_words": Counter(
            w for e in sent_yes for w in SENTIMENT_WORDS if w in e["title"].lower()
        ).most_common(6),
    }
    
    # --- Freshness-Cues ---
    FRESHNESS = ["jetzt", "heute", "aktuell", "2024", "2025", "2026", "gerade", "live", "update", "neu"]
    fresh_yes = [e for e in with_titles if any(w in e["title"].lower() for w in FRESHNESS)]
    fresh_no  = [e for e in with_titles if not any(w in e["title"].lower() for w in FRESHNESS)]
    
    freshness = {
        "with_freshness_count": len(fresh_yes),
        "without_freshness_count": len(fresh_no),
        "with_freshness_avg_impressions": avg_imp(fresh_yes),
        "without_freshness_avg_impressions": avg_imp(fresh_no),
        "winner": "Mit Freshness-Cue" if avg_imp(fresh_yes) > avg_imp(fresh_no) else "Ohne Freshness-Cue",
        "diff_pct": round((avg_imp(fresh_yes) - avg_imp(fresh_no)) / max(avg_imp(fresh_no), 1) * 100, 1),
    }
    
    # --- Top-Performer: gemeinsame Muster ---
    top_20_pct = with_titles[:max(1, len(with_titles)//5)]  # obere 20% nach Position (bereits sortiert nach Impressionen)
    top_words = Counter(
        word.lower() for e in top_20_pct
        if e["title"] 
        for word in re.findall(r'\b[a-zA-ZäöüÄÖÜß]{4,}\b', e["title"])
        if word.lower() not in {"dass", "sich", "oder", "wird", "sind", "auch", "nach", "beim",
                                 "eine", "eine", "einem", "einen", "ihrer", "diese", "haben",
                                 "sein", "mehr", "über", "unter", "durch", "beim", "wird", "kann",
                                 "wann", "warum", "welche", "welchen", "welches", "wurde", "wurden"}
    ).most_common(15)
    
    return {
        "total_with_titles": len(with_titles),
        "questions": questions,
        "entity_prefix": entity_prefix,
        "length": {
            "analysis": length_analysis,
            "best_bucket": best_length_bucket,
            "over_100_count": len(over_100),
            "over_100_examples": [e["title"] for e in over_100[:5]],
        },
        "numbers": numbers,
        "sentiment": sentiment,
        "freshness": freshness,
        "top_words_in_top_performers": top_words,
    }
```

### Phase 7: CTR-Ausreißer und Long-tail-Analyse

```python
def analyze_ctr_outliers(data: dict, section_data: dict) -> dict:
    """
    - High impressions, low CTR: Discover zeigt den Artikel, User klicken nicht
    - High CTR, low impressions: Gute Formel, aber zu wenig Reichweite
    - Long-tail performers: Artikel mit Impressionen über viele Tage verteilt
    """
    df = data["pages"]
    if df is None:
        return {}
    
    baseline_ctr = df["ctr"].median()
    
    # High impressions / low CTR (potential waste)
    high_imp_threshold = df["impressions"].quantile(0.75)
    low_ctr_threshold  = df["ctr"].quantile(0.25)
    
    potential_headline_fixes = df[
        (df["impressions"] >= high_imp_threshold) &
        (df["ctr"] <= low_ctr_threshold)
    ].head(20)
    
    # High CTR / low impressions (good formula, not seeded)
    high_ctr_threshold = df["ctr"].quantile(0.75)
    low_imp_threshold  = df["impressions"].quantile(0.25)
    
    good_formula = df[
        (df["ctr"] >= high_ctr_threshold) &
        (df["impressions"] <= low_imp_threshold)
    ].head(10)
    
    # Single-article breakout per Ressort
    breakout_flags = []
    if "sections" in section_data:
        for sec in section_data["sections"]:
            if sec["article_count"] >= 3:
                sec_total_clicks = sec["total_clicks"]
                # Stichprobe: prüfe ob eine URL >50% der Sektion-Klicks hat
                sec_urls = df[df["url"].str.contains(sec["section"].strip("/"), regex=False, na=False)]
                if len(sec_urls) > 0:
                    max_clicks = sec_urls["clicks"].max()
                    if sec_total_clicks > 0 and max_clicks / sec_total_clicks > 0.5:
                        top_url = sec_urls.loc[sec_urls["clicks"].idxmax(), "url"]
                        breakout_flags.append({
                            "section": sec["section"],
                            "top_url": top_url,
                            "share": round(max_clicks / sec_total_clicks * 100, 1),
                        })
    
    return {
        "baseline_ctr": round(baseline_ctr * 100, 2),
        "potential_headline_fixes": potential_headline_fixes[["url", "clicks", "impressions", "ctr"]].to_dict("records"),
        "good_formula_low_reach": good_formula[["url", "clicks", "impressions", "ctr"]].to_dict("records"),
        "breakout_flags": breakout_flags,
    }
```

### Phase 8: Actionable Insights generieren

```python
def generate_actionable_insights(
    limit_check: dict,
    scope: dict,
    indexation: dict,
    sections: dict,
    headlines: dict,
    ctr_analysis: dict,
) -> list[dict]:
    """
    Leitet die 3–6 wichtigsten konkreten Handlungsempfehlungen ab.
    Jeder Insight hat: priority (1–3), title, detail, metric.
    """
    insights = []
    
    # Insight: 1000-Zeilen-Limit
    if limit_check.get("truncated"):
        top_sugg = limit_check.get("suggestions", [{}])[0]
        insights.append({
            "priority": 1,
            "icon": "⚠️",
            "title": "Unvollständige Daten — Analyse nach Ressort filtern",
            "detail": (
                f"Der Export enthält genau 1000 Zeilen — das GSC-Limit. "
                f"Alle Zahlen und Empfehlungen in diesem Report sind verzerrt. "
                f"Exportiere den Report neu mit dem Filter '{top_sugg.get('gsc_filter', '')}' "
                f"(~{top_sugg.get('estimated_share', '?')}% der URLs im aktuellen Export)."
            ),
            "metric": "1000/1000 Zeilen",
        })
    
    # Insight: Schwache Ressorts die trotzdem viel publizieren
    if sections.get("weak_sections"):
        for sec in sections["weak_sections"][:2]:
            insights.append({
                "priority": 2,
                "icon": "📉",
                "title": f"Ressort {sec['section']}: publiziert viel, kaum Discover-Erfolg",
                "detail": (
                    f"{sec['article_count']} Artikel mit Ø {sec['avg_clicks_per_article']:,} Klicks/Artikel "
                    f"({sec['baseline_ratio']}× Baseline). Entweder Inhaltstyp für Discover optimieren "
                    f"oder Publish-Frequenz reduzieren und Ressourcen umverteilen."
                ),
                "metric": f"Ø {sec['avg_clicks_per_article']:,} Klicks/Artikel",
            })
    
    # Insight: Hidden Gems
    if sections.get("hidden_gems"):
        for gem in sections["hidden_gems"][:1]:
            insights.append({
                "priority": 1,
                "icon": "💎",
                "title": f"Ressort {gem['section']}: wenige Artikel, aber überdurchschnittlich stark",
                "detail": (
                    f"Nur {gem['article_count']} Artikel, aber Ø {gem['avg_clicks_per_article']:,} Klicks "
                    f"({gem['baseline_ratio']}× Baseline). Mehr Content in diesem Bereich publizieren."
                ),
                "metric": f"{gem['baseline_ratio']}× Baseline",
            })
    
    # Insight: Headline-Länge
    if headlines and "length" in headlines:
        best = headlines["length"]["best_bucket"]
        over_100 = headlines["length"]["over_100_count"]
        if over_100 > 0:
            insights.append({
                "priority": 2,
                "icon": "✂️",
                "title": f"{over_100} Headlines über 100 Zeichen — kürzen",
                "detail": (
                    f"Die besten Headlines im Datensatz sind {best['bucket']} Zeichen lang "
                    f"(Ø {best['avg_impressions']:,} Impressionen). "
                    f"Headlines über 100 Zeichen werden in Discover abgeschnitten."
                ),
                "metric": f"{over_100} zu lange Headlines",
            })
    
    # Insight: Entity-Prefix
    if headlines and "entity_prefix" in headlines:
        ep = headlines["entity_prefix"]
        if abs(ep["diff_pct"]) >= 15:
            winner_detail = (
                f"Headlines mit Entity-Prefix (z.B. 'FC Bayern: ...') erzielen "
                f"{ep['diff_pct']:+.0f}% {'mehr' if ep['diff_pct'] > 0 else 'weniger'} Impressionen."
            )
            insights.append({
                "priority": 3,
                "icon": "🏷️",
                "title": f"Entity-Prefix {'funktioniert' if ep['diff_pct'] > 0 else 'schadet'} in Discover",
                "detail": winner_detail,
                "metric": f"{ep['diff_pct']:+.0f}% Impressionen-Differenz",
            })
    
    # Insight: Fragen
    if headlines and "questions" in headlines:
        q = headlines["questions"]
        if abs(q["diff_pct"]) >= 20 and q["question_count"] >= 5:
            insights.append({
                "priority": 3,
                "icon": "❓",
                "title": f"Frage-Headlines {'übertreffen' if q['diff_pct'] > 0 else 'underperformen'} den Durchschnitt",
                "detail": (
                    f"{q['question_count']} Frage-Headlines mit Ø {int(q['question_avg_impressions']):,} Impressionen "
                    f"vs. {int(q['non_question_avg_impressions']):,} ohne Fragezeichen "
                    f"({q['diff_pct']:+.0f}%)."
                ),
                "metric": f"{q['diff_pct']:+.0f}% Impressionen-Differenz",
            })
    
    # Insight: CTR-Ausreißer
    if ctr_analysis.get("potential_headline_fixes"):
        n = len(ctr_analysis["potential_headline_fixes"])
        insights.append({
            "priority": 2,
            "icon": "🎯",
            "title": f"{n} Artikel mit hohen Impressionen aber schwacher CTR — Headlines überarbeiten",
            "detail": (
                f"Diese Artikel werden Nutzern gezeigt, werden aber nicht geklickt. "
                f"Headline-Überarbeitung kann hier unmittelbar Klicks steigern ohne mehr Reichweite zu benötigen. "
                f"Baseline-CTR: {ctr_analysis['baseline_ctr']}%."
            ),
            "metric": f"Ø CTR < {ctr_analysis['baseline_ctr']}% bei Top-Reichweite",
        })
    
    # Nach Priorität sortieren
    insights.sort(key=lambda x: x["priority"])
    return insights
```

### Phase 9: HTML-Report generieren

Das Report hat diese Sektionen (immer in dieser Reihenfolge):
1. Header (Domain, Zeitraum, Gesamtmetriken)
2. **Actionable Insights** (farbige Karten, immer ganz oben)
3. ⚠️ 1000-Zeilen-Warning (falls zutreffend, direkt nach Insights)
4. Ressort-Performance (Tabelle + 3-Stufen-Farbkodierung)
5. Indexierungsrate (Balkendiagramm als inline SVG)
6. Headline-Analyse (Vergleichstabellen)
7. CTR-Ausreißer
8. Länderdaten
9. Vollständige URL-Tabelle (suchbar)

```python
def generate_html_report(
    filepath: str,
    scope: dict,
    limit_check: dict,
    indexation: dict,
    sections: dict,
    headlines: dict,
    ctr_analysis: dict,
    insights: list,
    data: dict,
) -> str:
    from datetime import date as date_cls

    domain    = scope.get("domain", "unknown")
    period    = scope.get("date_range_raw", "")
    today     = date_cls.today().isoformat()
    out_path  = os.path.expanduser(f"~/Downloads/discover-report-{domain}-{today}.html")
    
    df_pages  = data["pages"]
    df_dates  = data["dates"]
    df_countries = data["countries"]
    
    total_clicks     = int(df_pages["clicks"].sum()) if df_pages is not None else 0
    total_impr       = int(df_pages["impressions"].sum()) if df_pages is not None else 0
    overall_ctr      = round(total_clicks / total_impr * 100, 2) if total_impr else 0
    row_count        = len(df_pages) if df_pages is not None else 0
    
    # --- Actionable Insights HTML ---
    def insight_card(ins):
        colors = {"1": "#C42B19", "2": "#D97706", "3": "#16A34A"}
        color = colors.get(str(ins["priority"]), "#888")
        return f"""
        <div class="insight-card" style="border-left-color:{color}">
          <div class="insight-icon">{ins['icon']}</div>
          <div class="insight-body">
            <div class="insight-title">{ins['title']}</div>
            <div class="insight-detail">{ins['detail']}</div>
          </div>
          <div class="insight-metric">{ins.get('metric','')}</div>
        </div>"""
    
    insights_html = "".join(insight_card(i) for i in insights)
    
    # --- 1000-Zeilen-Warning ---
    limit_html = ""
    if limit_check.get("truncated"):
        sugg_items = "".join(
            f"<li><code>{s['gsc_filter']}</code> — ca. {s['estimated_share']}% der URLs in diesem Export</li>"
            for s in limit_check.get("suggestions", [])[:5]
        )
        limit_html = f"""
        <div class="warning-box">
          <strong>⚠️ Export-Limit erreicht ({limit_check['row_count']} Zeilen)</strong><br>
          {limit_check['warning']}<br>
          <ul style="margin-top:8px">{sugg_items}</ul>
        </div>"""
    
    # --- Ressort-Tabelle ---
    def tier_color(tier):
        return {"strong": "#16A34A", "medium": "#D97706", "weak": "#C42B19"}.get(tier, "#888")
    
    section_rows = ""
    if sections.get("sections"):
        for s in sections["sections"][:30]:
            c = tier_color(s["tier"])
            section_rows += f"""<tr>
              <td><strong>{s['section']}</strong></td>
              <td style="text-align:right">{s['article_count']}</td>
              <td style="text-align:right">{s['total_clicks']:,}</td>
              <td style="text-align:right">{s['total_impressions']:,}</td>
              <td style="text-align:right">{s['avg_clicks_per_article']:,}</td>
              <td style="text-align:right">{s['avg_ctr']}%</td>
              <td style="color:{c};font-weight:600">{s['tier_label']}</td>
            </tr>"""
    
    # --- Indexierungsrate Balkendiagramm (inline SVG) ---
    idx_chart_html = ""
    if indexation.get("periods"):
        periods = indexation["periods"]
        max_rate = max((p["rate"] for p in periods), default=1) or 1
        bar_w = max(3, min(24, 600 // max(len(periods), 1)))
        bars = ""
        for p in periods:
            h = int(p["rate"] / max_rate * 80)
            color = "#16A34A" if p["rate"] >= indexation.get("avg_rate", 10) else "#D97706"
            bars += f'<rect x="{len(bars.split("rect")) * (bar_w + 2)}" y="{90 - h}" width="{bar_w}" height="{h}" fill="{color}" title="{p["period"]}: {p["rate"]}%"/>'
        
        idx_chart_html = f"""
        <div class="section-block">
          <h2>Indexierungsrate ({indexation['granularity']} | Ø {indexation['avg_rate']}%)</h2>
          <p class="meta">Anteil publizierter Artikel die in Discover erschienen sind</p>
          <p>Gesamt: {indexation['total_discovered']} von {indexation['total_sitemap_articles']} Sitemap-Artikeln in Discover sichtbar</p>
          <div style="overflow-x:auto">
            <svg width="{len(periods)*(bar_w+2)+20}" height="100" style="margin-top:8px">
              <line x1="0" y1="90" x2="{len(periods)*(bar_w+2)+20}" y2="90" stroke="var(--border)" stroke-width="1"/>
              {bars}
            </svg>
          </div>
        </div>"""
    
    # --- Headline-Analyse HTML ---
    def cmp_row(label, with_val, without_val, winner, diff_pct, with_n, without_n):
        w = "font-weight:600;color:var(--green)" if winner.startswith(label.split(" ")[0]) else "color:var(--text2)"
        return f"""<tr>
          <td>{label} ({with_n})</td>
          <td style="text-align:right">{int(with_val):,}</td>
          <td>{'← stärker' if diff_pct > 0 else ''}</td>
        </tr>
        <tr>
          <td>Nicht {label.lower()} ({without_n})</td>
          <td style="text-align:right">{int(without_val):,}</td>
          <td>{'← stärker' if diff_pct < 0 else ''}</td>
        </tr>"""
    
    headline_html = ""
    if headlines and "questions" in headlines:
        q  = headlines["questions"]
        ep = headlines["entity_prefix"]
        nm = headlines["numbers"]
        st = headlines["sentiment"]
        fr = headlines["freshness"]
        lb = headlines["length"]
        
        # Längen-Vergleich
        len_rows = "".join(
            f"<tr><td>{r['bucket']} Zeichen ({r['count']})</td>"
            f"<td style='text-align:right'>{int(r['avg_impressions']):,}</td>"
            f"<td style='text-align:right'>{r['avg_clicks']:,}</td>"
            f"{'<td style=\"color:var(--green);font-weight:600\">← Beste</td>' if r['bucket'] == lb['best_bucket']['bucket'] else '<td></td>'}"
            f"</tr>"
            for r in lb["analysis"]
        )
        
        # Top-Entities
        entity_list = "".join(f"<li><em>{e}</em> ({n}×)</li>" for e, n in ep.get("top_entities", []))
        
        # Top-Wörter in Top-Performern
        word_chips = " ".join(
            f'<span class="chip">{w} ({n})</span>' 
            for w, n in headlines.get("top_words_in_top_performers", [])[:12]
        )
        
        headline_html = f"""
        <div class="section-block">
          <h2>Headline-Analyse ({headlines['total_with_titles']} Artikel mit og:title)</h2>
          <div class="hl-grid">
          
            <div class="hl-card">
              <h3>Frage-Headlines</h3>
              <table><thead><tr><th>Typ</th><th>Ø Impressionen</th><th></th></tr></thead><tbody>
              {cmp_row("Frage", q["question_avg_impressions"], q["non_question_avg_impressions"], q["winner"], q["diff_pct"], q["question_count"], q["non_question_count"])}
              </tbody></table>
              <p class="hl-verdict">{'✅' if abs(q['diff_pct']) >= 10 else 'ℹ️'} {q['winner']} {'+' if q['diff_pct'] > 0 else ''}{q['diff_pct']}%</p>
            </div>
            
            <div class="hl-card">
              <h3>Entity-Prefix (z.B. "FC Bayern: …")</h3>
              <table><thead><tr><th>Typ</th><th>Ø Impressionen</th><th></th></tr></thead><tbody>
              {cmp_row("Entity-Prefix", ep["entity_avg_impressions"], ep["non_entity_avg_impressions"], ep["winner"], ep["diff_pct"], ep["entity_count"], ep["non_entity_count"])}
              </tbody></table>
              <p class="hl-verdict">{'✅' if abs(ep['diff_pct']) >= 10 else 'ℹ️'} {ep['winner']} {'+' if ep['diff_pct'] > 0 else ''}{ep['diff_pct']}%</p>
              <ul class="entity-list">{entity_list}</ul>
            </div>
            
            <div class="hl-card">
              <h3>Headline-Länge</h3>
              <table><thead><tr><th>Länge</th><th>Ø Impressionen</th><th>Ø Klicks</th><th></th></tr></thead>
              <tbody>{len_rows}</tbody></table>
              {'<p class="hl-warn">⚠️ ' + str(lb["over_100_count"]) + ' Headlines über 100 Zeichen — werden abgeschnitten</p>' if lb["over_100_count"] > 0 else ''}
            </div>
            
            <div class="hl-card">
              <h3>Zahlen im Titel</h3>
              <table><thead><tr><th>Typ</th><th>Ø Impressionen</th><th></th></tr></thead><tbody>
              {cmp_row("Mit Zahl", nm["with_number_avg_impressions"], nm["without_number_avg_impressions"], nm["winner"], nm["diff_pct"], nm["with_number_count"], nm["without_number_count"])}
              </tbody></table>
              <p class="hl-verdict">{'✅' if abs(nm['diff_pct']) >= 10 else 'ℹ️'} {nm['winner']} {'+' if nm['diff_pct'] > 0 else ''}{nm['diff_pct']}%</p>
            </div>
            
            <div class="hl-card">
              <h3>Sentiment-Signalwörter</h3>
              <table><thead><tr><th>Typ</th><th>Ø Impressionen</th><th></th></tr></thead><tbody>
              {cmp_row("Mit Signal", st["with_signal_avg_impressions"], st["without_signal_avg_impressions"], st["winner"], st["diff_pct"], st["with_signal_count"], st["without_signal_count"])}
              </tbody></table>
              <p class="hl-verdict">{'✅' if abs(st['diff_pct']) >= 10 else 'ℹ️'} {st['winner']} {'+' if st['diff_pct'] > 0 else ''}{st['diff_pct']}%</p>
            </div>
            
            <div class="hl-card">
              <h3>Freshness-Cues (jetzt, heute, aktuell…)</h3>
              <table><thead><tr><th>Typ</th><th>Ø Impressionen</th><th></th></tr></thead><tbody>
              {cmp_row("Mit Freshness", fr["with_freshness_avg_impressions"], fr["without_freshness_avg_impressions"], fr["winner"], fr["diff_pct"], fr["with_freshness_count"], fr["without_freshness_count"])}
              </tbody></table>
              <p class="hl-verdict">{'✅' if abs(fr['diff_pct']) >= 10 else 'ℹ️'} {fr['winner']} {'+' if fr['diff_pct'] > 0 else ''}{fr['diff_pct']}%</p>
            </div>
            
          </div>
          
          <div class="section-block" style="margin-top:20px">
            <h3>Häufige Wörter in Top-20%-Artikeln</h3>
            <div style="margin-top:8px">{word_chips}</div>
          </div>
        </div>"""
    
    # --- CTR-Ausreißer ---
    ctr_rows = ""
    for r in ctr_analysis.get("potential_headline_fixes", [])[:15]:
        url_short = r["url"].split("/")[-1][:60] if "/" in r["url"] else r["url"][:60]
        ctr_pct = round(float(r["ctr"]) * 100, 2)
        ctr_rows += f"""<tr>
          <td><a href="{r['url']}" target="_blank" class="url-link">{url_short}</a></td>
          <td style="text-align:right">{int(r['impressions']):,}</td>
          <td style="text-align:right">{int(r['clicks']):,}</td>
          <td style="text-align:right;color:var(--red)">{ctr_pct}%</td>
        </tr>"""
    
    # --- URL-Tabelle (Top 500) ---
    url_rows = ""
    if df_pages is not None:
        for _, row in df_pages.head(500).iterrows():
            url = row["url"]
            url_short = url.split("/")[-1][:55] if "/" in url else url[:55]
            ctr_pct = round(float(row["ctr"]) * 100, 2)
            url_rows += f"""<tr>
              <td><a href="{url}" target="_blank" class="url-link">{url_short}</a></td>
              <td style="text-align:right">{int(row['clicks']):,}</td>
              <td style="text-align:right">{int(row['impressions']):,}</td>
              <td style="text-align:right">{ctr_pct}%</td>
            </tr>"""
    
    # --- Länder Top 10 ---
    country_rows = ""
    if df_countries is not None:
        for _, row in df_countries.head(10).iterrows():
            ctr_pct = round(float(row["ctr"]) * 100, 2)
            country_rows += f"""<tr>
              <td>{row['country']}</td>
              <td style="text-align:right">{int(row['clicks']):,}</td>
              <td style="text-align:right">{int(row['impressions']):,}</td>
              <td style="text-align:right">{ctr_pct}%</td>
            </tr>"""
    
    # --- Breakout-Flags ---
    breakout_html = ""
    if ctr_analysis.get("breakout_flags"):
        items = "".join(
            f"<li><strong>{b['section']}</strong>: ein Artikel macht {b['share']}% der Klicks. <a href='{b['top_url']}' target='_blank' class='url-link'>URL</a></li>"
            for b in ctr_analysis["breakout_flags"]
        )
        breakout_html = f"""
        <div class="section-block">
          <h2>⚡ Single-Article Breakouts</h2>
          <p class="meta">Ressorts deren Discover-Performance von einem einzigen Artikel dominiert wird — diese Sektionen sollten nicht überbewertet werden.</p>
          <ul style="margin-top:12px;line-height:2">{items}</ul>
        </div>"""
    
    # === HTML zusammenbauen ===
    html = f"""<!DOCTYPE html>
<html lang="de">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>Discover Report — {domain} — {today}</title>
<style>
  :root {{
    --bg:#F3F2EF; --surface:#fff; --border:#DDD9D2; --text:#1B1A17;
    --text2:#6A6660; --red:#C42B19; --amber:#D97706; --green:#16A34A;
  }}
  @media(prefers-color-scheme:dark){{
    :root{{--bg:#131210;--surface:#1D1C1A;--border:#2C2A27;--text:#E6E3DC;--text2:#A09C95;--red:#F87171;--green:#4ADE80;--amber:#FCD34D;}}
  }}
  *{{box-sizing:border-box;margin:0;padding:0}}
  body{{font-family:system-ui,sans-serif;background:var(--bg);color:var(--text);line-height:1.5}}
  .hdr{{background:var(--surface);border-bottom:1px solid var(--border);padding:28px 32px 20px}}
  .hdr-domain{{font-size:22px;font-weight:700}}
  .hdr-meta{{font-size:13px;color:var(--text2);margin-top:4px}}
  .kpi-row{{display:flex;gap:20px;margin-top:16px;flex-wrap:wrap}}
  .kpi{{background:var(--bg);border:1px solid var(--border);border-radius:8px;padding:12px 20px}}
  .kpi-val{{font-size:24px;font-weight:700}}
  .kpi-lbl{{font-size:11px;color:var(--text2);text-transform:uppercase;letter-spacing:.06em;margin-top:2px}}
  .main{{max-width:1100px;margin:0 auto;padding:28px 32px}}
  h2{{font-size:15px;font-weight:700;letter-spacing:.05em;text-transform:uppercase;
      color:var(--text2);margin-bottom:12px}}
  h3{{font-size:13px;font-weight:600;margin-bottom:8px}}
  .meta{{font-size:12px;color:var(--text2);margin-bottom:12px}}
  .section-block{{background:var(--surface);border:1px solid var(--border);border-radius:8px;
                  padding:20px 24px;margin-bottom:20px}}
  .insight-card{{background:var(--surface);border:1px solid var(--border);border-left:4px solid var(--red);
                 border-radius:8px;padding:16px 20px;margin-bottom:10px;
                 display:grid;grid-template-columns:32px 1fr auto;gap:12px;align-items:start}}
  .insight-icon{{font-size:20px;line-height:1}}
  .insight-title{{font-weight:600;font-size:14px;margin-bottom:4px}}
  .insight-detail{{font-size:13px;color:var(--text2)}}
  .insight-metric{{font-size:12px;font-weight:600;color:var(--text2);white-space:nowrap;text-align:right}}
  .warning-box{{background:#FEF2F2;border:1px solid #FECACA;border-radius:8px;padding:16px 20px;
                margin-bottom:20px;font-size:13px;line-height:1.7}}
  @media(prefers-color-scheme:dark){{.warning-box{{background:#3B1515;border-color:#7F1D1D}}}}
  table{{width:100%;border-collapse:collapse;font-size:13px}}
  th{{text-align:left;padding:8px 10px;border-bottom:2px solid var(--border);font-size:11px;
      letter-spacing:.08em;text-transform:uppercase;color:var(--text2)}}
  td{{padding:8px 10px;border-bottom:1px solid var(--border);vertical-align:top}}
  tr:hover td{{background:var(--bg)}}
  .hl-grid{{display:grid;grid-template-columns:repeat(auto-fill,minmax(320px,1fr));gap:16px}}
  .hl-card{{background:var(--bg);border:1px solid var(--border);border-radius:8px;padding:16px}}
  .hl-verdict{{font-size:12px;margin-top:10px;font-weight:600}}
  .hl-warn{{font-size:12px;color:var(--red);margin-top:8px;font-weight:600}}
  .entity-list{{font-size:12px;color:var(--text2);margin-top:8px;padding-left:16px}}
  .entity-list li{{line-height:1.8}}
  .chip{{display:inline-block;background:var(--bg);border:1px solid var(--border);border-radius:20px;
         padding:3px 10px;font-size:12px;margin:2px}}
  .url-link{{color:inherit;text-decoration:none;font-size:11.5px}}
  .url-link:hover{{text-decoration:underline}}
  .tab-bar{{display:flex;gap:6px;border-bottom:2px solid var(--border);margin-bottom:20px;flex-wrap:wrap}}
  .tab{{padding:8px 16px;font-size:13px;font-weight:500;cursor:pointer;border-bottom:2px solid transparent;
        margin-bottom:-2px;color:var(--text2)}}
  .tab.active{{color:var(--text);border-bottom-color:var(--text)}}
  .tab-content{{display:none}}.tab-content.active{{display:block}}
  #url-search{{width:100%;padding:8px 12px;border:1px solid var(--border);border-radius:6px;
               background:var(--surface);color:var(--text);font-size:13px;margin-bottom:12px}}
</style>
</head>
<body>

<div class="hdr">
  <div class="hdr-domain">Google Discover — {domain}</div>
  <div class="hdr-meta">{period} · Exportiert aus GSC · Report erstellt {today} · {row_count} URLs{'  ⚠️ LIMIT ERREICHT' if limit_check.get('truncated') else ''}</div>
  <div class="kpi-row">
    <div class="kpi"><div class="kpi-val">{total_clicks:,}</div><div class="kpi-lbl">Klicks gesamt</div></div>
    <div class="kpi"><div class="kpi-val">{total_impr:,}</div><div class="kpi-lbl">Impressionen gesamt</div></div>
    <div class="kpi"><div class="kpi-val">{overall_ctr}%</div><div class="kpi-lbl">Gesamt-CTR</div></div>
    <div class="kpi"><div class="kpi-val">{row_count}</div><div class="kpi-lbl">Seiten im Export{'  ⚠️' if limit_check.get('truncated') else ''}</div></div>
    {f'<div class="kpi"><div class="kpi-val">{indexation.get("avg_rate","—")}%</div><div class="kpi-lbl">Ø Indexierungsrate</div></div>' if indexation.get("avg_rate") else ''}
  </div>
</div>

<div class="main">

  <div class="tab-bar">
    <div class="tab active" onclick="switchTab('insights_tab',this)">Actionable Insights</div>
    <div class="tab" onclick="switchTab('sections_tab',this)">Ressorts</div>
    <div class="tab" onclick="switchTab('indexation_tab',this)">Indexierung</div>
    <div class="tab" onclick="switchTab('headlines_tab',this)">Headlines</div>
    <div class="tab" onclick="switchTab('ctr_tab',this)">CTR-Analyse</div>
    <div class="tab" onclick="switchTab('urls_tab',this)">Alle URLs</div>
  </div>

  <div id="insights_tab" class="tab-content active">
    {limit_html}
    <h2>Was du jetzt tun kannst</h2>
    {insights_html}
  </div>

  <div id="sections_tab" class="tab-content">
    <div class="section-block">
      <h2>Ressort-Performance</h2>
      <p class="meta">Baseline: Ø {sections.get('baseline_clicks_per_article','—')} Klicks/Artikel · 🟢 ≥1.5× · 🟡 0.6–1.5× · 🔴 &lt;0.6×</p>
      <div style="overflow-x:auto">
      <table>
        <thead><tr><th>Ressort</th><th>Artikel</th><th>Klicks</th><th>Impressionen</th><th>Ø Klicks</th><th>Ø CTR</th><th>Tier</th></tr></thead>
        <tbody>{section_rows}</tbody>
      </table>
      </div>
    </div>
    {breakout_html}
  </div>

  <div id="indexation_tab" class="tab-content">
    {idx_chart_html if idx_chart_html else '<div class="section-block"><p>Keine Sitemap-Daten verfügbar für Indexierungsraten-Berechnung.</p></div>'}
  </div>

  <div id="headlines_tab" class="tab-content">
    {headline_html if headline_html else '<div class="section-block"><p>Keine Headline-Daten (og:title konnte nicht geladen werden).</p></div>'}
  </div>

  <div id="ctr_tab" class="tab-content">
    <div class="section-block">
      <h2>Hohe Impressionen, schwache CTR — Headlines überarbeiten</h2>
      <p class="meta">Baseline CTR: {ctr_analysis.get('baseline_ctr','—')}% · Diese Artikel werden gezeigt, aber nicht geklickt</p>
      <div style="overflow-x:auto">
      <table>
        <thead><tr><th>URL</th><th>Impressionen</th><th>Klicks</th><th>CTR</th></tr></thead>
        <tbody>{ctr_rows}</tbody>
      </table>
      </div>
    </div>
    {f'''<div class="section-block">
      <h2>Länder-Verteilung</h2>
      <table>
        <thead><tr><th>Land</th><th>Klicks</th><th>Impressionen</th><th>CTR</th></tr></thead>
        <tbody>{country_rows}</tbody>
      </table>
    </div>''' if country_rows else ''}
  </div>

  <div id="urls_tab" class="tab-content">
    <div class="section-block">
      <h2>Alle Seiten ({row_count})</h2>
      <input id="url-search" type="search" placeholder="URL suchen…" oninput="filterUrls(this.value)">
      <div style="overflow-x:auto">
      <table id="url-table">
        <thead><tr><th>URL</th><th>Klicks</th><th>Impressionen</th><th>CTR</th></tr></thead>
        <tbody id="url-tbody">{url_rows}</tbody>
      </table>
      </div>
    </div>
  </div>

</div>

<script>
function switchTab(id, el) {{
  document.querySelectorAll('.tab-content').forEach(t => t.classList.remove('active'));
  document.querySelectorAll('.tab').forEach(t => t.classList.remove('active'));
  document.getElementById(id).classList.add('active');
  el.classList.add('active');
}}
function filterUrls(q) {{
  document.querySelectorAll('#url-tbody tr').forEach(row => {{
    row.style.display = row.textContent.toLowerCase().includes(q.toLowerCase()) ? '' : 'none';
  }});
}}
</script>
</body>
</html>"""
    
    with open(out_path, "w", encoding="utf-8") as f:
        f.write(html)
    return out_path
```

---

## Vollständiger Ausführungsfluss

```python
def run_discover_analysis(filepath: str) -> str:
    """Vollständiger Analyse-Durchlauf. Gibt den Pfad zur HTML-Datei zurück."""
    print(f"\n{'='*60}")
    print(f"  Discover Report Analyzer")
    print(f"{'='*60}\n")
    
    # 1. Parsen
    print("📂 Datei einlesen…")
    data = parse_gsc_discover_export(filepath)
    
    # 2. 1000-Zeilen-Check
    print("🔍 Zeilen-Limit prüfen…")
    limit_check = check_row_limit(data)
    if limit_check.get("truncated"):
        print(f"  ⚠️  {limit_check['warning'][:80]}…")
    
    # 3. Scope/Zeitraum
    print("📅 Zeitraum und Domain ermitteln…")
    scope = detect_report_scope(data)
    print(f"  Domain: {scope['domain']}, Zeitraum: {scope['date_range_raw']} ({scope['days']} Tage, {scope['granularity']})")
    
    # 4. Sitemap für Indexierungsrate
    indexation = {}
    if scope.get("domain"):
        print("🗺️  Sitemaps für Indexierungsrate laden…")
        try:
            sitemap_articles = fetch_articles_from_sitemaps(scope["domain"], scope)
            print(f"  {len(sitemap_articles)} Artikel aus Sitemaps geladen")
            indexation = calculate_indexation_rate(data, scope, sitemap_articles)
            print(f"  Indexierungsrate Ø: {indexation.get('avg_rate','?')}%")
        except Exception as e:
            print(f"  ! Sitemap-Fetch fehlgeschlagen: {e}")
            indexation = {}
    
    # 5. Ressort-Analyse
    print("📊 Ressort-Performance analysieren…")
    sections = analyze_sections(data, scope)
    print(f"  {len(sections.get('sections',[]))} Ressorts analysiert, Baseline: {sections.get('baseline_clicks_per_article','?')} Klicks/Artikel")
    
    # 6. Headline-Analyse
    print("📰 og:title scrapen und Headlines analysieren…")
    headlines = {}
    if data["pages"] is not None and len(data["pages"]) > 0:
        try:
            title_map = scrape_og_titles(data["pages"], max_urls=150)
            print(f"  {len(title_map)} Titles geladen")
            headlines = analyze_headlines(data["pages"], title_map)
        except Exception as e:
            print(f"  ! Headline-Analyse fehlgeschlagen: {e}")
    
    # 7. CTR-Analyse
    print("🎯 CTR-Ausreißer identifizieren…")
    ctr_analysis = analyze_ctr_outliers(data, sections)
    
    # 8. Actionable Insights
    print("💡 Actionable Insights generieren…")
    insights = generate_actionable_insights(
        limit_check, scope, indexation, sections, headlines, ctr_analysis
    )
    print(f"  {len(insights)} Insights generiert")
    
    # 9. HTML-Report
    print("🖥️  HTML-Report generieren…")
    out_path = generate_html_report(
        filepath, scope, limit_check, indexation, sections,
        headlines, ctr_analysis, insights, data
    )
    print(f"\n✅  Report: {out_path}")
    return out_path
```

---

## Abhängigkeiten

```
openpyxl      # XLSX-Parsing
pandas        # Datenverarbeitung
requests      # HTTP für Sitemaps + og:title
beautifulsoup4 + lxml  # HTML/XML-Parsing
```

---

## Output-Pfad

```
~/Downloads/discover-report-{domain}-{YYYY-MM-DD}.html
```

Keine externen Abhängigkeiten im HTML — vollständig self-contained.
