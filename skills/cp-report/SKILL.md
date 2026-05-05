---
name: cp-report
description: >
  Creates a live, interactive monthly performance report as a self-contained HTML file.
  Use this skill whenever the user asks to create, generate, open, or update a performance
  report, monthly report, marketing report, or client dashboard. Trigger also when the
  user says things like "zeig mir die Kampagnen", "erstell mir einen Report", "wie laufen
  die Ads", "monatlicher Bericht", "Report für den Kunden vorbereiten", or anything
  implying a visual performance summary. Currently covers Google Ads; GA4 and Sistrix
  will be added in future versions.
  The skill auto-detects the account ID and client name from the project's CLAUDE.md.
---

# Google Ads Report Skill

Generates a self-contained HTML report from live Google Ads data. The report is written
to `outputs/cp-report.html` — öffne es im Browser für das Kundenmeeting.

Zwei Ansichten im Report:
- **Intern (Codeplay)**: Vollständige Metriken, Kampagnentabelle, Kostenverteilung, Empfehlungen
- **Kundenansicht**: Vereinfachte Sprache, keine Fachjargon, "Was wir nächsten Monat tun"

## Voraussetzungen

Folgende MCPs müssen verbunden sein:
- `mcp__gscServer__*` — Google Search Console
- `mcp__sistrix__*` — Sistrix
- `mcp__adloop__*` — Google Ads
- `mcp__analytics-mcp__*` — Google Analytics GA4 
- `mcp__screaming-frog__*` — Screaming Frog SEO Spider

## Step 1: Konfiguration aus CLAUDE.md lesen

Read the project's `CLAUDE.md` and extract:
- **Google Ads ID** — look for a line like `Google Ads ID: 5054130439` or `Nutze immer die Google Ads ID: XXXXXXXXXX`
- **Client name** — look for the client/project name in the file or filename context

If no Google Ads ID is found in CLAUDE.md, ask:
"Welche Google Ads ID soll ich für diesen Report verwenden?"

## Step 2: Kampagnendaten abrufen

Call `mcp__adloop__get_campaign_performance` with:
- `customer_id`: the Google Ads ID from Step 1 (digits only)

The tool returns campaign performance for the last 30 days. Parse the response correctly:
- If the response has the shape `{ content: [{ type: "text", text: "..." }] }`, parse the inner JSON string with `JSON.parse(result.content[0].text)`
- Otherwise use the response directly

The parsed object contains a `campaigns` array. Each campaign has:
- `campaign.name`, `campaign.status`, `campaign.advertising_channel_type`
- `metrics.cost`, `metrics.conversions`, `metrics.cpa`, `metrics.clicks`, `metrics.impressions`, `metrics.ctr`

## Step 3: Report-Periode bestimmen

Use today's date to compute:
- `__GENERATED_AT__`: German month name + year, e.g. "Mai 2026"
  - German month names: Januar, Februar, März, April, Mai, Juni, Juli, August, September, Oktober, November, Dezember

## Step 4: HTML-Report generieren

1. Use the **Read tool** to read the template. The path is relative to this SKILL.md file:
   `assets/report_template.html`

2. Take the template string and make these three substitutions (replace ALL occurrences of each placeholder):
   - `__CLIENT_NAME__` → client name (e.g. "Dr. Meyer Immobilien AG")
   - `__GENERATED_AT__` → formatted period (e.g. "Mai 2026")
   - `__REPORT_DATA_JSON__` → the full campaign response serialized as JSON
     (replace the literal string `__REPORT_DATA_JSON__` with the JSON — this lands inside `var REPORT_DATA = ...;` in the script tag)

3. Use the **Bash tool** to create the outputs directory if it does not exist:
   `mkdir -p outputs`

4. Use the **Write tool** to write the final HTML string to:
   `outputs/cp-report.html`
   **Do NOT output the HTML content in the chat.** The file must be written to disk.

## Step 5: Artifact erstellen (Cowork)

Call `mcp__cowork__list_artifacts` to check if an artifact with id `cp-report` already exists.

- If it does NOT exist: call `mcp__cowork__create_artifact` with:
  - `id`: `cp-report`
  - `html_path`: `outputs/cp-report.html`
  - `description`: "Monatlicher Performance Report für [Client] — Google Ads letzte 30 Tage. Zwei Ansichten: Intern (Codeplay) und Kundenansicht."

- If it already exists: call `mcp__cowork__update_artifact` with:
  - `id`: `cp-report`
  - `html_path`: `outputs/cp-report.html`
  - `update_summary`: "Daten aktualisiert — [GENERATED_AT]"

If neither tool is available (e.g. outside Cowork), skip this step silently.

## Step 6: Bestätigung

Tell the user:

"✅ Report für [Client] ist bereit — [GENERATED_AT], letzte 30 Tage.

Zwei Ansichten im Artifact:
- **Intern (Codeplay)**: vollständige Metriken und Empfehlungen
- **Kundenansicht**: vereinfachte Darstellung für das Meeting

HTML-Datei auch gespeichert unter `outputs/cp-report.html`."
