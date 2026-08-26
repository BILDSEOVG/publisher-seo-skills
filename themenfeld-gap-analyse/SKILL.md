---
name: themenfeld-gap-analyse
description: Identifiziert planbare Themenfelder mit hohem Suchvolumen, bei denen ein Publisher noch nicht aktiv ist — durch URL-Cluster-Vergleich mit Wettbewerbern (Sistrix) und Planbarkeits-Scoring (Google Trends). Liefert zwei Handlungslisten: Inventar optimieren (Typ B) und neue Cluster aufbauen (Typ C). Use when asked to find content gaps, topic opportunities, or plannable SEO niches for a publisher.
tools: Read, Edit, Write, Bash
---

# Themenfeld-Gap-Analyse

Findet spezifische Content-Nischen (nicht grobe Ressorts), die stark gesucht werden und bei denen der eigene Publisher kaum oder gar nicht sichtbar ist. Fokus auf **planbare, wiederkehrende Muster** — keine Spontan-Events.

---

## ⚠ Was hier NICHT reinkommt — Ausschlussliste

Diese Kategorien aktiv filtern. Ohne Filter landen sie vorne in der Ergebnisliste und machen das Ergebnis unbrauchbar.

### Cluster-Typen die rausfliegen

| Kategorie | Beispiel-Cluster | Warum raus |
|---|---|---|
| **Online-Spiele** | spiele.stern.de/solitaer, games.focus.de/mahjong | Anderes Produkt, kein redaktioneller Content |
| **Sportwetten / Gambling** | kicker.de/wetten, sport1.de/sportwetten | Regulatorisch, anderes Geschäftsmodell |
| **Online-Apotheke** | apotheken-umschau.de/produkte | E-Commerce, kein Publisher-Thema |
| **Kreditkarten & Bankprodukte** | focus.de/finanzen/kreditkarte, check24.de/kreditkarten | Vergleichsportal-Logik, kein Ratgeber |
| **Kaufberatung / Test / Vergleich** | stern.de/kaufkosmos, focus.de/kaufberatung | Testberichte sind anderes Content-Format |
| **Sub-Domain-Silos die kein redakt. Thema sind** | email.t-online.de, login.stern.de | Technische Bereiche, kein Content |
| **Gattungsnamen ohne Tiefe** | stern.de/panorama, focus.de/wohnen | Ressort-Label, keine spezifische Nische |

### Keywords die vorab rausgefiltert werden (Stufe 0 — vor allem anderen)

Brand-Keywords haben in der Analyse nichts zu suchen. Sie produzieren Cluster wie `t-online.de` mit 11 Mio. Traffic, obwohl kein einziges davon ein redaktionelles Themenfeld ist. Wenn dieser Filter fehlt, dominieren Brand-Cluster die gesamte Top-Liste.

Ein Keyword ist ein Brand-Keyword wenn es:
- den Domain-Namen enthält (stern, focus, t-online, bild, spiegel)
- eine Domain-Schreibweise ist (.de, .com, Punkt+Buchstaben)
- eine Marken-Variation ist (t online, tonline, t-online.de)
- ein Service-Keyword der Marke ist (t-online mail, focus login, stern newsletter)

```python
_BRAND_TOKENS = {
    "stern", "focus", "t-online", "tonline", "bild", "spiegel",
    "zeit", "welt", "sueddeutsche", "faz", "tagesspiegel",
    # Publisher-spezifisch ergänzen
}
_BRAND_IN_KW = (".de", ".com", "login", " mail", " app", " abo", " plus")

def is_brand_keyword(kw: str, competitor_domains: list[str]) -> bool:
    lower = kw.lower()
    # Domain-Schreibvarianten
    for domain in competitor_domains:
        base = domain.replace(".de", "").replace(".", "").replace("-", "")
        if base in lower.replace(" ", "").replace("-", ""):
            return True
    for token in _BRAND_TOKENS:
        if token in lower:
            return True
    for suffix in _BRAND_IN_KW:
        if suffix in lower:
            return True
    return False
```

---

## Was macht einen GUTEN Cluster aus

Ein guter Cluster ist eine spezifische, planbare Nische — kein Ressort-Label.

