---
name: artikel-link-checker
description: Crawlt die News-Sitemap eines Publishers, scrapt alle Artikel vom Vortag und analysiert die interne Verlinkungsqualität — Artikel ohne Links, Link-Verteilung, First-Paragraph-Links, Autorseiten, H1/H2-Ähnlichkeit und SEO-Score. Use when asked to check article linking quality, find articles without internal links, or analyze linking patterns for a news publisher.
tools: Read, Edit, Write, Bash
---

# Artikel-Link-Checker

Tägliche Qualitätsmessung der internen Verlinkung auf Artikel-Ebene. Crawlt die News-Sitemap, scrapt alle Artikel vom Vortag parallel und produziert ein strukturiertes JSON-Tagesfile + History.

---

## 1. Was der Check misst

| Dimension | Was |
|---|---|
| **Artikel ohne Links** | Free-Artikel (kein Video, kein Plus) mit `link_count == 0` |
| **Link-Verteilung** | Bucket-Histogramm: 0 / 1 / 2 / 3 / 4 / 5+ Links |
| **Intern vs. Extern** | Pro Artikel: mehr interne, mehr externe oder gleich viele |
| **First-Paragraph** | Hat der erste `<p>` im `<main>` mindestens einen `text-link`? |
| **Autorseiten** | Separater Check für `/autor/`-URLs |
| **H1/H2-Ähnlichkeit** | SequenceMatcher-Ratio zwischen H1-Tag und `h2.document-title` |
| **SEO-Score** | Punktesystem (H3-Anzahl, Links, Keyword in H3, Rich Media, Meta, etc.) |

Links zählen als `class="text-link"` im `<main id="main">`. Bildlinks (`images.`, `bildstatic.de`) und Links in `.video-centre` werden ignoriert.

---

## 2. Einstiegspunkt: Sitemap holen + Vortags-Artikel filtern

```python
import requests
from bs4 import BeautifulSoup
from datetime import datetime, timedelta
import pytz

TZ_BERLIN = pytz.timezone("Europe/Berlin")
USER_AGENT = "SitemapLinkChecker/1.0 (Firefox)"
SITEMAP_URL = "https://www.bild.de/sitemap-news.xml"  # anpassen je Publisher

def fetch_sitemap() -> BeautifulSoup | None:
    resp = requests.get(SITEMAP_URL, timeout=10, headers={"User-Agent": USER_AGENT})
    resp.raise_for_status()
    return BeautifulSoup(resp.content, "xml")

def get_yesterday_articles(soup: BeautifulSoup) -> list[dict]:
    yesterday_date = (datetime.now(TZ_BERLIN) - timedelta(days=1)).date()
    articles = []
    for url_elem in soup.find_all("url"):
        loc = url_elem.find("loc")
        news = url_elem.find("news:news")
        if not loc or not news:
            continue
        date_tag = news.find("news:publication_date")
        if not date_tag:
            continue
        pub_dt = datetime.fromisoformat(date_tag.string.replace("Z", "+00:00"))
        if pub_dt.astimezone(TZ_BERLIN).date() == yesterday_date:
            title_tag = news.find("news:title")
            articles.append({
                "url": loc.string,
                "title": title_tag.string if title_tag else "Unknown",
                "published_date": date_tag.string,
            })
    return articles
```

---

## 3. Artikel scrapen (parallel, 10 Worker)

