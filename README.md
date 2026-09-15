# Partner 360 / Partner Tracker Dashboard — Phase 1 Prototype

> **Current scope:** `app.py` right now is just the **Income Calculator** —
> adjust assumptions, watch the projection update live, click **Submit**
> to lock in a scenario, then download it as PDF or CSV. Everything else
> described below (Business Performance, Client Analytics, Target vs
> Achievement, the 120-RM consolidation pipeline, etc.) still exists as
> working code in the other files — `data_layer.py`, `calculations.py`,
> `excel_loader.py`, `consolidate.py`, `report_export.py` — it's just not
> imported into `app.py` at the moment. Nothing was deleted; say the word
> and any of it can be wired back in.

A Streamlit dashboard for RMs and MFD/Partners to view business performance,
targets, client insights, product performance, gaps/opportunities, and
review action items. Built so the dummy-data layer can be swapped for real
SQL later without touching the UI code.

## What's in this folder

| File               | Purpose |
|--------------------|---------|
| `app.py`           | The Streamlit UI — sidebar filters, navigation, all 10 dashboard sections. |
| `data_layer.py`    | **The only file that knows where data comes from.** Currently generates dummy data. Replace the body of each `load_*` function with a SQL query in Phase 2 — keep the function names, column names, and `@st.cache_data` decorators the same. |
| `calculations.py`  | Achievement %, gap, growth %, run-rate forecast, opportunity detection (SIP/cross-sell/reactivation/dormant/high-value), and the auto-generated insight text. Pure functions, no Streamlit calls — reusable and unit-testable. |
| `formatting.py`     | Indian numbering: ₹ Lakh/Crore formatting, count formatting, achievement-status buckets (On Track / Attention Required / Critical). |
| `income_projection.py` | The Income Calculator's simulation engine — month-by-month client acquisition, SIP step-ups, lumpsum, market compounding, trail income, and optional cross-sell streams, aggregated to yearly milestones. Pure functions, no Streamlit — independently testable. |
| `report_export.py`  | Builds the multi-sheet Excel and one-page PDF reports used in Download / Reports. |
| `excel_loader.py`  | Reads a daily Excel workbook matching the data model (see "Getting daily data in from Excel" below), validates it, and returns the same 5 DataFrames `data_layer.py` produces. |
| `consolidate.py`   | Merges many per-RM Excel exports from a shared folder into one "Processed Partner Data" set — see "120-RM consolidation pipeline" below. Run as a scheduled job, not inside the app. |
| `partner360_data_template.xlsx` | The workbook your ops team fills in and uploads each day. |
| `requirements.txt` | Python dependencies. |
| `.streamlit/config.toml` | Theme (navy/gold, financial-services look). |

## Run locally

```bash
pip install -r requirements.txt
streamlit run app.py
```

## Deploy for free (Phase 1 hosting)

1. Push this folder to a **public or private GitHub repo**.
2. Go to https://share.streamlit.io, sign in with GitHub.
3. "New app" → pick the repo, branch, and `app.py` as the entry point.
4. Deploy. You get a permanent URL like `https://partner-360.streamlit.app`.
5. Share that one link — anyone who opens it sees the same app; the sidebar
   selectors are how each user narrows to "their" view for now (see the
   Security note below).

No paid domain or paid hosting is required for this stage.

## ⚠️ Before connecting real business data

The dummy data in `data_layer.py` is randomly generated — it is not real.
**Do not point this prototype at production data on a public Streamlit
Community Cloud URL until:**
1. Row-level access control is implemented (see Phase 5 below), and
2. Your company has approved exposing that data via this hosting path.

Right now the "Role" and "Partner" selectors in the sidebar *simulate*
access scoping for demo purposes only — they do not enforce it. Anyone
with the link can currently switch to any partner. Treat Phase 1 as an
internal, dummy-data demo only.

## Dashboard sections

1–7 are the analytics views from the original spec (Overview, Business
Performance, Client Analytics, Product Performance, Target vs Achievement,
Growth & Trends, Opportunity/Gap Analysis). 8–14 are:

8. **Partner Review** — log a new review; shows only currently *open*
   items (full history moved to its own section below).
9. **Review History** — the complete chronological review log for the
   selected scope, filterable by status and date range.
