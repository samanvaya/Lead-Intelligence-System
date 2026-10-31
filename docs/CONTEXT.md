# Context

> **Sources:** `Mini_Project_Brief.docx` (the brief), `data/DATA_README.md`, `data/EXPECTED_OUTCOMES_NOTE.md`, and profiling of the three CSVs in `data/`.
> **Labels:** **[Brief]** = stated in the brief or the data docs. **[Observed]** = found by profiling the data. **[Assumption]** = not stated anywhere; my working assumption, to confirm with the client.

## Problem statement

- **Who:** the sales team at a **mid-market SaaS company** **[Brief]**. The brief doesn't name the product or the ideal customer profile (ICP) **[Observed]**.
- **Circumstance:** they receive about **1,200 inbound leads a month**, but only about **60** get qualified and contacted, which is a 95% loss rate. Each qualification decision takes a rep **8–12 minutes** of manual work **[Brief]**.
- **Progress wanted:** a system that **auto-qualifies leads, sequences outreach, and logs insights** the team can act on immediately. It should answer two questions: which leads should we pursue, and in what order? It should also draft a personalized first message. The company suspects it is **missing good opportunities and wasting effort on poor fits** **[Brief]**.
- **Current workaround:** reps judge each lead by hand, case by case. There's no documented rubric, scoring, or prioritization **[Brief]**. At 1,200 leads × ~10 min, that's about **200 rep-hours a month** **[Observed, derived]**, and only 5% of leads end up being worked.

## Success metric

| Metric | Current baseline | Target |
|---|---|---|
| Leads processed per run with no manual intervention | 0 (fully manual) **[Brief]** | **≥ 100 leads, zero crashes** **[Brief: minimum bar]** |
| Time per qualification decision | **8–12 min** per lead **[Brief]** | "Faster than manual" **[Brief]**. Working target: < 5 s per lead, so a 100-lead run finishes in minutes **[Assumption]** |
| Share of inbound leads qualified and contacted | **5%** (60 of 1,200 a month) **[Brief]** | Not set in the brief. Working target: **20–30%**, matched to how many leads reps can actually work once triage time is freed up **[Assumption; see `docs/PLAN.md` §2.2]** |
| LLM calls per run | n/a | **Batched**, never one call per lead **[Brief]**. Working target: ⌈leads sent to LLM ÷ 15⌉ **[Assumption]** |
| Decision quality | No rubric | **3–5 explicit factors** with stated weights. Decisions consistent and defensible; close calls go to human review **[Brief]** |
| Message quality | Written by hand | Personalized with specific lead details; **2–3 distinct variants**; on-brand **[Brief]** |

> **There's no ground truth.** None of the datasets has an outcome column (such as converted or closed-won). Grading is on **consistency and defensibility**, not accuracy against an answer key **[Brief: EXPECTED_OUTCOMES_NOTE]**. So you can't measure conversion lift yet. A reply-rate or conversion metric would need post-launch data **[Assumption]**.

## Data available (schema / data dictionary)

### Tables

| Table (file) | Rows | Purpose | Data quality |
|---|---|---|---|
| `data/leads_training.csv` | **29** (README says 30) **[Observed]** | Build and test the rubric and prompts | Complete |
| `data/leads_full_scale_100.csv` | 100 | Production-scale run; submit the report from this file | Complete. **Rows 1–29 repeat the training file** (with one name differing slightly) **[Observed]** |
| `data/leads_testing.csv` | 20 | Robustness and edge cases | About 60% complete by design: missing, `NA`, and unknown values; extreme sizes; stale dates **[Brief + Observed]** |

All three files share the same schema.

### Columns

| Column | Type (expected → as found) | Description **[Brief]** | Observed values and quirks **[Observed]** |
|---|---|---|---|
| `name` | string | Contact's full name | Personal data. In the testing file, names are case labels (`Test_Solo`). One row has a person's name in the **company** column. |
| `company` | string | Company name | Mostly descriptive ("Cloud Infrastructure"). One row is blank (`Test_MissingName`). |
| `company_size` | integer (employees) → **string** | Number of employees, approximate; the docs say to use ranges | Range **1–50,000** across files (3–8,000 in the 100-lead file). Uses the literal string `NA` for unknown, even though the README says missing is blank. |
| `industry` | categorical string | Business industry | **46** distinct labels in the 100-lead file, with overlap (SaaS / Software / Enterprise Software / Technology). Includes `Unknown Sector` and `NA`. |
| `source` | categorical string | How the lead was acquired | 6 known values: Inbound demo request, LinkedIn outreach, Referral, Webinar attendee, Content download, Sales call. Also blank, `NA`, and `Unknown Source`. Values may grow in production **[Brief]**. |
| `last_interaction_date` | date `YYYY-MM-DD` → **string** | Last time the company was in touch with the lead | 2022-06-15 to 2024-01-20. The 100-lead file starts at 2023-08-30 (the README says 2023-12-05). Also blank and `NA`. Every date is about **2.8 years** before today. |

