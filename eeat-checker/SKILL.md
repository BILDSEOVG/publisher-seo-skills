---
name: eeat-checker
description: Analysiert einen Artikel oder Text auf E-E-A-T-Qualität anhand von 8 operationalisierten Fragen. Prüft automatisch strukturelle HTML-Signale (Schema.org, Autorseiten-Link, YMYL-Thema) und analysiert den Inhalt per LLM. Liefert eine Bewertung pro Dimension mit konkreten Belegen und Handlungsempfehlungen. Use when asked to check an article for E-E-A-T quality, evaluate content trustworthiness, assess author expertise visibility, or audit editorial credibility for Google Search Quality purposes.
tools: Read, Edit, Write, Bash
---

# E-E-A-T Checker

Bewertet einen Artikel auf alle acht E-E-A-T-Kerndimensionen. Ergebnis: Ampel-Bewertung pro Dimension, Belege aus dem Text, konkrete Handlungsempfehlungen.

---

## Was E-E-A-T ist (und warum die 8 Fragen entscheidend sind)

E-E-A-T steht für **Experience, Expertise, Authoritativeness, Trustworthiness** — das zentrale Qualitätskonzept in Googles Quality Rater Guidelines (QRG). Google nutzt E-E-A-T nicht direkt als Ranking-Signal, sondern als Trainingsdaten für die Qualitätsbewertung durch Rater, die wiederum die Core-Update-Richtung beeinflussen.

Die 8 Fragen in diesem Skill operationalisieren E-E-A-T auf Artikel-Ebene:

| # | Frage | E-E-A-T-Dimension |
|---|---|---|
| 1 | Was weiß dieser Artikel, was die anderen nicht wissen? | Experience + Expertise |
| 2 | Woher wissen wir das? | Trustworthiness |
| 3 | Sind die entscheidenden Behauptungen durch Primär-/seriöse Quellen belegt? | Trustworthiness |
| 4 | Hat Autor oder Redaktion echte Erfahrung bzw. Expertise zum Thema? | Experience + Expertise |
| 5 | Wird diese Expertise im Inhalt sichtbar oder nur im Autorenkasten behauptet? | Experience + Expertise |
| 6 | Gibt es eigene Analyse, Einordnung oder Beispiele statt bloßer Zusammenfassung? | Experience + Expertise |
| 7 | Kann ein Leser nachvollziehen, wer den Inhalt erstellt und wie recherchiert wurde? | Authoritativeness + Trustworthiness |
| 8 | Würde ein fachkundiger Dritter den Text für korrekt und vertrauenswürdig halten? | Trustworthiness + Authoritativeness |

**Wichtig für Publisher:** YMYL-Themen (Your Money Your Life — Gesundheit, Finanzen, Recht, Sicherheit) unterliegen deutlich höheren E-E-A-T-Anforderungen. Der Checker erkennt YMYL-Themen automatisch und passt die Bewertung an.

---

## Workflow-Übersicht

```
Input (URL oder Text)
  ↓
Schritt 1: Extraktion (HTML-Crawl oder Text-Bereinigung)
  ↓
Schritt 2: Strukturelle Signale (Schema.org, Autorseite, YMYL)
  ↓
Schritt 3: LLM-Analyse der 8 Dimensionen
  ↓
Schritt 4: Report ausgeben (Terminal + optionales HTML)
```

---

## Schritt 1: Extraktion

### Variante A — URL (bevorzugt)

