---
name: cp-analysis-short
description: >
  Erstellt eine kompakte Multi-Source-SEO- und Performance-Analyse für eine Domain als selbstständige HTML-Datei.
  Verwende diesen Skill immer wenn der Nutzer sagt: «Kurzanalyse», «analysiere die Website», «wie läuft die Domain»,
  «mach mir eine Analyse für X», «zeig mir den Status von X», «was läuft bei X» oder ähnliches — auch wenn keine
  explizite Quellenangabe gemacht wird. Der Skill zieht parallel Daten aus GSC, Sistrix, GA4 (adloop) und
  Screaming Frog und liefert einen strukturierten HTML-Report inkl. Kernerkenntnis und Top-5-Massnahmen.
---

# cp-analysis-short

Erstellt eine kompakte Kurzanalyse für eine Domain. Zieht Daten aus Google Search Console, Sistrix, GA4 (via adloop) und Screaming Frog — parallel, so schnell wie möglich — und befüllt das HTML-Template mit den Ergebnissen.

## Voraussetzungen

Folgende MCPs müssen verbunden sein:
- `mcp__gscServer__*` — Google Search Console
- `mcp__sistrix__*` — Sistrix
- `mcp__adloop__*` — GA4 (und optional Google Ads)
- `mcp__screaming-frog__*` — Screaming Frog SEO Spider

## Schritt 1: Domain klären

Extrahiere die Domain aus dem Prompt des Nutzers. Falls unklar: mit `AskUserQuestion` nachfragen. Normalisiere auf `example.com` ohne Protokoll und Trailing Slash (für Sistrix/SF), und merke dir die vollständige URL `https://example.com/` (für GSC).

Prüfe via `mcp__gscServer__list_properties`, ob die Domain als `sc-domain:example.com` oder als `https://example.com/` verfügbar ist. Bevorzuge die Domain-Property (`sc-domain:`), da sie alle Subdomains abdeckt.

## Schritt 2: Daten parallel fetchen

Starte alle Requests im selben Turn — nicht sequenziell warten.

### GSC (90 Tage)
```
mcp__gscServer__get_performance_overview  → site_url, days=90
mcp__gscServer__get_search_analytics      → dimensions=query, row_limit=20
mcp__gscServer__get_search_analytics      → dimensions=page, row_limit=10
```

### Sistrix (Land = ch, ausser anders gewünscht)
```
mcp__sistrix__domain_visindex_overview    → domain, country
mcp__sistrix__domain_kwcount_seo          → domain, country
mcp__sistrix__domain_ranking_distribution → domain, country
mcp__sistrix__domain_opportunities        → domain, country, limit=10
mcp__sistrix__domain_competitors_seo      → domain, country, limit=8
mcp__sistrix__domain_traffic_estimation   → domain, country
```

### GA4 (via adloop)
```
mcp__adloop__run_ga4_report → date_range_start=28daysAgo, date_range_end=today
                               dimensions=[sessionMedium], metrics=[sessions, totalUsers, newUsers, screenPageViews]
```
Falls GA4 einen Fehler zurückgibt: im Report als «nicht verfügbar» markieren und Fehlertext notieren. Nicht abbrechen.

### Screaming Frog
```
mcp__screaming-frog__list_crawls → prüfen ob ein aktueller Crawl (<7 Tage) vorhanden ist
```
- Wenn ja: `mcp__screaming-frog__export_crawl` mit `export_tabs=Internal:All,Response Codes:All,Page Titles:All,H1:All` → dann `mcp__screaming-frog__read_crawl_data` für die relevanten CSVs lesen.
- Wenn nein: `mcp__screaming-frog__crawl_site` starten (max_urls=500), auf Abschluss warten, dann exportieren.
- Wenn GUI läuft und Export blockiert ist: Hinweis im Report, SF-Daten überspringen.

## Schritt 3: Daten auswerten

Bevor du das Template befüllst, leite aus den Rohdaten folgende Punkte ab:

**KPI-Status** (ok / warn / danger):
- Klicks: >500 = ok, 100–500 = warn, <100 = danger
- CTR: >3% = ok, 1–3% = warn, <1% = danger
- Ø Position: <10 = ok, 10–20 = warn, >20 = danger
- Keywords (Sistrix): >50 = ok, 10–50 = warn, <10 = danger

**Kernerkenntnis**: Identifiziere 3–5 der auffälligsten Muster — sowohl Probleme als auch Chancen. Typische Muster:
- Hohe Impressionen + tiefe CTR = Snippet-Problem
- Brand-Abhängigkeit (>40% Klicks aus Brand-Queries)
- Seiten mit gutem Ranking aber schlechter CTR
- Opportunity-Keywords knapp ausserhalb Top 10
- Technische Auffälligkeiten aus SF (4xx, Redirect-Chains, fehlende Meta-Tags)

**Top 5 Massnahmen**: Priorisiere nach Aufwand/Impact. Hebel-Icons: 🔴 = hoch, 🟠 = mittel, 🟡 = tief/technisch.

## Schritt 4: Template lesen und befüllen

Lese das Template aus `assets/report_template.html` (relativ zum Skill-Verzeichnis).

Befülle alle `{{VARIABLEN}}` mit den echten Daten:
- GSC: Top-Queries-Zeilen nutzen `GSC_Q_*` (z. B. `GSC_Q_QUERY`, `GSC_Q_KLICKS`, `GSC_Q_BAR_PX`), Top-Seiten `GSC_P_*` (z. B. `GSC_P_SEITE`, `GSC_P_KLICKS`, `GSC_P_BAR_PX`) — so kollidieren die Platzhalter nicht zwischen den beiden Tabellen
- Nicht benötigte Zeilen (z.B. bei weniger als 5 Opportunities) einfach entfernen
- Bei GA4/SF nicht verfügbar: Option A (Hinweis-Box) aktiv lassen, Option B auskommentiert lassen
- Bei GA4/SF verfügbar: Option B aktivieren, Option A entfernen
- Bar-Breite für CTR-Visualisierung: `(ctr_prozent / max_ctr_im_set) * 80` px, gerundet

Wähle Bar-Klasse nach CTR:
- ≥ 5% → `ok`
- 1–5% → `warn`
- < 1% → `danger`

## Schritt 5: Report speichern

Speicherpfad: `outputs/{{PROJEKTNAME}}/analysis-short-{{DATUM}}.html`

Projektname = Domain ohne TLD und Punkte, z.B. `codeplay-ch` für `codeplay.ch`.
Datum = `YYYY-MM-DD`.

Gib dem Nutzer einen direkten `computer://`-Link zur Datei. Keine lange Erklärung danach — der Report ist selbsterklärend.

## Wichtige Hinweise

- **Nicht auf GA4 warten**: Wenn GA4 Fehler gibt, sofort weitermachen und im Report dokumentieren.
- **Nicht auf SF warten wenn GUI läuft**: Hinweis genügt, kein Abbruch.
- **Keine Füllsätze im Report**: Nur echte Daten. Fehlende Daten = leere Zelle oder Hinweis-Box, nicht Platzhaltertext.
- **Sprache**: Der Report ist immer auf Deutsch, unabhängig von der Sprache des Nutzers.
- **Land Sistrix**: Standard ist `ch`. Falls der Nutzer eine .de- oder .at-Domain übergibt, automatisch auf `de` bzw. `at` wechseln.