### Gut ✓ vs. Schlecht ✗

| Schlecht (zu breit) | Gut (spezifisch) | Warum besser |
|---|---|---|
| `focus.de/wohnen` | **Haushaltstipps** (natron fußbad, essig gegen kalk) | Konkreter Intent: Alltagsprobleme lösen |
| `t-online.de/garten` | **Gartentipps Sommer** (bougainvillea überwintern, wann darf man nicht mähen) | Saisonales Muster, planbar |
| `stern.de/lifestyle/leute` | **Promi-Kinder** (pax thien jolie-pitt, zahara jolie-pitt) | Spezifisches Cluster, nicht "alle Promis" |
| `stern.de/lifestyle/leute` | **Tanja Spengler** | Einzelperson als eigenes Cluster wenn Traffic groß genug |
| `focus.de/gesundheit/ratgeber` | **Neue Medikamente** (viagra frauen, pille für den mann, abnehmpille höhle der löwen) | Spezifischer Keyword-Intent: "Was gibt es Neues in der Medizin?" |
| `focus.de/gesundheit/ratgeber` | **Warnsignale & übersehene Symptome** (zuckendes auge warnsignal, schmerzen linker arm) | Eigenständige Nische mit klarem Leserbedürfnis |
| `focus.de/gesundheit/ratgeber` | **Atemwegs-Ratgeber** (keuchhusten trotz impfung, kopfgrippe, nasenspülung wasser bleibt im kopf) | Konkrete Beschwerden, nicht "Gesundheit allgemein" |

**Faustregel:** Wenn man die Keywords laut vorliest und sie klingen wie "Sachen die jemand googelt der ein konkretes Problem hat" → gutes Cluster. Wenn sie klingen wie ein Zeitschriften-Ressort → zu breit.

### Erkennungsmerkmale guter Cluster

- **Keyword-Intent ist einheitlich**: Alle Top-Keywords haben dasselbe Leserbedürfnis
- **Mindestens 3 verschiedene Keywords** die auf dasselbe Thema zeigen
- **Planbar**: Das Thema wird jedes Jahr zu einer ähnlichen Zeit gesucht (oder gleichmäßig ganzjährig)
- **Kein Eigenname-Monopol**: Wenn 9 von 10 Keywords ein einzelner Prominame sind → eher Einzelperson-Cluster statt Themenfeld

---

## Bekannte Schwäche: URL-Pfade sind grobe Proxies

Das URL-Cluster-Verfahren gruppiert Keywords nach der Pfadstruktur der Wettbewerber. Das erzeugt oft zu breite Cluster:

- `focus.de/immobilien/wohnen` enthält *sowohl* Immobilien-Keywords *als auch* Haushalts-Tipps — weil focus.de beides unter `/wohnen` ablegt
- `stern.de/gesundheit` enthält Ratgeber-Tipps, Produktrückrufe UND Medizin-News

**Konsequenz**: Nach der URL-Clusterung die Keyword-Liste des Clusters durchsehen und manuell (oder per LLM-Klassifikation) in Sub-Cluster aufteilen, wenn der Intent zu gemischt ist. Der URL-Cluster-Name ist Ausgangspunkt, nicht Endpunkt.

---

## 1. Methodischer Rahmen

### Kernfrage
Welche Themenfelder werden regelmäßig und vorhersehbar gesucht, bei denen der Publisher aktuell kaum rankt?

### Vierstufiger Trichter
```
Stufe 0: Brand-Keywords filtern (KRITISCH — zuerst)
Stufe 1: Wettbewerber-URL-Cluster bauen (Sistrix)
Stufe 2: Publisher-Lücken klassifizieren (Sistrix)
Stufe 3: Planbarkeits-Score (Google Trends)
```

### Zwei Ausgabetypen

| Typ | Befund | Handlungsempfehlung |
|---|---|---|
| **B** | Publisher rankt, aber Position > 30 | Inventar updaten & optimieren |
| **C** | Publisher hat keinen URL-Cluster in diesem Bereich | Neuen Cluster aufbauen |

---

## 2. Stufe 0 — Brand-Keywords und Blacklist-Cluster ausschließen