```python
import requests, json, re
from bs4 import BeautifulSoup

UA = "EEATChecker/1.0 (Mozilla Firefox)"

def fetch_article(url: str) -> dict:
    r = requests.get(url, timeout=15, headers={"User-Agent": UA}, allow_redirects=True)
    r.raise_for_status()
    soup = BeautifulSoup(r.content, "lxml")

    # Haupttext — Main oder Article-Tag bevorzugen
    main = (
        soup.find("main")
        or soup.find("article")
        or soup.find(class_=re.compile(r"article|story|content|body", re.I))
        or soup.body
    )
    raw_text = main.get_text(separator="\n", strip=True) if main else soup.get_text(separator="\n", strip=True)

    # Byline / Autorenname
    author_name = ""
    author_url  = ""
    author_tag  = soup.find("a", href=re.compile(r"/autor[en]?/|/author/|/redakteur/", re.I))
    if author_tag:
        author_name = author_tag.get_text(strip=True)
        author_url  = author_tag.get("href", "")

    # Titel + Meta
    title_tag = soup.find("title")
    title     = title_tag.get_text(strip=True) if title_tag else ""
    h1_tag    = soup.find("h1")
    h1        = h1_tag.get_text(strip=True) if h1_tag else ""
    meta_desc_tag = soup.find("meta", attrs={"name": "description"}) or soup.find("meta", property="og:description")
    meta_desc = meta_desc_tag.get("content", "") if meta_desc_tag else ""

    # JSON-LD Schema
    schema_items = []
    for script in soup.find_all("script", type="application/ld+json"):
        try:
            data = json.loads(script.string or "")
            if isinstance(data, list):
                schema_items.extend(data)
            elif isinstance(data, dict):
                schema_items.append(data)
        except Exception:
            pass

    # Externe Links im Artikel — Quellenindikator
    ext_links = []
    if main:
        for a in main.find_all("a", href=True):
            href = a.get("href", "")
            if href.startswith("http") and not _is_same_domain(href, url):
                ext_links.append({"href": href, "text": a.get_text(strip=True)})

    return {
        "url":         url,
        "title":       title,
        "h1":          h1,
        "meta_desc":   meta_desc,
        "text":        raw_text[:15000],   # Limit für LLM-Kontext
        "author_name": author_name,
        "author_url":  author_url,
        "schema":      schema_items,
        "ext_links":   ext_links,
    }

def _is_same_domain(href: str, base_url: str) -> bool:
    from urllib.parse import urlparse
    return urlparse(href).netloc == urlparse(base_url).netloc
```

### Variante B — Text direkt

Wenn der User Text einfügt statt einer URL, ohne Crawl weiterarbeiten:

```python
def wrap_text_input(text: str, author_name: str = "", title: str = "") -> dict:
    return {
        "url":         None,
        "title":       title,
        "h1":          title,
        "meta_desc":   "",
        "text":        text[:15000],
        "author_name": author_name,
        "author_url":  "",
        "schema":      [],
        "ext_links":   [],
    }
```

---

## Schritt 2: Strukturelle Signale

Automatische Prüfungen — unabhängig vom LLM, direkt aus HTML und Schema.

```python
YMYL_PATTERNS = re.compile(
    r"\b(gesundheit|krankheit|diagnose|symptom|medikament|therapie|arzt|"
    r"finanzen|kredit|steuer|rente|invest|aktie|fonds|versicherung|"
    r"recht|gesetz|urteil|anwalt|klage|"
    r"sicherheit|katastrophe|notfall|unfall|crisis)\b",
    re.I
)

def extract_structural_signals(data: dict) -> dict:
    schema   = data.get("schema", [])
    text     = data.get("text", "")
    signals  = {}

    # --- Schema.org: NewsArticle / Article vorhanden? ---
    article_schema = next(
        (s for s in schema if isinstance(s, dict)
         and s.get("@type") in ("NewsArticle", "Article", "ReportageNewsArticle")),
        None
    )
    signals["has_article_schema"] = bool(article_schema)

    # --- Schema-Autor mit URL? ---
    if article_schema:
        author = article_schema.get("author", {})
        if isinstance(author, list):
            author = author[0] if author else {}
        signals["schema_author_name"] = (author.get("name", "") if isinstance(author, dict) else str(author))
        signals["schema_author_url"]  = (author.get("url", "")  if isinstance(author, dict) else "")
        signals["date_published"]     = article_schema.get("datePublished", "")
        signals["date_modified"]      = article_schema.get("dateModified", "")
        signals["has_publisher"]      = bool(article_schema.get("publisher"))
        # Bildinformation für Discover / Glaubwürdigkeit
        img = article_schema.get("image", {})
        if isinstance(img, list): img = img[0] if img else {}
        signals["image_width"] = (img.get("width", 0) if isinstance(img, dict) else 0)
    else:
        signals.update({
            "schema_author_name": data.get("author_name", ""),
            "schema_author_url":  data.get("author_url", ""),
            "date_published": "", "date_modified": "",
            "has_publisher": False, "image_width": 0,
        })

    # --- Autorenbyline sichtbar? ---
    signals["has_author_byline"]   = bool(data.get("author_name") or signals.get("schema_author_name"))
    signals["has_author_page_link"] = bool(data.get("author_url") or signals.get("schema_author_url"))

    # --- Externe Links (Quellen im Text) ---
    ext_links = data.get("ext_links", [])
    signals["external_link_count"] = len(ext_links)
    signals["ext_link_sample"]     = ext_links[:5]

    # --- YMYL-Thema erkannt? ---
    ymyl_matches = YMYL_PATTERNS.findall(text[:3000])
    signals["is_ymyl"]         = len(set(ymyl_matches)) >= 2
    signals["ymyl_terms_found"] = list(set(ymyl_matches))[:5]

    # --- Wortanzahl (Proxy für Tiefe) ---
    signals["word_count"] = len(text.split())

    # --- Zitate / Direkte Rede (Expertise-Indikator) ---
    quote_count = text.count('"') + text.count("„") + text.count("»")
    signals["has_quotes"] = quote_count >= 4

    return signals
```

