# Data Schema — Company Backgrounder Workflow

This document describes the data storage architecture for the company backgrounder workflow: what files are created, what columns/fields they contain, which API or MCP source populates each field, and how findings are linked back to the dossier through footnote citations.

---

## File Structure

```
everfi-backgrounder/
├── skill.md                 ← Workflow instructions (how to run)
├── schema.md                ← This file (data architecture)
├── README.md                ← Human-facing documentation
│
├── data/
│   ├── sources.csv          ← Master citation index (all footnotes)
│   ├── corporate.md         ← Corporate structure findings
│   ├── financials.md        ← Financial history findings
│   └── news_context.md      ← Business model, gov footprint, risk, campaigns
│
├── templates/
│   └── dossier_template.html ← Reusable HTML template for new companies
│
└── output/
    └── everfi_dossier.html  ← Final rendered dossier
```

---

## `data/sources.csv` — Master Citation Index

Every factual claim in the dossier must have a row in this file. Footnotes in the HTML (`[fn:N]`) correspond to `footnote_id` in this file.

### Columns

| Column | Type | Description |
|--------|------|-------------|
| `footnote_id` | integer | Unique sequential ID; matches `[fn:N]` in dossier |
| `source_name` | string | Human-readable name of the source |
| `source_type` | enum | `MCP`, `API`, `Web`, `SEC`, `News` |
| `url` | string | Direct URL to source; `https://enigma.com` for MCP sources |
| `retrieval_date` | date (YYYY-MM-DD) | When the data was retrieved |
| `finding_category` | enum | `corporate_structure`, `financials`, `business_model`, `government_connections`, `political_activity`, `risk_flags` |
| `api_key_required` | string | Whether API authentication/subscription is needed |
| `notes` | string | Key facts returned by this source |

### Source Types

- **MCP** — Enigma tools accessed via Claude's MCP integration. Subscription-gated.
- **API** — Direct REST API call (e.g., USASpending.gov, SEC EDGAR full-text search)
- **Web** — WebFetch or WebSearch (press releases, news articles, public pages)
- **SEC** — Direct SEC EDGAR filing
- **News** — Journalism/editorial coverage

---

## `data/corporate.md` — Entity Structure

### Populated by

- **Enigma KYB** (`mcp__enigma__search_kyb`) — primary source for all registration data
- **Enigma Business Search** (`mcp__enigma__search_business`) — NAICS, brand ID, locations
- **Enigma Gov Archive** (`mcp__enigma__search_gov_archive`) — DC/state business licenses

### Fields documented

| Field | Source | Notes |
|-------|--------|-------|
| Legal name(s) | Enigma KYB | All name variants across registrations |
| Entity type | Enigma KYB | Corporation, LLC, etc. |
| Formation date | Enigma KYB / Delaware SOS | Domestic state formation date |
| Delaware file number | Enigma KYB | DE is standard incorporation state for US corps |
| NAICS code + description | Enigma Business Search | Primary industry classification |
| State registrations table | Enigma KYB | State, type (domestic/foreign), status, file #, issue date |
| Registered agent | Enigma KYB | Name and address across states |
| Beneficial owners | Enigma KYB + Enigma Gov Archive | From beneficial ownership filings where disclosed |
| Officer names + titles | Enigma KYB | Across all state filings |
| Address history | Enigma KYB + Gov Archive | Headquarters and mailing addresses over time |
| DC business license status | Enigma Gov Archive | From DC Open Data / Basic Business Licenses dataset |

---

## `data/financials.md` — Financial History

### Populated by

- **WebSearch** — funding rounds, acquisition terms, investor names
- **WebFetch** — press releases, SEC 8-K filings, investor relations pages
- **SEC EDGAR** — acquisition disclosure (if parent company is public)
- **Crunchbase** — funding round details (via WebSearch)

### Fields documented

| Field | Source | Notes |
|-------|--------|-------|
| Founding date | Multiple | Cross-referenced with corporate formation date |
| VC funding rounds | WebSearch / Crunchbase | Date, amount, lead investors |
| Notable investors | WebSearch / EverFi press releases | Named individuals and funds |
| Prior divestitures | WebSearch | Business units sold before main acquisition |
| Acquisition details | WebFetch — Blackbaud PR, SEC 8-K | Price, structure, closing date, stated rationale |
| Post-acquisition revenue | WebFetch — Blackbaud earnings | Segment revenue data where disclosed |
| Impairment charges | WebSearch / SEC filings | Noncash write-downs against acquired assets |
| Divestiture | WebFetch / PR Newswire | Sale price, buyer, closing date |

