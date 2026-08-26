---
name: themenfeld-gap-analyse
description: Identifiziert planbare Themenfelder mit hohem Suchvolumen, bei denen ein Publisher noch nicht aktiv ist — durch URL-Cluster-Vergleich mit Wettbewerbern (Sistrix) und Planbarkeits-Scoring (Google Trends). Liefert zwei Handlungslisten: Inventar optimieren (Typ B) und neue Cluster aufbauen (Typ C). Use when asked to find content gaps, topic opportunities, or plannable SEO niches for a publisher.
tools: Read, Edit, Write, Bash
---

# Themenfeld-Gap-Analyse

Findet spezifische Content-Nischen (nicht grobe Ressorts), die stark gesucht werden und bei denen der eigene Publisher kaum oder gar nicht sichtbar ist. Fokus auf **planbare, wiederkehrende Muster** — keine Spontan-Events.

---

## 1. Methodischer Rahmen

### Kernfrage
Welche Themenfelder werden regelmäßig und vorhersehbar gesucht, bei denen der Publisher aktuell kaum rankt?

### Zweistufiger Trichter
```
Sistrix (Wettbewerber-URL-Cluster) → Trichter → Google Trends (Planbarkeits-Score)
```

Sistrix liefert die Hypothesen über Themenfelder. Google Trends validiert, ob das Muster planbar ist. Das Trends-Quota (100 req/day bei der offiziellen v1alpha API) macht diesen Trichter zwingend notwendig.

### Zwei Ausgabetypen mit unterschiedlichen Handlungsempfehlungen

| Typ | Befund | Handlungsempfehlung |
|---|---|---|
| **B** | Publisher rankt, aber Position > 30 | Inventar updaten & optimieren |
| **C** | Publisher hat keinen URL-Cluster in diesem Bereich | Neuen Cluster aufbauen |

---

## 2. Stufe 1 — Wettbewerber-URL-Cluster (Sistrix)

### Ziel
Nicht eine flache Keyword-Liste, sondern **Content-Nischen**: Gruppen von Keywords, die alle auf denselben redaktionellen Bereich zeigen.

### Vorgehen

#### 2.1 Keyword-Rankings mit URLs ziehen
Für jeden Wettbewerber (z.B. stern.de, focus.de, t-online.de): Keywords + Ranking-URL abrufen.

```python
from backend.content_radar.sistrix import fetch_keywords_sistrix
import os

COMPETITORS = ["stern.de", "focus.de", "t-online.de"]

competitor_data = {}
for domain in COMPETITORS:
    keywords = fetch_keywords_sistrix(
        domain=domain,
        api_key=os.getenv("SISTRIX_API_KEY")
    )
    # keywords: [{"keyword": str, "url": str, "position": int, "volume": int}, ...]
    competitor_data[domain] = keywords
```

#### 2.2 URLs auf 2–3 Pfadsegmente normieren

```python
from urllib.parse import urlparse
from pathlib import PurePosixPath

def normalize_url_to_cluster(url: str, depth: int = 3) -> str:
    """
    https://www.focus.de/finanzen/ratgeber/rente/grundrente-berechnen_id123.html
    → focus.de/finanzen/ratgeber/rente
    """
    parsed = urlparse(url)
    domain = parsed.netloc.replace("www.", "")
    parts = [p for p in PurePosixPath(parsed.path).parts if p and p != "/"]
    cluster_path = "/".join(parts[:depth])
    return f"{domain}/{cluster_path}"

# Artikel-IDs und Parameter herausfiltern
def is_article_segment(segment: str) -> bool:
    """True wenn das Segment wie eine Artikel-ID aussieht (enthält _id, Zahl, sehr lang)."""
    return (
        "_id" in segment
        or segment.isdigit()
        or (len(segment) > 60)
        or segment.endswith(".html")
    )

def smart_normalize(url: str, max_depth: int = 3) -> str:
    parsed = urlparse(url)
    domain = parsed.netloc.replace("www.", "")
    parts = [p for p in PurePosixPath(parsed.path).parts if p and p != "/"]
    clean_parts = [p for p in parts if not is_article_segment(p)]
    cluster_path = "/".join(clean_parts[:max_depth])
    return f"{domain}/{cluster_path}" if cluster_path else domain
```