10. **Income Calculator** — a long-horizon (multi-year) income and AUM
    growth simulator, modeled on a client-acquisition pace + SIP
    step-ups + lumpsum + market compounding + trail income, with
    optional cross-sell income streams (Life/Health insurance, PMS,
    Demat) you can toggle on. This is a **planning/what-if tool driven
    entirely by the assumptions you enter** — it does not read actual
    transaction data, and the cross-sell commission rates are
    illustrative placeholders. Get your real payout structure from
    Finance before using this externally (e.g. in recruitment pitches).
11. **Target Projection** — a shorter-horizon what-if: enter an assumed
    daily run-rate for the rest of the current period and see the
    projected month-end achievement % and gap per metric. This is
    distinct from the Income Calculator — it projects against *this
    period's actual target*, using the days elapsed/remaining in the
    period you've selected in the sidebar.
12. **Final Target Submission** — a simple form to save proposed target
    values for the selected partner/period. Submissions are **not
    locked** and can be resubmitted anytime; they currently persist only
    for the browser session (see "Persisting target submissions" below)
    with a CSV download as a bridge until that's wired up.
13. **Action Tracker** — open/in-progress/completed/overdue action items.
14. **Download / Reports** — a combined multi-sheet **Excel** workbook and
    a one-page **PDF** summary (KPIs, target vs actual, insights) for the
    current filtered view, plus individual CSVs per table.

### Persisting target submissions

Section 12's submissions live in `st.session_state`, which resets when
the browser tab closes — fine for trying it out, not fine for real use.
To persist them, the lowest-effort path is writing submissions to a
Google Sheet (via `gspread` + a service account) or a small SQLite/
Postgres table; ask me to wire either one up once you've picked.

## 120-RM consolidation pipeline

For the "120 RMs → Central Data Folder → Processed Partner Data" setup:

1. Each RM exports/downloads their own Excel file (their Partner Master,
   Client Master and Transaction Fact rows) into one shared **Central
   Data Folder**. A separate, single **central file** in the same folder
   holds Partner Target and Partner Review (these are typically
   maintained by RM/Cluster Managers, not each RM individually) — every
   file in the folder is read the same way, so this central file just
   needs the two extra sheets filled in and the others left out.
2. Run the consolidation script — on a schedule (cron / Windows Task
   Scheduler), **not** inside the Streamlit app itself, since re-parsing
   ~120 files on every page load would be slow:
   ```bash
   python consolidate.py --raw-folder ./raw_data --out-folder ./processed_data
   ```
   This reads every `.xlsx` in `raw_data`, merges them into the 5 tables,
   de-duplicates on primary key (latest file wins if a partner/client/
   transaction ID appears in more than one file), and writes the result
   as CSVs plus `consolidation_report.json` into `processed_data`. **One
   bad file never blocks the other 119** — it's logged in the report and
   skipped.
3. In the app, pick **"Load Processed Data (Consolidated)"** as the data
   source and point it at that `processed_data` folder. The sidebar
   shows the last consolidation run time, row counts, and any skipped
   files.

This mode needs the app hosted somewhere with filesystem access to that
folder (an internal server), not Streamlit Community Cloud. Community
Cloud is still the right choice for Phase 1 demoing with "Upload Daily
Excel"; move to this pipeline once you're on internal infrastructure.

## Getting daily data in from Excel

You now have three data-source modes, selectable in the sidebar:

1. **Sample Data (demo)** — the dummy data used for the walkthrough.
2. **Upload Daily Excel** — works everywhere, including the free
   `*.streamlit.app` hosting, with zero IT setup. Someone (you, an ops
   person, whoever owns the daily export) fills in
   **`partner360_data_template.xlsx`** and uploads it in the sidebar each
   day. The dashboard re-reads it, validates it, and refreshes instantly.
   This is the right starting point regardless of where the raw numbers
   ultimately come from (a core-system export, an emailed report, a
   manually compiled sheet) — someone just needs to get that data into
   the template's shape once a day.
3. **Auto-load from folder** — for later, once you're hosting the app
   somewhere with access to a shared/network folder (not Community
   Cloud). Point it at a folder, and it picks up the newest file matching
   `partner360_data_*.xlsx` automatically — no one has to open the
   dashboard to upload anything.

### The template

