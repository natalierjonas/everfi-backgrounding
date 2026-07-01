# Company Backgrounder Skill

Run this skill to generate a structured investigative dossier on a company using public records and MCP data sources. It fans out searches across Enigma (Gov Archive, KYB, Business, Negative News), GovInfo (congressional and federal agency documents), SBIR (government R&D award history), and web sources, then stores findings in structured markdown and CSV files and renders a styled HTML dossier.

## How to Invoke

```
/company-backgrounder [company name] [optional: city] [optional: state]
```

**Example:**
```
/company-backgrounder EverFi Washington DC
```

---

## What the Skill Does

1. **Parallel research phase** — runs all searches simultaneously:
   - Enigma Gov Archive (business licenses, campaign finance, state procurement)
   - Enigma KYB (state-of-state registrations, officer names, beneficial owners, registered agent)
   - Enigma Business Search (NAICS code, brand ID, operating locations, industry classification)
   - Enigma Negative News (risk assessment, legal/regulatory issues, financial distress flags)
   - WebSearch × 3 (company overview + funding, government contracts, controversies/criticism)
   - WebFetch for Blackbaud/SEC or relevant public company disclosures if applicable
   - USASpending.gov keyword search if federal contracts are plausible

2. **Data storage phase** — writes structured files to `data/`:
   - `sources.csv` — footnote-indexed citations for every finding
   - `corporate.md` — entity identity, state registrations, officers, address history
   - `financials.md` — funding rounds, acquisitions, revenue, financial markers
   - `news_context.md` — business model analysis, government connections, risk flags, campaign finance

3. **Dossier generation** — produces `output/[company]_dossier.html` from findings

---

## Core APIs (required — three public records sources)

| API | Type | Access | Cost | What It Returns for a Company |
|-----|------|--------|------|-------------------------------|
| **Enigma** (KYB + Gov Archive + Business + Negative News) | MCP integration | Subscription | Paid | State registrations in all 50 states; officer names; beneficial owners; DC/state business licenses; campaign finance (DC, NYC, SF); state procurement records; TX franchise taxpayer; OR workers' comp; DOL ERISA 5500; risk assessment across legal/financial/labor/data categories |
| **GovInfo** (U.S. Government Publishing Office) | REST API | Free — API key required | Free | Congressional hearings (CHRG), Federal Register (FR), Congressional Record (CREC), GAO reports, public laws, CFR, Federal Reserve publications. Covers full text of federal government documents. Endpoint: `POST https://api.govinfo.gov/search` with header `X-Api-Key: $GOVINFO_API_KEY` |
| **SBIR** (Small Business Innovation Research) | REST API | Public — no auth | Free | Government R&D awards to small businesses from all federal agencies. Confirms whether a company received public R&D funding (vs. purely private capital). Endpoint: `GET https://api.www.sbir.gov/public/api/awards?firm=[NAME]` |

### Supplemental Sources

| Source | Type | Access | Cost | What It Returns |
|--------|------|--------|------|-----------------|
| WebSearch | Built-in | None | Free | Company overview, news coverage, funding history, controversies |
| WebFetch | Built-in | None | Free | SEC filings, press releases, government database pages |
| USASpending.gov | API/Web | None | Free | Federal contracts and grants by recipient |
| SEC EDGAR | Web | None | Free | 10-K, 8-K filings for public companies and disclosed M&A |

---

## Step-by-Step Execution

### Step 1 — Parallel Research (three required APIs + supplemental)

Run all of these in the **same message** so they execute concurrently:

**API 1: Enigma** (four tools — KYB, Gov Archive, Business, Negative News)
```
mcp__enigma__search_kyb(name="[COMPANY]", state="[STATE]")
mcp__enigma__search_gov_archive(query="[COMPANY]", include_row_details=false)
mcp__enigma__search_business(query="[COMPANY]", limit=3)
mcp__enigma__search_negative_news(business_name="[COMPANY]", address="[KNOWN ADDRESS]")
```

**API 2: GovInfo** (free — API key required; store in `.env` as `GOVINFO_API_KEY`)
```bash
# Full-text search across all GPO collections
curl -s -X POST "https://api.govinfo.gov/search" \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: $GOVINFO_API_KEY" \
  -d '{"query":"[COMPANY]","pageSize":20,"offsetMark":"*","sorts":[{"field":"score","sortOrder":"DESC"}]}'

# Target congressional hearings specifically (reduces false positives)
curl -s -X POST "https://api.govinfo.gov/search" \
  -H "Content-Type: application/json" \
  -H "X-Api-Key: $GOVINFO_API_KEY" \
  -d '{"query":"[COMPANY] collection:(CHRG OR BILLS OR FR)","pageSize":15,"offsetMark":"*","sorts":[{"field":"score","sortOrder":"DESC"}]}'

# Fetch a specific document summary once you have a packageId
curl -s "https://api.govinfo.gov/packages/[PACKAGE_ID]/summary?api_key=$GOVINFO_API_KEY"
```
Note: GovInfo tokenizes compound words, so "EverFi" matches documents with "ever" and "fi" separately — inflate results are expected. Filter by collection and verify document relevance before citing.