```python
from concurrent.futures import ThreadPoolExecutor
from bs4 import BeautifulSoup
from difflib import SequenceMatcher
from urllib.parse import urlparse

def scrape_article(url: str) -> dict:
    resp = requests.get(url, timeout=10, headers={"User-Agent": USER_AGENT})
    resp.raise_for_status()
    soup = BeautifulSoup(resp.content, "lxml")

    # Video / Plus-Artikel erkennen
    og_type = soup.find("meta", property="og:type")
    is_video = og_type and og_type.get("content") == "video"
    is_plus = bool(
        soup.find("div", class_="paywall")
        or soup.find("span", class_="plus-logo")
        or (lambda m: m and "isPremium=true" in m.get("content", ""))(
            soup.find("meta", attrs={"name": "ob:extras"})
        )
    )

    main = soup.find("main", id="main") or soup.find("main")
    link_count = first_para_links = internal = external = 0
    links_detail = []

    if main and not is_video:
        for a in main.find_all("a", class_="text-link"):
            href = a.get("href", "")
            if "images." in href.lower() or "bildstatic.de" in href.lower():
                continue
            if a.find_parent("div", class_="video-centre"):
                continue
            link_count += 1
            links_detail.append({"href": href, "anchor_text": a.get_text(strip=True)})
            if href.startswith("/") or "bild.de" in href.lower():
                internal += 1
            else:
                external += 1

        first_p = main.find("p")
        if first_p:
            first_para_links = sum(
                1 for a in first_p.find_all("a", class_="text-link")
                if "images." not in a.get("href", "").lower()
                and "bildstatic.de" not in a.get("href", "").lower()
                and not a.find_parent("div", class_="video-centre")
            )

    # H1 / H2 Ähnlichkeit
    h1 = soup.find("h1")
    h1_text = h1.get_text(strip=True) if h1 else ""
    h2 = soup.find("h2", class_=lambda c: c and "document-title" in c) if not is_video else None
    h2_text = h2.get_text(strip=True) if h2 else ""
    similarity = round(SequenceMatcher(None, h1_text.lower(), h2_text.lower()).ratio() * 100, 1) if h1_text and h2_text else 0.0

    # H3, Meta, Rich Media
    h3s = [h.get_text(strip=True) for h in (main or soup).find_all("h3") if h.get_text(strip=True)]
    meta_desc = ""
    for sel in [{"name": "description"}, {"property": "og:description"}]:
        tag = soup.find("meta", attrs=sel)
        if tag:
            meta_desc = tag.get("content", "")
            break
    page_title = soup.find("title")
    page_title = page_title.get_text(strip=True) if page_title else ""
    captions = [f.get_text(strip=True) for f in (main or soup).find_all("figcaption") if f.get_text(strip=True)]

    main_html = str(main).lower() if main else ""
    return {
        "url": url, "link_count": link_count, "first_paragraph_link_count": first_para_links,
        "is_video": int(is_video), "is_plus_article": int(is_plus),
        "internal_links": internal, "external_links": external, "links_detail": links_detail,
        "h1_text": h1_text, "h2_full_text": h2_text, "h1_h2_similarity": similarity,
        "h3_headings": h3s, "meta_description": meta_desc, "page_title": page_title,
        "img_captions": captions,
        "has_table": "<table" in main_html, "has_list": "<ul" in main_html or "<ol" in main_html,
        "embedded_video_count": main_html.count("video-centre"),
        "has_widget": any(x in main_html for x in ["<iframe", "twitter-tweet", "instagram-media"]),
        "has_infographic": any(x in main_html for x in ["<svg", "infografik", "infographic"]),
        "error": None,
    }

def scrape_parallel(articles: list[dict], max_workers: int = 10) -> dict[str, dict]:
    def worker(article):
        result = scrape_article(article["url"])
        result.update({"title": article["title"], "published_date": article["published_date"]})
        return article["url"], result

    with ThreadPoolExecutor(max_workers=max_workers) as ex:
        return dict(ex.map(worker, articles))
```

---

## 4. SEO-Score berechnen

Jeder Artikel bekommt einen Score aus diesen Einzelkriterien (kein Maximum, additive Punkte):