**KRITISCHER SCHRITT — muss vor allem anderen passieren.**

Wenn man diesen Schritt überspringt, dominieren Brand-Navigational-Cluster (t-online.de mit 11 Mio. Traffic, focus.de mit 2 Mio. Traffic) die gesamte Ergebnisliste und das Ergebnis ist wertlos.

```python
_BRAND_TOKENS = {
    "stern", "focus", "t-online", "tonline", "bild", "spiegel",
    "zeit", "welt", "sueddeutsche", "faz",
}
_BRAND_IN_KW = (".de", ".com", " mail", " login", " abo", " app", " plus", " online")

def is_brand_keyword(kw: str, competitor_domains: list[str]) -> bool:
    lower = kw.lower()
    for domain in competitor_domains:
        base = domain.replace(".de", "").replace("-", "")
        if base in lower.replace(" ", "").replace("-", ""):
            return True
    for token in _BRAND_TOKENS:
        if token in lower:
            return True
    for suffix in _BRAND_IN_KW:
        if suffix in lower:
            return True
    return False

def filter_brand_keywords(rows: list[dict], competitor_domains: list[str]) -> list[dict]:
    return [r for r in rows if not is_brand_keyword(r.get("kw", ""), competitor_domains)]
```

### URL-Cluster Blacklist (Pfad-basiert)

Nach der Clusterung noch mal filtern — Cluster raus, deren Pfad einer dieser Kategorien entspricht:

```python
_CLUSTER_BLACKLIST_SEGS = {
    # Spiele
    "spiele", "games", "solitaer", "solitaire", "mahjong", "puzzle", "spielen",
    # Gambling / Wetten
    "sportwetten", "wetten", "casino", "gambling", "glücksspiel",
    # E-Commerce / Vergleichsportale
    "kreditkarte", "kreditkarten", "kredit", "apotheke", "online-apotheke",
    "kaufberater", "kaufkosmos", "testberichte", "test-und-vergleich",
    # Technische Sub-Domains ohne Content
    "email", "mail", "login", "account", "mein", "mediathek",
}

def is_blacklisted_cluster(cluster_key: str) -> bool:
    segs = set(s.lower() for s in cluster_key.split("/"))
    return bool(segs & _CLUSTER_BLACKLIST_SEGS)

def filter_blacklisted(clusters: list[dict]) -> list[dict]:
    return [c for c in clusters if not is_blacklisted_cluster(c["cluster"])]
```

### Keyword-Blacklist (kaufen / vergleich / test)

Keywords die auf Transaktions- oder Kaufentscheidungs-Intent hinweisen — kein redaktionelles Themenfeld:

```python
_KW_INTENT_BLACKLIST = {
    "kaufen", "bestellen", "preisvergleich", "vergleich", "test 20", "testsieger",
    "günstig", "angebot", "rabatt", "coupon", "gutschein",
}

def is_transactional_keyword(kw: str) -> bool:
    lower = kw.lower()
    return any(token in lower for token in _KW_INTENT_BLACKLIST)
```

---

## 3. Stufe 1 — Wettbewerber-URL-Cluster bauen (Sistrix)

```python
from backend.content_radar.sistrix import domain_keywords_all
from backend.content_gap.clusters import build_competitor_clusters

COMPETITORS = ["stern.de", "focus.de", "t-online.de"]
KW_PER_DOMAIN = 3000

competitor_data = {}
for domain in COMPETITORS:
    raw = domain_keywords_all(domain, api_key, total=KW_PER_DOMAIN)
    competitor_data[domain] = filter_brand_keywords(raw, COMPETITORS)  # Stufe 0!

clusters = build_competitor_clusters(competitor_data, min_keywords=3, min_traffic=50)
clusters = filter_blacklisted(clusters)
```

### URL-Normierung auf 2–3 Segmente