`partner360_data_template.xlsx` has one sheet per table (Partner Master,
Client Master, Transaction Fact, Partner Target, Partner Review) plus an
**Instructions** tab. Column headers must match exactly; a yellow example
row on each sheet shows the expected format and should be deleted before
entering real rows. You never fill in "actual/achieved" values, an
Active/Inactive client status, or AUM contribution by hand — those are
always calculated by `excel_loader.py` from Transaction Fact, consistent
with the "never manually enter what can be calculated" principle from the
spec. In practice, **Transaction Fact is the sheet that changes daily**;
Partner Master and Client Master change rarely; Partner Target changes
monthly; Partner Review changes as reviews happen.

### Path to full automation (no daily human upload)

Once you know exactly where the raw data lives day to day, the manual
upload step can be removed:

- **If it comes from a core system/database**: skip Excel entirely and
  move straight to Phase 2 (SQL in `data_layer.py`) — that's a bigger
  step but the most robust long-term answer.
- **If it's an email attachment**: a Power Automate flow (or an Outlook
  rule + scheduled script) can save the daily attachment to a fixed
  filename in a folder, and "Auto-load from folder" mode picks it up —
  works if the app is hosted somewhere with access to that folder (an
  internal server, not Community Cloud).
- **If it's a manual export you want to keep on free hosting**: a small
  scheduled script (Task Scheduler / cron, or a GitHub Action) can commit
  the day's file into the GitHub repo under a fixed name each morning;
  point the app at reading that committed file directly instead of
  requiring an upload click. Ask me for this script once you know the
  export's source and I'll wire it up.

### Validation

`excel_loader.py` checks that every required sheet and column is present,
coerces dates/numbers, flags unparseable transaction dates (dropping just
those rows rather than failing the whole file), and warns on orphaned
`partner_id`/`client_id` references before handing data to the dashboard.
Fatal errors (missing sheet/column) stop the load and tell you exactly
what to fix; everything else loads with a warning banner.



- **Phase 1 — Prototype (this build):** dummy data, partner selector, KPI
  cards, target vs actual, client analytics, product performance, charts,
  auto-generated insights.
- **Phase 2 — Real data:** replace each function in `data_layer.py` with a
  SQL query (e.g. via `sqlalchemy` + `pyodbc`/`psycopg2`/your warehouse's
  driver) returning a DataFrame with the same columns. Nothing in
  `app.py` or `calculations.py` needs to change.
- **Phase 3 — Target management:** replace the in-session "Log a New
  Review" form (Section 8) with a write-back to a real `partner_target`
  and `partner_review` table (a small database, Google Sheet via API, or
  an internal admin tool).
- **Phase 4 — Opportunity engine:** the opportunity functions in
  `calculations.py` (`sip_opportunity`, `cross_sell_opportunity`,
  `reactivation_opportunity`, `dormant_opportunity`,
  `high_value_low_penetration`) already implement this logic on dummy
  data — extend thresholds/rules with the business team once real data is
  connected.
- **Phase 5 — Authentication & row-level access:** replace the sidebar
  "Role" selector with real login (e.g. `streamlit-authenticator`, or an
  SSO-backed reverse proxy) and enforce that a Partner only ever sees rows
  where `partner_id` matches their logged-in identity; an RM only sees
  partners where `rm == logged_in_rm`; etc. This is the point where you
  should also move off the free public URL if the data is sensitive.
- **Phase 6 — Production hosting:** move from Streamlit Community Cloud to
  company-approved infrastructure (internal VM, Azure/AWS App Service,
  internal Kubernetes, etc.) once real data and auth are in place.

## Design notes

- **Filters cascade**: Region → Cluster → RM → Partner, plus Year/Month,
  all live in the sidebar and apply to every section — matching the "select
  once, every section updates" requirement.
- **"All Partners (Roll-up)"** in the Partner dropdown gives an RM or
  Cluster Manager a rolled-up view across every partner in the current
  filter, instead of forcing a single-partner view.
- **Never manually enter actuals.** All "Actual" figures come from
  `transaction_fact` / `client_master` via `data_layer.py`. Only targets and
  review notes are manually entered (Section 8), consistent with the
  spec's core principle.
- Charts use Plotly; KPI cards use `st.metric` inside a light CSS wrapper
  for a "website" look rather than a raw Streamlit default theme.