```python
GERMAN_STOP_WORDS = {
    "dass", "oder", "aber", "auch", "nach", "beim", "sein", "ihre", "ihren",
    "haben", "hatte", "wird", "wurden", "wurde", "durch", "über", "unter",
    "von", "vom", "dem", "den", "der", "die", "das", "eine", "einen",
    # ... vollständige Liste in run_check.py
}
GENERIC_ANCHORS = {
    "hier", "hier klicken", "mehr", "mehr lesen", "weiter", "zum artikel",
    "jetzt lesen", "weiterlesen", ">>>", "»", "«",
}

def calculate_seo_score(article: dict) -> dict:
    keywords = {
        w.strip('.,!?:;-"\'()[]{}»«')
        for w in (article.get("h2_full_text") or article.get("h1_text", "")).lower().split()
        if len(w.strip('.,!?:;-"\'()[]{}»«')) >= 4
        and w.strip('.,!?:;-"\'()[]{}»«') not in GERMAN_STOP_WORDS
    }
    h3s = article.get("h3_headings", [])
    links = article.get("links_detail", [])
    score = 0
    criteria = {}

    score += (h3_pts := len(h3s));                          criteria["h3_count"] = h3_pts
    score += (link_pts := min(article.get("link_count", 0), 8)); criteria["links"] = link_pts
    score += (kw_h3 := sum(1 for h in h3s if keywords and any(k in h.lower() for k in keywords))); criteria["keyword_in_h3"] = kw_h3

    rich = {k: bool(article.get(k, 0)) for k in ["has_table", "embedded_video_count", "has_widget", "has_list", "has_infographic"]}
    score += (rich_pts := sum(rich.values()));              criteria["rich_media"] = rich_pts

    score += (no_fp := int(article.get("first_paragraph_link_count", 0) == 0)); criteria["no_first_para_link"] = no_fp
    hrefs = [l["href"] for l in links]
    score += (no_dupes := int(not hrefs or len(hrefs) == len(set(hrefs))));      criteria["no_duplicate_links"] = no_dupes

    captions = " ".join(article.get("img_captions", [])).lower()
    score += (kw_cap := int(bool(keywords and article.get("img_captions") and any(k in captions for k in keywords)))); criteria["keyword_in_caption"] = kw_cap

    score += (anch_pts := sum(
        1 for l in links
        if l.get("anchor_text", "").strip().lower() not in GENERIC_ANCHORS
        and len(l.get("anchor_text", "").strip()) >= 3
    ));                                                      criteria["anchor_texts"] = anch_pts

    title = article.get("page_title", "").lower()
    score += (kw_title := int(bool(keywords and any(k in title for k in keywords)))); criteria["keyword_in_title"] = kw_title
    score += (title_ok := int(0 < len(article.get("page_title", "")) < 100));         criteria["title_length"] = title_ok

    meta = article.get("meta_description", "")
    score += (meta_ok := int(0 < len(meta) < 160));         criteria["meta_desc_length"] = meta_ok

    title_words = {w.strip(".,!?") for w in title.split()}
    desc_words  = {w.strip(".,!?") for w in meta.lower().split()}
    score += (meta_extra := int(len(desc_words - title_words - GERMAN_STOP_WORDS) >= 3)); criteria["meta_desc_extra"] = meta_extra

    score += (uniq_h3 := sum(1 for h in h3s if len(h.split()) >= 3 or any(c.isdigit() for c in h))); criteria["unique_h3"] = uniq_h3

    return {"total": score, "criteria": criteria}
```

---

## 5. Ressort aus URL ableiten (bild.de)

```python
from urllib.parse import urlparse

RESSORT_MAPPING = {
    "sport": "Sport", "unterhaltung": "Unterhaltung", "leben": "Leben",
    "politik": "Politik", "wirtschaft": "Wirtschaft",
    "regional": "Regional", "bayern": "Regional", "berlin": "Regional",
    "hamburg": "Regional", "nrw": "Regional", "frankfurt": "Regional",
    "muenchen": "Regional", "köln": "Regional", "duesseldorf": "Regional",
    "niedersachsen": "Regional", "sachsen": "Regional", "schleswig-holstein": "Regional",
    "news": "News", "us": "News", "world": "News", "bild-plus": "News",
    "author": "News", "autor": "News",
}

def get_ressort(url: str) -> str:
    parts = urlparse(url).path.split("/")
    return RESSORT_MAPPING.get(parts[1].lower() if len(parts) > 1 else "", "News")

def is_autor_page(url: str) -> bool:
    return "/autor/" in urlparse(url).path
```