#### 2.3 Cluster aggregieren

```python
from collections import defaultdict

clusters = defaultdict(lambda: {"keywords": [], "total_volume": 0, "domains": set()})

for domain, keywords in competitor_data.items():
    for kw in keywords:
        cluster_key = smart_normalize(kw["url"])
        clusters[cluster_key]["keywords"].append(kw["keyword"])
        clusters[cluster_key]["total_volume"] += kw.get("volume", 0)
        clusters[cluster_key]["domains"].add(domain)

# Sortieren nach Gesamtvolumen
sorted_clusters = sorted(
    [{"cluster": k, **v} for k, v in clusters.items()],
    key=lambda x: x["total_volume"],
    reverse=True
)
```

---

## 3. Stufe 2 — Publisher-Lücken klassifizieren (Sistrix)

### Ziel
Jeden Wettbewerber-Cluster als Typ B (schlechtes Ranking) oder Typ C (kein Cluster) klassifizieren.

### Vorgehen

```python
publisher_keywords = fetch_keywords_sistrix(
    domain="bild.de",  # oder anderer Publisher
    api_key=os.getenv("SISTRIX_API_KEY")
)

# Index: keyword → position
publisher_index = {kw["keyword"]: kw["position"] for kw in publisher_keywords}

# Index: cluster_key → hat publisher URLs in diesem Bereich
publisher_clusters = set()
for kw in publisher_keywords:
    publisher_clusters.add(smart_normalize(kw["url"]))

def classify_gap(cluster: dict, publisher_index: dict, publisher_clusters: set) -> str:
    """
    Typ C: Publisher hat keinen eigenen URL-Cluster in diesem Bereich
    Typ B: Publisher hat Cluster, aber rankt für Top-Keywords nicht in Top 30
    Kein Gap: Publisher rankt für ≥1 Top-Keyword in Top 30
    """
    # Prüfe ob Publisher eigenen Cluster hat
    has_publisher_cluster = any(
        pc.split("/")[0] == "bild.de"  # publisher domain
        and "/".join(cluster["cluster"].split("/")[1:]) in pc
        for pc in publisher_clusters
    )

    # Prüfe Ranking für Top-Keywords des Clusters
    top_keywords = sorted(cluster["keywords"], key=lambda k: publisher_index.get(k, 999))[:10]
    best_position = min((publisher_index.get(kw, 999) for kw in top_keywords), default=999)

    if not has_publisher_cluster and best_position > 30:
        return "C"  # Kein Cluster → aufbauen
    elif best_position > 30:
        return "B"  # Cluster vorhanden aber schlecht → optimieren
    else:
        return "active"  # Publisher ist aktiv

for cluster in sorted_clusters:
    cluster["gap_type"] = classify_gap(cluster, publisher_index, publisher_clusters)

gap_clusters = [c for c in sorted_clusters if c["gap_type"] in ("B", "C")]
```

---

## 4. Stufe 3 — Planbarkeits-Score (Google Trends)

### Ziel
Aus den Gap-Clustern nur solche behalten, die **wiederkehrende, planbare Suchmuster** zeigen — keine Einmal-Events.

### Repräsentant-Keyword auswählen

Pro Cluster genau 1 Keyword → 1 Trends-Call.

```python
def pick_representative_keyword(cluster: dict, publisher_index: dict) -> str:
    """
    Wählt das volumenstärkste Keyword, das nicht zu generisch ist
    und nicht wie ein Event-Keyword aussieht.
    """
    candidates = sorted(
        cluster["keywords"],
        key=lambda k: -sum(
            kw.get("volume", 0)
            for kw in competitor_data_flat
            if kw["keyword"] == k
        )
    )
    # Einfache Event-Filter: Keywords mit Jahreszahlen oder Namen bevorzugt überspringen
    for kw in candidates:
        tokens = kw.lower().split()
        has_year = any(t.isdigit() and len(t) == 4 for t in tokens)
        if not has_year:
            return kw
    return candidates[0]  # Fallback
```

### Saisonalitäts- und Planbarkeits-Score