---

## `data/news_context.md` — Business Model, Gov Footprint, Risk, Campaigns

### Populated by

- **WebSearch** — journalism, policy analysis, criticism
- **Enigma Gov Archive** — DC/NYC campaign finance records
- **Enigma Negative News** — automated risk scanning
- **WebFetch** — company press releases, government partnership pages

### Fields documented

#### Business Model
| Field | Source |
|-------|--------|
| Revenue model description | WebSearch + Enigma Negative News |
| Sponsor examples (banks, corps) | WebFetch — company press releases, bank PRs |
| School/university reach | WebFetch — EverFi press releases |
| Conflict of interest analysis | WebSearch — American Prospect, peer criticism |

#### Government & Regulatory Footprint
| Field | Source |
|-------|--------|
| Federal database appearances | Enigma KYB (NY procurement, TX franchise, OR workers comp, DOL 5500) |
| DC/state business licenses | Enigma Gov Archive |
| Federal contracts/grants | USASpending.gov via WebFetch/WebSearch |
| White House / agency partnerships | WebFetch — company press releases |
| Regulatory context (Title IX, Clery, CRA) | WebSearch — journalism |

#### Risk Flags
| Field | Source |
|-------|--------|
| Overall risk level | Enigma Negative News |
| Specific risk categories | Enigma Negative News + supplemental WebSearch |
| Data privacy exposure | Enigma Negative News + editorial judgment |
| Organizational instability signals | WebSearch + Enigma KYB (revoked registrations) |

#### Campaign Finance
| Field | Source |
|-------|--------|
| Employer-listed individual contributions | Enigma Gov Archive (DC: resource fac816f7; NYC: resource 34e8f0ef) |
| Candidate, committee, amount, year | Enigma Gov Archive row_details |

---

## Three Required Public Records APIs

This workflow uses three distinct public records APIs. Each covers different record types; together they triangulate corporate structure, federal policy footprint, and government R&D funding status.

| API | Primary use | Auth | Free tier |
|-----|-------------|------|-----------|
| **Enigma** | US business licenses, campaign finance, state procurement, KYB | MCP / subscription | No |
| **GovInfo** | Congressional hearings, Federal Register, GAO reports, Federal Reserve publications, public laws | API key (free) — store in `.env` | Yes — free with registration at api.govinfo.gov |
| **SBIR** | Government R&D award history — confirms whether company received public small-business grants | None — public endpoint | Yes — fully public |

---

## API Sources — Technical Details

### Enigma Gov Archive (`mcp__enigma__search_gov_archive`)

- **Two-pass pattern:** First call with `include_row_details=false` to identify relevant `resource_id`s. Second call with specific `resource_ids` and `include_row_details=true` to retrieve full record data.
- **Key resource IDs used for EverFi:**
  - `5c2035f1-778d-4d97-8680-32160d76a752` — DC Basic Business Licenses
  - `629e11d5-b335-4282-8e38-1d1583e02a89` — DC Open Data Portal Business Datasets
  - `fac816f7-9abc-4ecf-81d6-19cc9ae3f2d0` — DC Campaign Financial Contributions
  - `34e8f0ef-19d6-42ef-8995-d908f51828d1` — NYC Campaign Contributions
- **Covers:** Business registrations, licenses, campaign finance, procurement, professional licenses, court records, health/safety records
- **Access:** Enigma subscription required
- **Authentication:** Via MCP server configured in Claude Code settings

### Enigma KYB (`mcp__enigma__search_kyb`)

- **Requires:** At minimum, business name. State/city narrow results.
- **Returns:** SOS registrations across all states, officer names, registered agent, formation date, beneficial owners (where disclosed), risk summary tasks
- **Access:** Enigma subscription required

### Enigma Business Search (`mcp__enigma__search_business`)

- **Returns:** NAICS codes, brand IDs, operating location count, sample addresses, websites
- **Use for:** Quick industry classification check; finding brand IDs for deeper `get_brand_locations` queries
- **Access:** Enigma subscription required

### Enigma Negative News (`mcp__enigma__search_negative_news`)