---

## 6. Output-Format (Tages-JSON)

Jede tägliche Analyse wird in `data/YYYY-MM-DD.json` geschrieben:

```json
{
  "stats": {
    "run_date": "2026-08-25",
    "total": 312,
    "with_links": 241,
    "without_links": 71,
    "pct_without_links": 22.8
  },
  "articles_without_links": [
    {
      "url": "https://www.bild.de/politik/...",
      "title": "Artikeltitel",
      "category": "Politik",
      "link_count": 0,
      "published_date": "2026-08-24T08:30:00Z"
    }
  ],
  "link_distribution": {
    "0_links": 71, "1_link": 42, "2_links": 55,
    "3_links": 48, "4_links": 39, "5plus_links": 57
  },
  "internal_external": { "more_internal": 198, "more_external": 31, "equal": 83 },
  "totals": { "total_internal": 890, "total_external": 120, "pct_internal": 88.1 },
  "first_paragraph": {
    "stats": { "total": 312, "with_links": 95, "without_links": 217, "pct_with_links": 30.4 },
    "categories": { "Sport": { "total": 80, "with_links": 22, "pct_with_links": 27.5 } }
  },
  "autorenseiten": {
    "stats": { "total": 14, "with_links": 11, "without_links": 3, "pct_without_links": 21.4 },
    "articles_without_links": []
  },
  "h1_h2": {
    "stats": { "pct_above_90": 72.1, "pct_above_80_plus": 84.3, "average_similarity": 86.5 },
    "articles": []
  }
}
```

`data/history.json` enthält die letzten 30 Tages-Summaries für den Trend-Chart.

---

## 7. Vollständiger Run

```python
import json, os

DATA_DIR = "data"

def run_check(data_dir: str = DATA_DIR) -> dict:
    os.makedirs(data_dir, exist_ok=True)

    sitemap = fetch_sitemap()
    articles = get_yesterday_articles(sitemap)
    scraped  = scrape_parallel(articles)

    autor_articles   = {u: d for u, d in scraped.items() if is_autor_page(u)}
    regular_articles = {u: d for u, d in scraped.items() if not is_autor_page(u)}
    free_articles    = {u: d for u, d in regular_articles.items()
                        if not d["is_video"] and not d["is_plus_article"]}

    for a in free_articles.values():
        a["category"]  = get_ressort(a["url"])
        a["seo_score"] = calculate_seo_score(a)

    run_date = (datetime.now(TZ_BERLIN) - timedelta(days=1)).date().isoformat()

    # ... aggregiere Statistiken (link_distribution, internal_external, first_paragraph, etc.)
    # ... schreibe data/{run_date}.json
    # ... hänge summary an data/history.json (max 30 Einträge)
```

---

## 8. Technische Abhängigkeiten

| Paket | Zweck |
|---|---|
| `requests` | HTTP-Calls (Sitemap + Artikel) |
| `beautifulsoup4` + `lxml` | HTML-Parsing |
| `lxml` (xml-mode) | Sitemap-Parsing |
| `pytz` | Berlin-Timezone für Datums-Matching |
| `concurrent.futures` | Parallel-Scraping (ThreadPoolExecutor) |

Keine externen API-Keys erforderlich — rein HTTP-basiert.

---

## 9. Konfiguration

| Parameter | Wert | Begründung |
|---|---|---|
| Sitemap-URL | `https://www.bild.de/sitemap-news.xml` | Publisher anpassen |
| Link-Klasse | `class="text-link"` | BILD-spezifisch; anpassen bei anderem Publisher |
| Max Workers | 10 | ~13 Sek. für 325 Artikel |
| Timeout | 10 s | Pro Request |
| History-Länge | 30 Tage | Rollierendes Fenster im Trend-Chart |
| Link-Cap für Score | 8 | Mehr als 8 Links bringt keinen Extra-Score |