---

## Schritt 3: LLM-Analyse der 8 Dimensionen

Jede Dimension wird als eigener Aufruf bewertet — strukturierter Output per JSON.

```python
def build_eeat_prompt(data: dict, signals: dict) -> str:
    author_info = f"Autor: {signals.get('schema_author_name') or data.get('author_name') or '(nicht erkennbar)'}"
    if signals.get("has_author_page_link"):
        author_info += " (Autorseite verlinkt ✓)"
    else:
        author_info += " (keine Autorseite verlinkt)"

    ymyl_note = ""
    if signals.get("is_ymyl"):
        ymyl_note = f"\n⚠️ YMYL-Thema erkannt ({', '.join(signals.get('ymyl_terms_found', []))}). Strengere E-E-A-T-Maßstäbe anwenden."

    ext_note = ""
    if signals.get("ext_link_sample"):
        ext_note = "\nIm Text gefundene externe Links (mögliche Quellen):\n" + "\n".join(
            f"  - {l['text']}: {l['href']}" for l in signals["ext_link_sample"]
        )

    return f"""Du bist ein erfahrener SEO-Redakteur und E-E-A-T-Gutachter. Bewerte den folgenden Artikel anhand von 8 Dimensionen.

Metadaten:
- Titel: {data.get('title') or data.get('h1', '(kein Titel)')}
- {author_info}
- Schema.org vorhanden: {'ja' if signals.get('has_article_schema') else 'nein'}
- Wörter: {signals.get('word_count', '?')}
- Externe Links im Text: {signals.get('external_link_count', 0)}{ymyl_note}{ext_note}

Artikel-Text (kann bei langen Texten gekürzt sein):
---
{data.get('text', '')}
---

Bewerte jede der 8 Dimensionen einzeln. Antworte NUR als JSON-Objekt in diesem Format:

{{
  "dimensions": [
    {{
      "id": 1,
      "frage": "Was weiß dieser Artikel, was die anderen nicht wissen?",
      "bewertung": "grün|gelb|rot",
      "beleg": "Direkte Textstelle oder Beobachtung die zur Bewertung führt (max. 2 Sätze)",
      "empfehlung": "Konkrete Maßnahme wenn nicht grün, sonst leer"
    }},
    {{
      "id": 2,
      "frage": "Woher wissen wir das?",
      "bewertung": "grün|gelb|rot",
      "beleg": "...",
      "empfehlung": "..."
    }},
    {{
      "id": 3,
      "frage": "Sind die entscheidenden Behauptungen durch Primär-/seriöse Quellen belegt?",
      "bewertung": "grün|gelb|rot",
      "beleg": "...",
      "empfehlung": "..."
    }},
    {{
      "id": 4,
      "frage": "Hat Autor oder Redaktion echte Erfahrung bzw. Expertise zum Thema?",
      "bewertung": "grün|gelb|rot",
      "beleg": "...",
      "empfehlung": "..."
    }},
    {{
      "id": 5,
      "frage": "Wird diese Expertise im Inhalt sichtbar oder nur im Autorenkasten behauptet?",
      "bewertung": "grün|gelb|rot",
      "beleg": "...",
      "empfehlung": "..."
    }},
    {{
      "id": 6,
      "frage": "Gibt es eigene Analyse, Einordnung oder Beispiele statt bloßer Zusammenfassung?",
      "bewertung": "grün|gelb|rot",
      "beleg": "...",
      "empfehlung": "..."
    }},
    {{
      "id": 7,
      "frage": "Kann ein Leser nachvollziehen, wer den Inhalt erstellt und wie recherchiert wurde?",
      "bewertung": "grün|gelb|rot",
      "beleg": "...",
      "empfehlung": "..."
    }},
    {{
      "id": 8,
      "frage": "Würde ein fachkundiger Dritter den Text für korrekt und vertrauenswürdig halten?",
      "bewertung": "grün|gelb|rot",
      "beleg": "...",
      "empfehlung": "..."
    }}
  ],
  "gesamt": "grün|gelb|rot",
  "gesamt_begruendung": "Zusammenfassung in 2-3 Sätzen",
  "staerken": ["Stärke 1", "Stärke 2"],
  "schwaechen": ["Schwäche 1", "Schwäche 2"],
  "prioritaeten": ["Wichtigste Handlungsempfehlung", "Zweitwichtigste"]
}}

Bewertungsmaßstab:
- grün = Dimension klar erfüllt, kaum Verbesserungsbedarf
- gelb = Ansatzweise vorhanden, aber ausbaufähig
- rot = Fehlt oder nicht erkennbar

{f'Bei YMYL-Themen: grün nur wenn echte Expertise oder Primärquellen eindeutig belegbar.' if signals.get('is_ymyl') else ''}
"""
```