**Columns that don't exist:** email, phone, job title, website, budget, timeline, deal stage, interaction count, and any outcome or conversion label **[Observed]**. Signals like "budget" or "timeline" that the brief mentions as example rubric factors **can't be measured from this data**.

## Fields the current solution already uses

The brief doesn't document what reps look at today. It only says decisions are manual **[Brief]**.

- **[Assumption]** Reps see the same 6 fields shown above, from the CRM or an inbound form, and judge mostly on **company size, industry, and source**, plus gut feel.
- **The fields the new system will use:** all 6.
  - `company_size`, `industry`, `source`, and `last_interaction_date` → scoring factors.
  - `name` and `company` → personalization and deduplication only. Never used as a scoring signal.

## Sample rows (masked)

**Masking rule:** real-looking contact names are reduced to initials. The `Test_*` labels in the testing file are kept because they aren't personal data, and they document each edge case. Company names are kept because they're fictional and generic. Raw values (`NA`, blanks) are shown exactly as they appear in the files.

36 rows: 16 from the clean files (clear wins, clear rejects, ambiguous leads, and a duplicate pair) and all 20 failure and edge-case rows from the testing file.

| # | File | name (masked) | company | company_size | industry | source | last_interaction_date | Why included |
|---|---|---|---|---|---|---|---|---|
| 1 | full_scale_100 | A. C. | CloudScale AI | 50 | SaaS | Inbound demo request | 2024-01-15 | Clear win: ICP size, core industry, demo request, recent |
| 2 | full_scale_100 | B. R. | TechVentures Inc | 8 | SaaS | LinkedIn outreach | 2024-01-20 | Too small; outbound source |
| 3 | full_scale_100 | C. S. | National Bank Corp | 5000 | Finance | Content download | 2023-12-01 | Enterprise, low-intent source |
| 4 | full_scale_100 | D. M. | StartupXYZ | 3 | SaaS | Referral | 2024-01-18 | 3 employees: knockout |
| 5 | full_scale_100 | E. T. | RetailWorks Solutions | 200 | Retail | Inbound demo request | 2023-11-10 | Good fit, but 71 days since last touch |
| 6 | full_scale_100 | F. L. | Healthcare Systems Ltd | 1200 | Healthcare | Sales call | 2024-01-12 | Mid-market healthcare, sales call: solid but not obvious |
| 7 | full_scale_100 | G. L. | Manufacturing Co | 450 | Manufacturing | Webinar attendee | 2023-10-05 | Older than the README's stated date range |
| 8 | full_scale_100 | N. B. | Enterprise Solutions Corp | 3500 | Software | Inbound demo request | 2024-01-13 | Large company, high intent: ambiguous |
| 9 | full_scale_100 | O. G. | Local Restaurant Group | 12 | Hospitality | LinkedIn outreach | 2023-08-30 | Oldest date in file; poor-fit industry |
| 10 | full_scale_100 | R. K. | Government Agency | 800 | Government | Content download | 2023-12-10 | Government: long procurement |
| 11 | full_scale_100 | A. B. | Supply Chain Co | 320 | Supply Chain | LinkedIn outreach | 2024-01-02 | Near-duplicate of training row "Aaron Benj" |
| 12 | full_scale_100 | D. J. | Cloud Infrastructure | 200 | SaaS | Inbound demo request | 2024-01-19 | Clear win |
| 13 | full_scale_100 | H. K. | Oil & Gas Corp | 8000 | Energy | Content download | 2023-09-01 | Largest in file (8,000) |
| 14 | full_scale_100 | M. P. | Blockchain Startup | 22 | Cryptocurrency | Referral | 2024-01-15 | Niche/regulated industry |
| 15 | full_scale_100 | Q. M. | Volunteer Management | 40 | NonProfit | Webinar attendee | 2023-12-09 | Non-profit |
| 16 | training | A. B. | Supply Chain Co | 320 | Supply Chain | LinkedIn outreach | 2024-01-02 | Near-duplicate of full-scale row "Aaron Benjamin" |
| 17 | testing | Test_MissingSize | Test Inc | NA | SaaS | Inbound demo request | 2024-01-20 | Size is the literal string `NA` |
| 18 | testing | Test_VeryLargeEnterprise | Global Fortune 500 Corp | 50000 | Enterprise Software | Inbound demo request | 2024-01-20 | Size 50,000, far outside any mid-market ICP |
| 19 | testing | Test_Solo | John Doe Consulting | 1 | Consulting | LinkedIn outreach | 2024-01-20 | 1 employee: knockout |
| 20 | testing | Test_StaleInteraction | Old Company Inc | 100 | Manufacturing | Sales call | 2022-06-15 | Last touch about 19 months before the newest date |
| 21 | testing | Test_RecentEngagement | Hot Lead Corp | 200 | SaaS | Inbound demo request | 2024-01-20 | Same-day interaction, strong fit |
| 22 | testing | Test_MissingDate | Mystery Co | 150 | Technology | *(blank)* | *(blank)* | Source **and** date blank |
| 23 | testing | Test_UnknownIndustry | Mysterious Startup | 15 | Unknown Sector | Referral | 2024-01-19 | `Unknown Sector`, an unmapped industry |
| 24 | testing | Test_VeryRecentSignup | Brand New Startup | 2 | SaaS | Inbound demo request | 2024-01-20 | Strong intent but 2 employees |
| 25 | testing | Test_MultipleInteractions | Active Company | 500 | Finance | Inbound demo request | 2024-01-20 | Label implies an interaction count; no such column exists |
| 26 | testing | Test_NoSourceInfo | Source Unknown Corp | 300 | Retail | Unknown Source | 2024-01-18 | `Unknown Source` |
| 27 | testing | Test_HighValueMatch | Perfect Fit Enterprise | 2000 | SaaS | Inbound demo request | 2024-01-20 | Label and company name hint at the answer (label-leak risk) |
| 28 | testing | Test_LowValueNiche | Niche Small Op | 5 | Agriculture | LinkedIn outreach | 2024-01-15 | 5 employees, Tier-3 industry, outbound |
| 29 | testing | Test_MissingName | *(blank)* | 400 | Consulting | Sales call | 2024-01-19 | Label says name missing, but the **company** is blank (column shift) |
| 30 | testing | Test_NameOnly | J. W. *(person's name in company column)* | NA | NA | NA | NA | 4 of 5 attributes are literal `NA`; insufficient data |
| 31 | testing | Test_PartialData | Tech Partial | NA | SaaS | *(blank)* | 2024-01-20 | Size `NA` and source blank |
| 32 | testing | Test_WeekOldInteraction | Week Old Corp | 250 | Logistics | Webinar attendee | 2024-01-13 | Typical mid-fit lead |
| 33 | testing | Test_MidMarketSoft | Mid Market Software | 1500 | SaaS | Content download | 2024-01-18 | Strong fit, low-intent source |
| 34 | testing | Test_StartupShutdown | Failed Startup | 8 | SaaS | Referral | 2023-05-01 | Label implies the company failed; data can't show this |
| 35 | testing | Test_GovernmentBuyer | Government Tech | 3000 | Government | Sales call | 2024-01-17 | Government, large |
| 36 | testing | Test_SMBAverage | Average SMB Co | 120 | Retail | Inbound demo request | 2024-01-19 | Average SMB, high intent |

## Constraints

### Privacy and consent
- **[Brief]** Don't submit API keys. Free-tier LLM APIs are allowed.
- **[Assumption]** `name` is personal data, and it's sent to a third-party LLM API. Free tiers may keep data or use it for training. So send the **minimum** needed: first name only, for personalization. Don't send it in scoring calls, and don't log full rows.
- **[Assumption]** The consent basis for contacting these leads isn't stated. Outreach messages are therefore **drafts for a rep to review and send**, not auto-sent. Anti-spam rules (CAN-SPAM, and GDPR or ePrivacy for EU contacts) apply to whoever sends them. No email addresses are present, so the system can't send anything anyway.

### Latency and cost budget
- **[Brief]** There's no explicit latency budget. The bar is "faster than manual" (8–12 min per lead) and **≥ 100 leads in one run without intervention**.
- **[Brief]** Calls must be batched, never one per lead. The system must handle API timeouts and rate limits (retry, log, alert).
- **[Assumption]** Stay within free-tier rate limits, so throttle requests per minute. A 100-lead run should finish in a few minutes.

### Fields filled only after the outcome is known (leakage)
- **[Observed]** There are no outcome fields (no conversion, deal stage, or won/lost), so there's no direct leakage.
- **[Assumption, risk]** `last_interaction_date` could be **updated by sales activity that happens after qualification** (for example, a follow-up call). If the data is re-exported from a CRM later, recency would partly reflect outcomes. Score using the value at intake, against a fixed `reference_date`.
- **[Observed, risk]** In the testing file, the **name and company text hint at the expected answer** ("Test_HighValueMatch", "Perfect Fit Enterprise", "Failed Startup"). These must not influence scoring.

### Legally or ethically off-limits
- **[Assumption]** Never infer or use **protected attributes** (gender, ethnicity, nationality, religion) from contact names. The names in this data are diverse, and an LLM could pick up bias from them. Names are therefore excluded from scoring, and the prompt forbids using them as a signal.
- **[Assumption]** Messages must not invent facts (prior conversations, metrics, events) that aren't in the data. That's both a trust issue and a deceptive-marketing risk.

### Delivery constraints
- **[Brief]** Code in your choice of language (Python, Node.js, or another). Must use an LLM API. The rubric needs 3–5 factors with explicit weights. Close calls must be flagged for human review, not guessed. The README is 1 page maximum. Keep it simple; it isn't meant to be a production system.
- **[Brief]** Submission is a zip containing the source code, `leads.csv`, `output_report.json` (or CSV), and the README. It must not contain API keys, large intermediate files, or model weights.
- **[Assumption]** Time box is **5 hours** (the brief allows 5–7 days / 8–10 hours).