- **Returns:** Risk level (high/medium/low/unknown), categorized findings with source URLs
- **Note:** Useful as a first-pass flag but limited by indexable web sources; may miss private litigation or internal compliance issues
- **Access:** Enigma subscription required

### GovInfo API (U.S. Government Publishing Office)

- **Base URL:** `https://api.govinfo.gov/`
- **Authentication:** HTTP header `X-Api-Key: YOUR_KEY` (not query param). Store key in `.env` as `GOVINFO_API_KEY`. Register free at api.govinfo.gov.
- **Key endpoint: Search**
  - Method: **POST** (not GET — GET returns 400)
  - URL: `https://api.govinfo.gov/search`
  - Required headers: `Content-Type: application/json`, `X-Api-Key: $GOVINFO_API_KEY`
  - Body schema:
    ```json
    {
      "query": "COMPANY NAME",
      "pageSize": 20,
      "offsetMark": "*",
      "sorts": [{"field": "score", "sortOrder": "DESC"}]
    }
    ```
  - Returns: `count` (total indexed results), `results[]` array with `packageId`, `title`, `collectionCode`, `dateIssued`, `governmentAuthor`, `download.pdfLink`
- **Key endpoint: Package summary**
  - `GET https://api.govinfo.gov/packages/{packageId}/summary?api_key=$GOVINFO_API_KEY`
  - Returns full metadata for a specific document once you have its `packageId`
- **Collection codes to know:**
  | Code | Content |
  |------|---------|
  | `CHRG` | Congressional hearings (committee testimony) |
  | `BILLS` | Congressional bills and resolutions |
  | `FR` | Federal Register (proposed/final rules, agency notices) |
  | `CREC` | Congressional Record (floor statements) |
  | `GOVPUB` | Government publications (agency reports, Fed Reserve, Treasury, GAO) |
  | `PLAW` | Public laws |
  | `CFR` | Code of Federal Regulations |
- **Important caveat:** GovInfo full-text search tokenizes words. Searching "EverFi" matches documents containing "ever" + "fi" as separate tokens — results will be massively inflated (98,773 for EverFi). Always apply collection filters and verify document relevance before citing.
- **Why use it:** Only source for congressional testimony, Federal Register rulemaking, and GAO/IG reports that may reference a company in a regulatory or oversight context. EverFi relevant collections: CHRG (federal agency compliance training hearings), GOVPUB-FR (Federal Reserve financial literacy publications).

---

### SBIR API (Small Business Innovation Research)