### Analyse durch Claude (direkt)

Dieser Skill läuft in Claude Code — Claude ist das LLM. Kein externer Prozess nötig.

Nachdem `build_eeat_prompt(data, signals)` den Kontext zusammengestellt hat, analysiert Claude den Inhalt direkt und gibt das Ergebnis als JSON-Objekt aus. Anschließend wird das JSON in die Report-Funktionen übergeben:

```python
import json, re

def parse_llm_json(text: str) -> dict:
    """Extrahiert das JSON-Objekt aus Claudes Textantwort."""
    match = re.search(r'\{.*\}', text, re.DOTALL)
    if match:
        return json.loads(match.group())
    raise ValueError(f"Kein JSON in Antwort: {text[:200]}")

---

## Schritt 4: Report ausgeben

### Terminal-Report

```python
COLORS = {
    "grün":  "\033[32m●\033[0m",
    "gelb":  "\033[33m●\033[0m",
    "rot":   "\033[31m●\033[0m",
}
LABELS = {"grün": "OK", "gelb": "Ausbaufähig", "rot": "Fehlt"}

def print_report(data: dict, signals: dict, analysis: dict):
    url_label = data.get("url") or "(Text-Input)"
    title     = data.get("title") or data.get("h1") or "(kein Titel)"
    is_ymyl   = signals.get("is_ymyl", False)
    gesamt    = analysis.get("gesamt", "?")

    print(f"\n{'='*65}")
    print(f"  E-E-A-T Check — {title[:55]}")
    print(f"  {url_label[:60]}")
    if is_ymyl:
        print(f"  ⚠️  YMYL-Thema ({', '.join(signals.get('ymyl_terms_found', [])[:3])})")
    print(f"{'='*65}\n")

    print("  Strukturelle Signale:")
    def sig(label, ok):
        mark = "\033[32m✓\033[0m" if ok else "\033[31m✗\033[0m"
        print(f"    {mark}  {label}")

    sig("Schema.org (NewsArticle/Article)", signals.get("has_article_schema"))
    sig("Autorenbyline sichtbar",            signals.get("has_author_byline"))
    sig("Autorseite verlinkt",               signals.get("has_author_page_link"))
    sig("Externe Links / Quellenbelege",     signals.get("external_link_count", 0) > 0)
    sig("Direkte Zitate vorhanden",          signals.get("has_quotes"))
    print(f"    → {signals.get('word_count', '?')} Wörter\n")

    print("  Inhaltliche Bewertung:")
    for dim in analysis.get("dimensions", []):
        b = dim.get("bewertung", "?")
        icon = COLORS.get(b, "?")
        label = LABELS.get(b, b)
        print(f"    {icon}  [{label:<12}]  {dim.get('frage', '')}")
        if dim.get("beleg"):
            print(f"               ↳ {dim['beleg'][:90]}")
        if dim.get("empfehlung"):
            print(f"               💡 {dim['empfehlung'][:90]}")
    
    print(f"\n  Gesamtbewertung: {COLORS.get(gesamt, '?')} {gesamt.upper()}")
    print(f"  {analysis.get('gesamt_begruendung', '')}\n")

    if analysis.get("prioritaeten"):
        print("  Wichtigste Maßnahmen:")
        for i, p in enumerate(analysis["prioritaeten"], 1):
            print(f"    {i}. {p}")
    print()