```python
import numpy as np
from backend.google_trends.client import GoogleTrendsClient

async def score_plannability(keyword: str, client: GoogleTrendsClient) -> dict:
    """
    Holt 2 Jahre Zeitreihe und berechnet zwei Scores:
    - seasonality_score: Wiederkehrendes Jahresmuster (0-1)
    - spike_penalty: Strafe für einmalige Ausreißer (0-1, höher = schlechter)
    """
    # 730 Tage = 2 Jahre (innerhalb des 1800-Tage-Limits)
    ts = await client.fetch_time_series(keyword, country="DE", start_days_ago=730, resolution="WEEK")
    values = [point["value"] for point in ts]

    if len(values) < 52:
        return {"keyword": keyword, "seasonality_score": 0, "spike_penalty": 1, "peak_month": None}

    values_np = np.array(values, dtype=float)

    # Saisonalitätsindex: Korrelation Jahr 1 vs Jahr 2
    year1 = values_np[:52]
    year2 = values_np[52:104] if len(values_np) >= 104 else values_np[52:]
    min_len = min(len(year1), len(year2))
    if min_len > 10:
        corr = float(np.corrcoef(year1[:min_len], year2[:min_len])[0, 1])
        seasonality_score = max(0, corr)  # nur positive Korrelation zählt
    else:
        seasonality_score = 0

    # Spike-Penalty: Strafpunkt wenn maximaler Wert > 3× Median (Einmal-Event-Signal)
    median = float(np.median(values_np[values_np > 0])) if np.any(values_np > 0) else 1
    max_val = float(np.max(values_np))
    spike_ratio = max_val / median if median > 0 else 1
    spike_penalty = min(1.0, max(0, (spike_ratio - 3) / 7))  # 0 bei ratio≤3, 1 bei ratio≥10

    # Peak-Monat aus Jahr 1
    peak_week = int(np.argmax(year1))
    peak_month = (peak_week // 4) + 1  # Näherung: Woche → Monat

    return {
        "keyword": keyword,
        "seasonality_score": round(seasonality_score, 2),
        "spike_penalty": round(spike_penalty, 2),
        "planability_score": round(seasonality_score * (1 - spike_penalty), 2),
        "peak_month": peak_month,
    }
```

### Batch-Scoring mit Quota-Management

```python
import asyncio

async def score_top_clusters(gap_clusters: list, max_calls: int = 90) -> list:
    """
    Scored die Top-N Cluster per Google Trends.
    max_calls: unter 100 bleiben (daily quota).
    """
    client = GoogleTrendsClient()

    # Vorfilter: nur Cluster mit Volumen > 5.000 und ≥ 5 Keywords
    candidates = [
        c for c in gap_clusters
        if c["total_volume"] >= 5000 and len(c["keywords"]) >= 5
    ][:max_calls]

    results = []
    for cluster in candidates:
        kw = pick_representative_keyword(cluster, publisher_index)
        score = await score_plannability(kw, client)
        results.append({**cluster, **score})
        await asyncio.sleep(0.7)  # rate limit

    return results
```

---

## 5. Stufe 4 — Output: Zwei priorisierte Listen

### Finale Priorisierung

```python
MONTHS_DE = {
    1: "Januar", 2: "Februar", 3: "März", 4: "April",
    5: "Mai", 6: "Juni", 7: "Juli", 8: "August",
    9: "September", 10: "Oktober", 11: "November", 12: "Dezember"
}

def build_output(scored_clusters: list) -> dict:
    plannable = [c for c in scored_clusters if c["planability_score"] >= 0.3]

    type_b = sorted(
        [c for c in plannable if c["gap_type"] == "B"],
        key=lambda x: -x["total_volume"] * x["planability_score"]
    )
    type_c = sorted(
        [c for c in plannable if c["gap_type"] == "C"],
        key=lambda x: -x["total_volume"] * x["planability_score"]
    )

    return {
        "optimieren": [  # Inventar updaten
            {
                "cluster": c["cluster"],
                "top_keywords": c["keywords"][:5],
                "volume": c["total_volume"],
                "planability": c["planability_score"],
                "peak_month": MONTHS_DE.get(c.get("peak_month"), "–"),
                "action": "Bestehende Artikel aktualisieren, Keyword-Abdeckung verbessern",
            }
            for c in type_b[:25]
        ],
        "aufbauen": [   # Neuen Cluster aufbauen
            {
                "cluster": c["cluster"],
                "top_keywords": c["keywords"][:5],
                "volume": c["total_volume"],
                "planability": c["planability_score"],
                "peak_month": MONTHS_DE.get(c.get("peak_month"), "–"),
                "action": "Neues Themenfeld redaktionell erschließen, Cluster-Architektur planen",
            }
            for c in type_c[:25]
        ],
    }
```