```python
from urllib.parse import urlparse
from pathlib import PurePosixPath

_ARTICLE_SIGNALS = ("_id", "id_", "_artikel", "_news", "_beitrag")

def is_article_segment(seg: str) -> bool:
    lower = seg.lower()
    return (
        any(sig in lower for sig in _ARTICLE_SIGNALS)
        or lower.endswith((".html", ".htm", ".php"))
        or len(seg) > 60
        or (len(seg) > 15 and sum(c.isdigit() for c in seg) > 4)
    )

def normalize_url(url: str, depth: int = 3) -> str:
    """
    https://www.focus.de/finanzen/ratgeber/rente/grundrente_id123.html
    → focus.de/finanzen/ratgeber/rente
    """
    parsed = urlparse(url)
    domain = parsed.netloc.replace("www.", "")
    parts = [p for p in PurePosixPath(parsed.path).parts if p and p != "/"]
    clean = [p for p in parts if not is_article_segment(p)]
    cluster = "/".join(clean[:depth])
    return f"{domain}/{cluster}" if cluster else domain
```

---

## 4. Stufe 2 — Publisher-Lücken klassifizieren

### Das Grundproblem: Keyword-Matching vs. URL-Semantik

Für News-Publisher wie BILD gilt: Die Top-3.000 Keywords aus Sistrix sind fast ausschließlich Brand-Navigational-Queries (Platz 1 für "bild", "bild.de", "bild zeitung"). Content-Keywords (wo BILD auf Pos. 10–40 rankt) liegen tiefer in der Liste.

**Konsequenz:** Reines Keyword-Matching ergibt 0 Überschneidungen und 100% Typ C — obwohl BILD viele Themenfelder bereits abdeckt.

**Lösung: URL-semantisches Matching** (zusätzlich zum Keyword-Matching):
1. BILD's URL-Cluster aufbauen (welche Pfad-Segmente hat BILD in seiner Top-Liste?)
2. Mit Competitor-Pfad-Segmenten abgleichen
3. Vollständige Überschneidung → aktiv; Teilüberschneidung bei 2+ Pfad-Segmenten → Typ B; keine → Typ C

```python
def classify_gaps_semantic(clusters, publisher_rows, position_threshold=30):
    """
    Pass 1: Exaktes Keyword-Matching → aktiv / Typ B
    Pass 2: URL-Segment-Matching → aktiv / Typ B / Typ C
    """
    # Pass 1
    pub_kw_pos = {}
    for row in publisher_rows:
        kw, pos = row.get("kw"), row.get("position")
        if kw and pos is not None:
            pub_kw_pos[kw] = min(pub_kw_pos.get(kw, 999), int(pos))

    # Pass 2: Publisher URL-Segmente aufbauen
    pub_segs = build_publisher_topic_segs(publisher_rows, min_kws_per_cluster=5)

    result = []
    for cluster in clusters:
        top_kws = cluster["top_keywords"][:10]
        best = min((pub_kw_pos.get(kw, 999) for kw in top_kws), default=999)

        if best <= position_threshold:
            gap_type = "active"
        elif best < 999:
            gap_type = "B"
        else:
            # URL-Semantik
            comp_segs = url_path_segs(cluster["cluster"])
            shared = comp_segs & pub_segs
            depth = len(cluster["cluster"].split("/")) - 1

            if not comp_segs or len(shared) == len(comp_segs):
                gap_type = "active"
            elif shared and depth >= 2:
                gap_type = "B"
            else:
                gap_type = "C"

        result.append({**cluster, "gap_type": gap_type})
    return result
```

---

## 5. Stufe 3 — Planbarkeits-Score (Google Trends)

### Wichtig: Token-Management

Google Trends v1alpha benötigt OAuth2. Das Token läuft nach einigen Wochen ab (`invalid_grant`). Refresh-Skript:

```bash
bash .claude/skills/api-access/get-google-trends-token.sh
```

Danach das Skript erneut ausführen — Sistrix-Daten kommen aus dem 7-Tage-Cache (0s), nur Trends-Calls werden neu gemacht.

### Scoring