```

### HTML-Report (optional)

Wenn der User einen HTML-Report will, in `~/Downloads/eeat-check-{slug}-{datum}.html` schreiben:

```python
import os
from datetime import date

def generate_html_report(data: dict, signals: dict, analysis: dict) -> str:
    slug    = re.sub(r'[^a-z0-9]+', '-', (data.get("title") or "artikel")[:40].lower()).strip('-')
    today   = date.today().isoformat()
    out     = os.path.expanduser(f"~/Downloads/eeat-check-{slug}-{today}.html")
    is_ymyl = signals.get("is_ymyl", False)
    gesamt  = analysis.get("gesamt", "?")

    COLOR_MAP  = {"grün": "#16A34A", "gelb": "#D97706", "rot": "#DC2626"}
    CIRCLE_MAP = {"grün": "●", "gelb": "●", "rot": "●"}

    dims_html = ""
    for dim in analysis.get("dimensions", []):
        b     = dim.get("bewertung", "rot")
        color = COLOR_MAP.get(b, "#888")
        dims_html += f"""
        <div class="dim">
          <div class="dim-header">
            <span class="dot" style="color:{color}">●</span>
            <span class="dim-q">{dim.get('frage','')}</span>
          </div>
          {'<p class="beleg">'+dim['beleg']+'</p>' if dim.get('beleg') else ''}
          {'<p class="empf">💡 '+dim['empfehlung']+'</p>' if dim.get('empfehlung') else ''}
        </div>"""

    gesamt_color = COLOR_MAP.get(gesamt, "#888")

    prio_html = ""
    for i, p in enumerate(analysis.get("prioritaeten", []), 1):
        prio_html += f'<li>{p}</li>'

    html = f"""<!DOCTYPE html>
<html lang="de">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>E-E-A-T Check — {data.get('title','')[:60]}</title>
<style>
  :root {{--bg:#F3F2EF;--surface:#fff;--border:#DDD9D2;--text:#1B1A17;--text2:#6A6660;}}
  @media(prefers-color-scheme:dark){{:root{{--bg:#131210;--surface:#1D1C1A;--border:#2C2A27;--text:#E6E3DC;--text2:#A09C95;}}}}
  *{{box-sizing:border-box;margin:0;padding:0}}
  body{{font-family:system-ui,sans-serif;background:var(--bg);color:var(--text);padding:32px}}
  .card{{background:var(--surface);border:1px solid var(--border);border-radius:10px;padding:24px;margin-bottom:16px}}
  h1{{font-size:18px;font-weight:700;margin-bottom:4px}}
  .meta{{font-size:12px;color:var(--text2);margin-bottom:16px}}
  .ymyl{{background:#FEF9C3;border:1px solid #FDE047;border-radius:6px;padding:8px 12px;font-size:13px;margin-bottom:12px}}
  .gesamt{{font-size:22px;font-weight:700;color:{gesamt_color}}}
  .sig-grid{{display:grid;grid-template-columns:1fr 1fr;gap:6px;margin:12px 0}}
  .sig{{font-size:13px;display:flex;gap:8px;align-items:center}}
  .dim{{border-bottom:1px solid var(--border);padding:14px 0}}
  .dim:last-child{{border-bottom:none}}
  .dim-header{{display:flex;gap:10px;align-items:flex-start;margin-bottom:4px}}
  .dot{{font-size:18px;flex-shrink:0;line-height:1.4}}
  .dim-q{{font-size:14px;font-weight:500}}
  .beleg{{font-size:13px;color:var(--text2);margin:4px 0 0 28px}}
  .empf{{font-size:13px;color:#2563EB;margin:4px 0 0 28px}}
  ol{{padding-left:20px;font-size:14px;line-height:1.8}}
  h2{{font-size:13px;font-weight:700;letter-spacing:.06em;text-transform:uppercase;color:var(--text2);margin-bottom:12px}}
</style>
</head>
<body>
<div class="card">
  <h1>{data.get('title') or data.get('h1','(kein Titel)')}</h1>
  <div class="meta">{data.get('url') or 'Text-Input'} · {today} · {signals.get('word_count','?')} Wörter</div>
  {'<div class="ymyl">⚠️ YMYL-Thema erkannt: '+', '.join(signals.get("ymyl_terms_found",[])[:5])+'</div>' if is_ymyl else ''}
  <div class="gesamt">Gesamtbewertung: {gesamt.upper()}</div>
  <p style="margin-top:8px;font-size:14px">{analysis.get('gesamt_begruendung','')}</p>
</div>

<div class="card">
  <h2>Strukturelle Signale</h2>
  <div class="sig-grid">
    <div class="sig"><span>{'✅' if signals.get('has_article_schema') else '❌'}</span> Schema.org (NewsArticle)</div>
    <div class="sig"><span>{'✅' if signals.get('has_author_byline') else '❌'}</span> Autorenbyline sichtbar</div>
    <div class="sig"><span>{'✅' if signals.get('has_author_page_link') else '❌'}</span> Autorseite verlinkt</div>
    <div class="sig"><span>{'✅' if signals.get('external_link_count',0) > 0 else '❌'}</span> Externe Links ({signals.get('external_link_count',0)})</div>
    <div class="sig"><span>{'✅' if signals.get('has_quotes') else '❌'}</span> Direkte Zitate vorhanden</div>
  </div>
</div>

<div class="card">
  <h2>Inhaltliche Bewertung</h2>
  {dims_html}
</div>

<div class="card">
  <h2>Wichtigste Maßnahmen</h2>
  <ol>{prio_html}</ol>
</div>
</body>
</html>"""

    with open(out, "w", encoding="utf-8") as f:
        f.write(html)
    return out