**API 3: SBIR** (public — no authentication required)
```bash
# Search for SBIR/STTR awards by firm name
curl -s "https://api.www.sbir.gov/public/api/awards?firm=[COMPANY]&rows=25"

# No results / Forbidden = company likely not an SBIR awardee (private VC-backed or too large)
# Document as a finding: confirms public R&D funding vs. private capital model
```

**Supplemental web searches** (run in parallel with the above)
```
WebSearch("[COMPANY] history founding acquisition revenue")
WebSearch("[COMPANY] government contracts federal grants")
WebSearch("[COMPANY] criticism controversy lawsuit compliance")
```

### Step 2 — Follow-up Detail Retrieval (if needed)

If Enigma Gov Archive returns useful `resource_ids`, run a second call with `include_row_details=true` and those specific resource IDs to retrieve full record data (especially useful for campaign finance records).

If the company is acquired by a public company (e.g., Blackbaud/BLKB), fetch:
```
WebFetch("[PARENT COMPANY] press release acquisition URL")
WebFetch("SEC EDGAR 8-K filing URL")
```

### Step 3 — USASpending.gov Check

```
WebSearch("[COMPANY] site:usaspending.gov OR site:sam.gov federal contract award")
WebFetch("https://www.usaspending.gov/keyword_search/[company-name]")
```

### Step 4 — Write Data Files

Create or update these files in `data/`:
- `sources.csv` — one row per citation, with columns: footnote_id, source_name, source_type, url, retrieval_date, finding_category, api_key_required, notes
- `corporate.md` — entity identity, registration table, officers, address history
- `financials.md` — funding table, acquisition details, revenue markers
- `news_context.md` — business model, government footprint, risk flags, campaign finance

### Step 5 — Generate HTML Dossier

Using data from all markdown and CSV files, generate `output/[company]_dossier.html` following the template in `templates/dossier_template.html`. The dossier should include:
- Case file header with retrieval date and source count
- Entity overview grid (two-column: corporate identity + financial markers)
- Sections: Background, Corporate Structure, Financial History, Government Footprint, Political Activity, Risk Flags
- Inline Canvas or SVG chart for financial timeline (if funding/acquisition data exists)
- Footnotes panel with all citations linked back to sources.csv

---

## Output Files

After running, the following files are created/updated:

```
everfi-backgrounder/
├── data/
│   ├── sources.csv          ← footnote index for the dossier
│   ├── corporate.md         ← entity structure and registrations
│   ├── financials.md        ← funding, acquisition, revenue history
│   └── news_context.md      ← business model, risk, campaign finance
└── output/
    └── [company]_dossier.html ← rendered dossier
```

---

## Customization Notes

- **If the company is private with no public financials:** Lean more heavily on Enigma KYB and USASpending; supplement with LinkedIn/Crunchbase via WebSearch
- **If the company is a nonprofit:** Add ProPublica Nonprofit Explorer (https://projects.propublica.org/nonprofits/) to the search stack for 990 filings
- **If the company has significant federal contracting:** Try the USASpending.gov API directly: `https://api.usaspending.gov/api/v2/autocomplete/recipient/?search_text=[company]&limit=5`
- **For non-US companies:** Replace Enigma KYB with Companies House (UK) or equivalent national registry WebFetch

---

## API Accessibility Notes

| API/Source | Free? | Auth Required? | Notes |
|------------|-------|----------------|-------|
| Enigma (all tools) | No | Yes — subscription | Enterprise pricing; academic/education discounts may apply through Enigma's partnerships |
| GovInfo | Yes | Yes — free API key | Register at api.govinfo.gov; store key in `.env` as `GOVINFO_API_KEY`; POST endpoint for search |
| SBIR | Yes | No | Public REST API; returns Forbidden if no awards on file — document as finding, not error |
| USASpending.gov | Yes | No | Fully open federal spending API |
| SEC EDGAR | Yes | No | Full text search and filings free at sec.gov |
| ProPublica Nonprofit Explorer | Yes | No | Free API for nonprofit 990 data |
| WebSearch / WebFetch | Built-in | No | Included in Claude Code |