### CSV-Export

```python
import csv, io

def to_csv(output: dict) -> str:
    rows = []
    for item in output["optimieren"]:
        rows.append({
            "typ": "B – Optimieren",
            "cluster": item["cluster"],
            "top_keywords": " | ".join(item["top_keywords"]),
            "volumen": item["volume"],
            "planability_score": item["planability"],
            "peak_monat": item["peak_month"],
            "empfehlung": item["action"],
        })
    for item in output["aufbauen"]:
        rows.append({
            "typ": "C – Aufbauen",
            "cluster": item["cluster"],
            "top_keywords": " | ".join(item["top_keywords"]),
            "volumen": item["volume"],
            "planability_score": item["planability"],
            "peak_monat": item["peak_month"],
            "empfehlung": item["action"],
        })

    buf = io.StringIO()
    writer = csv.DictWriter(buf, fieldnames=list(rows[0].keys()))
    writer.writeheader()
    writer.writerows(rows)
    return buf.getvalue()
```

---

## 6. Konfiguration & Schwellenwerte

| Parameter | Wert | Begründung |
|---|---|---|
| URL-Tiefe | 2–3 Segmente | Nischen-Granularität ohne Artikel-Ebene |
| Gap-Schwellenwert | Position > 30 | De facto unsichtbar in Search |
| Mindestvolumen | 5.000 / Monat | Relevante Nischen filtern |
| Min. Keywords/Cluster | 5 | Strukturiertes Themenfeld, kein Einzel-Keyword |
| Trends-Calls | max. 90/Tag | Sicherheitsabstand zum 100-Limit |
| Planability-Score min. | 0.30 | Erkennbares Saisonmuster vorhanden |
| Spike-Ratio Grenze | 3× Median | Unterhalb = kein Einmal-Event |

---

## 7. Datenquellen & Abhängigkeiten

| Quelle | Was | Client |
|---|---|---|
| Sistrix API | Keywords + Rankings + URLs für Wettbewerber + Publisher | `backend.content_radar.sistrix.fetch_keywords_sistrix` |
| Google Trends v1alpha | Zeitreihe pro Keyword (Planbarkeits-Score) | `backend.google_trends.client.GoogleTrendsClient` |

Beide Clients in `next-seo-as-tools`. Credentials in `.env`:
- `SISTRIX_API_KEY`
- Google Trends OAuth2 via `~/.google-trends/token.json`

---

## 8. Typisches Ergebnis-Format

```
AUFBAUEN (Typ C) — Neue Cluster erschließen
──────────────────────────────────────────
1. focus.de/finanzen/ratgeber/rente         Volumen: 82.000  Score: 0.78  Peak: Oktober
   Keywords: rente berechnen, grundrente, renteneintrittsalter, ...
   → Themenfeld "Rente Ratgeber" hat kein BILD-Äquivalent

2. stern.de/gesundheit/ernaehrung           Volumen: 61.000  Score: 0.65  Peak: Januar
   Keywords: abnehmen schnell, kalorien berechnen, intermittierendes fasten, ...
   → Themenfeld "Ernährung & Abnehmen" nur schwach bei BILD

OPTIMIEREN (Typ B) — Bestehendes verbessern
──────────────────────────────────────────
1. bild.de/ratgeber/finanzen                Volumen: 45.000  Score: 0.71  Peak: März
   Keywords: steuererklärung 2025, steuer frist, steuer tool, ...
   → BILD rankt auf Pos. 35–60, Wettbewerb auf Pos. 3–10
```