```

---

## Vollständiger Run (Einstiegspunkt)

```python
def run_eeat_check(input_value: str, author_name: str = "", want_html: bool = False):
    """
    input_value: URL oder Text
    author_name: optional, hilft wenn Autorenbyline nicht gescrapt werden kann
    want_html:   True → zusätzlich HTML-Report in ~/Downloads/
    """
    print("  Extraktion läuft…")

    # Input-Typ erkennen
    if input_value.startswith("http"):
        data = fetch_article(input_value)
    else:
        data = wrap_text_input(input_value, author_name=author_name)

    if author_name and not data.get("author_name"):
        data["author_name"] = author_name

    # Strukturelle Signale
    signals = extract_structural_signals(data)

    # LLM-Analyse: Prompt ausgeben, Claude antwortet direkt
    prompt = build_eeat_prompt(data, signals)
    # → Prompt an Claude übergeben; Claude gibt JSON aus
    # → JSON mit parse_llm_json() einlesen
    analysis = parse_llm_json(claude_response)  # claude_response = Claudes Antwort auf den Prompt

    # Report
    print_report(data, signals, analysis)

    if want_html:
        path = generate_html_report(data, signals, analysis)
        print(f"  HTML-Report: {path}")

    return {"data": data, "signals": signals, "analysis": analysis}
```

---

## Interaktiver Ablauf (Claude Code Skill)

Wenn dieser Skill in Claude Code ausgeführt wird, übernimmt Claude alle Schritte:

1. **Input erfragen** (falls nicht im Prompt enthalten):
   - URL oder Artikeltext
   - Autor bekannt? (optional — hilft bei reinem Text-Input)
   - HTML-Report gewünscht? (`~/Downloads/`)

2. **Extraktion per Python-Skript** (Bash-Tool):
   - URL → `fetch_article()` → `data`
   - Text → `wrap_text_input()` → `data`

3. **Strukturelle Signale per Python-Skript** (Bash-Tool):
   - `extract_structural_signals(data)` → `signals`

4. **Claude analysiert direkt:**
   - `build_eeat_prompt(data, signals)` → Kontext-String
   - Claude bewertet alle 8 Dimensionen anhand des Textes und gibt JSON aus
   - `parse_llm_json(antwort)` → `analysis`

5. **Report ausgeben:**
   - Terminal: `print_report(data, signals, analysis)`
   - Optional HTML: `generate_html_report(data, signals, analysis)` → `~/Downloads/`

**Bei Fehler (URL nicht erreichbar, Paywall, JS-Rendering):** User bitten, den Artikeltext direkt einzufügen — Schritt 3 und 4 funktionieren auch ohne gecrawlten HTML-Kontext.

---

## Abhängigkeiten

| Paket | Zweck |
|---|---|
| `requests` | URL-Fetch |
| `beautifulsoup4` + `lxml` | HTML-Parsing |
| `json`, `re` | Datenverarbeitung (stdlib) |

Kein API-Key, kein externer Prozess erforderlich — Claude Code ist das LLM-Backend.

---

## Output-Pfad (HTML)

```
~/Downloads/eeat-check-{slug}-{YYYY-MM-DD}.html
```