- **Base URL:** `https://api.www.sbir.gov/public/api/` (note: hostname is `api.www.sbir.gov`, not `api.sbir.gov`)
- **Authentication:** None — fully public
- **Key endpoint: Award search**
  - `GET https://api.www.sbir.gov/public/api/awards?firm=[COMPANY_NAME]&rows=25`
  - Returns: JSON array of award records with fields: `firm`, `award_title`, `agency`, `phase`, `program`, `award_amount`, `date_signed`, `abstract`
  - Returns `{"message":"Forbidden"}` if no records match (not an HTTP error — the API's way of saying zero results)
- **What it covers:** SBIR (Small Business Innovation Research) and STTR (Small Business Technology Transfer) grants from all federal agencies — DOD, NSF, NIH, DOE, USDA, DOEd, NASA, HHS, DHS, EPA, and others
- **What a no-result finding means:** SBIR awards go to small businesses doing government-contracted R&D. A company with no SBIR history was funded through private capital, CVC, or commercial revenue — not government R&D grants. This is itself a finding: it distinguishes VC-backed commercial edtech (EverFi) from research-stage govtech startups.
- **Rate limits:** None documented for public endpoint
- **EverFi result:** `{"message":"Forbidden"}` — confirmed zero SBIR/STTR awards. Consistent with EverFi's $251M private VC funding history.

---

### OpenCorporates API

- **Base URL:** `https://api.opencorporates.com/v0.4/`
- **Authentication:** Query parameter `?api_token=YOUR_TOKEN` appended to every request
- **Getting a token:** Register at `https://opencorporates.com/api_accounts/new`
  - **Free tier:** Up to 200 requests/month, 50/day — sufficient for a single company backgrounder
  - **Journalism/NGO access:** Free with application (opencorporates.com/pricing) — unlimited or large-scale
  - **Paid tiers:** £2,250/year (Essentials) through Enterprise
- **Key endpoints for company backgrounding:**

| Endpoint | Purpose |
|----------|---------|
| `GET /v0.4/companies/search?q=[name]&api_token=TOKEN` | Search by company name; returns company_number and jurisdiction_code needed for direct lookup |
| `GET /v0.4/companies/{jurisdiction}/{number}?api_token=TOKEN` | Full company record: name, status, incorporation_date, registered_address, officers array, filings array |
| `GET /v0.4/companies/{j}/{n}/officers?api_token=TOKEN` | Officers with position, name, start/end dates |
| `GET /v0.4/companies/{j}/{n}/filings?api_token=TOKEN` | Filing history with types and dates |
| `GET /v0.4/officers/search?q=[name]&api_token=TOKEN` | Find all companies where a person appears as officer |

- **EverFi jurisdiction codes:** `us_de` (Delaware, #4489432), `us_fl` (Florida, #F18000005152), `us_co`, `us_oh`, `us_ca`, `us_fl`, `us_mo`, `us_pa`, `us_wv` (see corporate.md for all 19)
- **Returned fields:** `company_number`, `name`, `jurisdiction_code`, `company_type`, `current_status`, `incorporation_date`, `dissolution_date`, `registered_address`, `officers[]`, `filings[]`, `opencorporates_url`, `source_url` (links back to primary SOS filing)
- **Why use it over Enigma KYB:** OpenCorporates covers 140+ jurisdictions globally — essential if the company or its acquirer has international subsidiaries. Also provides `source_url` linking directly to the primary SOS record, which Enigma does not always include.

---

### Tracers API

- **Base URL:** Not publicly documented — requires account setup via (877) 723-2689 or tracers.com sales
- **Authentication:** Account-based; no public API key mechanism described in documentation
- **Getting access:** Paid subscription; no free tier; contact tracers.com for pricing
- **Key search type for company backgrounding:** **Combined Business Search**
  - Inputs: business name (omit "LLC" or "Inc" for best results), state; OR FEIN/EIN number
  - For EverFi: search name "EVERFI" or FEIN **26-1818856**
  - Returns three merged record types:
    1. **DBA/FBN Records** — fictitious business names and "doing business as" registrations
    2. **FEIN/Tax ID Records** — IRS-linked entity records; cross-references the tax ID across databases
    3. **Corporate Records** — Secretary of State filings across all 50 states
  - **Comprehensive Business Report** (generated from Combined Business Search results) adds:
    - Employee information
    - Associated businesses at related addresses
    - UCC filings (assets pledged as collateral — reveals debt structure)
    - Domain names registered at company addresses
    - Asset records for the company and its officers/registered agents
    - Court records for the company and its officers
- **Why use it:** Tracers uniquely surfaces UCC filings and FEIN-linked cross-database lookups not available in OpenCorporates or Enigma. A UCC search on EverFi's FEIN (26-1818856) would reveal whether the company pledged recurring revenue contracts as collateral during the Blackbaud period — material to understanding the acquisition structure.
- **Availability documentation:** `https://www.tracers.com/searches-reports/` and `https://help.tracers.com/knowledge/how-to-run-a-combined-business-seach`

---

### USASpending.gov

- **Endpoint:** `https://api.usaspending.gov/api/v2/` (requires POST for most searches)
- **Web interface:** `https://www.usaspending.gov/keyword_search/[term]`
- **Authentication:** None — fully public API
- **Note:** Best accessed via the web UI using WebFetch on the keyword_search URL, or via POST to the award search endpoint

### SEC EDGAR

- **Full-text search:** `https://efts.sec.gov/LATEST/search-index?q=%22company+name%22&dateRange=custom&startdt=2020-01-01`
- **Company filings:** `https://www.sec.gov/cgi-bin/browse-edgar?company=companyname&action=getcompany`
- **Authentication:** None — fully public
- **Best for:** 8-K acquisition disclosures, 10-K annual reports, proxy statements

---

## Footnote Linking Convention

In all markdown data files, claims are annotated with `[fn:N]` references matching the `footnote_id` in `sources.csv`. In the HTML dossier, these render as `<sup>[fn:N]</sup>` elements, and the footnotes panel at the bottom of the dossier provides the full citation for each source.

This ensures every factual claim in the dossier is traceable to a specific source retrieved on a specific date, and that future users of the workflow can reproduce the research by repeating the same API calls.