```python
async def score_planability(points: list) -> dict:
    values = [p.get("scaledSearchInterest") or 0 for p in points]
    if len(values) < 20:
        return {"planability": None, "seasonality": None, "peak_month": None}

    half = len(values) // 2
    y1, y2 = values[:half], values[half:]
    # Pearson-Korrelation Jahr 1 vs Jahr 2 = Saisonalitäts-Score
    corr = pearson(y1[:min(len(y1), len(y2))], y2[:min(len(y1), len(y2))])
    seasonality = max(0.0, corr)

    # Spike-Penalty: wenn max > 3× Median → vermutlich Einmal-Event
    non_zero = [v for v in values if v > 0]
    median = sorted(non_zero)[len(non_zero) // 2] if non_zero else 1
    spike_ratio = max(values) / median if median > 0 else 1
    spike_penalty = min(1.0, max(0.0, (spike_ratio - 3) / 7))

    planability = round(seasonality * (1 - spike_penalty), 2)
    return {"planability": planability, "peak_month": _calc_peak_month(values)}
```

### Vertreter-Keyword auswählen (Jahreszahlen und Event-Tokens vermeiden)

```python
_EVENT_TOKENS = {"heute", "aktuell", "jetzt", "live", "2023", "2024", "2025", "2026", "2027"}

def pick_keyword(cluster: dict) -> str:
    for kw in cluster["top_keywords"]:
        tokens = set(kw.lower().split())
        has_year = any(t.isdigit() and len(t) == 4 for t in tokens)
        if not has_year and not (tokens & _EVENT_TOKENS):
            return kw
    return cluster["top_keywords"][0]
```

---

## 6. Konfiguration & Schwellenwerte

| Parameter | Wert | Begründung |
|---|---|---|
| URL-Tiefe | 2–3 Segmente | Nischen-Granularität ohne Artikel-Ebene |
| Gap-Schwellenwert | Position > 30 | De facto unsichtbar in Search |
| Min. Keywords/Cluster | 3 | Für erste Analyse; für finale Empfehlungen eher 5+ |
| Min. Traffic/Cluster | 50 | Rauschen entfernen |
| Min. Publisher-URLs/Segment | 5 | URL-Segment gilt als "signifikant" für publisher |
| Trends-Calls | max. 85–90/Tag | Sicherheitsabstand zum 100-Limit |
| Planability-Score min. | 0.30 | Erkennbares Saisonmuster vorhanden |
| Spike-Ratio Grenze | 3× Median | Unterhalb = kein Einmal-Event |

---

## 7. Typisches Ergebnis nach allen Filtern

Erwartbare Größenordnung bei 3 Wettbewerbern × 3.000 Keywords:

```
Cluster gesamt: ~370
  davon aktiv (Publisher ist aktiv): ~20
  davon Typ B (Oberthema vorhanden, Tiefe fehlt): ~60
  davon Typ C (echter Gap): ~280

Nach Blacklist-Filter (Spiele, Wetten, Apotheke, Kaufberatung):
  Typ C reduziert auf: ~220–250
```

Beispiel-Ergebnis für BILD vs. stern/focus/t-online:

```
TOP TYP C (neue Cluster aufbauen):
  Ernährung-Ratgeber      (burrata schwangerschaft, kalorien bier)   150k Traffic
  Rezepte                 (brot im airfryer, sandwichmaker rezepte)  100k Traffic
  Wissen/Begriffe-Nische  (gilf bedeutung, baddie, kreuzritter)       67k Traffic
  Heim/Garten/Tiere       (wespennest entfernen, wie alt werden tauben) 43k

TOP TYP B (Tiefe fehlt):
  Gesundheit Ratgeber-Tiefe  (warnsignale, neue medikamente)         193k Traffic
  Finanzen Steuern Ratgeber  (grundsteuer 1000 qm, steuerklasse 2)   14k Traffic
```

---

## 8. Datenquellen & Abhängigkeiten

| Quelle | Was | Client |
|---|---|---|
| Sistrix API | Keywords + Rankings + URLs | `backend.content_radar.sistrix.domain_keywords_all` |
| Google Trends v1alpha | Zeitreihe pro Keyword | `backend.google_trends.client.GoogleTrendsClient` |

Beide in `next-seo-as-tools`. Credentials in `.env`:
- `SISTRIX_API_KEY`
- Google Trends OAuth2 über `~/.google-trends/credentials.json`

Standalone-Skript: `scripts/content_gap_report.py`
