---
name: strategic-analysis
description: >
  Deep strategic analysis for B2B SaaS portfolio companies. Pulls CRM data (Salesforce or HubSpot),
  call transcripts (Gong, Fathom, or manual), interview evidence, and optional Revelio Labs enrichment.
  Answers: why we win/lose, ICP (descriptive and prescriptive), competitive positioning, feature gaps
  with ARR impact, pricing sensitivity, sales cycle dynamics, feature-persona importance mapping
  (which features matter most to which buyer personas), and
  question catalog (what prospects ask, segmented by persona and deal stage),
  Layered output (exec summary / medium /
  deep dive) plus Google Doc export. Use when the user says "strategic analysis", "win/loss", "why are we
  losing", "deal analysis", "ICP analysis", "competitive analysis", "persona analysis", "feature importance",
  "question catalog", "what are prospects asking",
  or wants a comprehensive view of a portfolio company's go-to-market performance.
user_invocable: true
---

# Strategic Analysis

Pull CRM deals, call transcripts, and interview evidence for a B2B SaaS company. Link them together, analyze by segment, and produce a layered report answering: why you win, why you lose, who your ICP is (and should be), what feature gaps cost you, how competitors beat you, and what data you're missing.

This skill is **CRM-agnostic** (Salesforce or HubSpot) and **call-recorder-agnostic** (Gong, Fathom, or manual paste).

**Output:** A layered .md report (executive summary, medium dive, deep dive) plus a standalone ICP file and a Google Doc.

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `--company Name` | *(required)* | Portfolio company name from companies.json |
| `--days N` | 180 | Lookback window in days |
| `--depth shallow\|medium\|deep\|all` | all | Which output layers to generate |
| `--segment acv\|vertical\|lifecycle\|source\|competitor\|owner\|all` | all | Segmentation dimensions to include |
| `--enrich` | false | Enable Revelio Labs enrichment via Snowflake MCP (adds headcount, growth, industry data) |
| `--skip-calls` | false | Skip live call recorder API calls; use only exported files |
| `--skip-ml` | false | Skip XGBoost predictive modeling |
| `--skip-gdoc` | false | Skip Google Doc creation |
| `--personas` | true | Enable buyer persona identification via Prospeo enrichment + CRM contacts. Set to false to skip persona analysis. |
| `--skip-prospeo` | false | Skip Prospeo API calls; use CRM Contact title fields only for persona classification |
| `--skip-questions` | false | Skip question catalog generation. Set to true to omit `/question-catalog` from the pipeline. |
| `--skip-transcript-intel` | false | Skip structured transcript intelligence. If true, later modules fall back to raw transcript search and recorder summaries. |
| `--transcript-scope closed-linked\|all-linked\|all` | closed-linked | Which calls receive full transcript processing. Use `closed-linked` by default for outcome-linked strategic analysis. |

**Announce:** "Running `/strategic-analysis` for {company} -- pulling CRM data, call transcripts, and interview evidence."

---

## Analytical Principles

These rules apply to EVERY analysis module and EVERY finding in the report. Violating these rules produces misleading analysis. Follow them strictly.

### Repeatability Over Outliers

- **Minimum sample size:** Do not draw conclusions from segments with fewer than 5 deals. Show the data, but label it "Insufficient sample (N={count}), not actionable."
- **Outlier flagging:** For any segment, identify deals that are >3x the median deal size. Report the metric WITH and WITHOUT the outlier. Example: "Partner channel: 55.6% win rate (20W/16L). Removing the $76K Smartrr migration outlier: 51.5% (17W/16L). The pattern holds."
- **Pattern threshold:** A finding is a "pattern" only if it appears in 5+ deals AND persists across at least 2 quarters. Otherwise label it "Emerging signal" not a conclusion.
- **Revenue concentration check:** For any revenue-based finding, report what % of the total comes from the top 1-2 deals. If >50% of a segment's revenue comes from 1 deal, lead with the deal count story, not the revenue story.
- **Median over mean:** Always use median for deal size and velocity. Report mean only as a secondary reference. Mean is easily distorted by a single large deal in a field of small ones.
- **Sub-segment before recommending:** Before recommending action on any segment (e.g., "invest in Partner channel"), always break it into sub-segments (e.g., agency vs. tech partners, by tier within the channel). The actionable insight is usually one level deeper than the top-line number.
- **One-time events:** Flag any pattern that could be driven by a non-repeatable event (e.g., a competitor shutting down, a platform migration wave, a one-time referral batch). Ask: "Would this pattern hold next quarter if that event didn't recur?"

### Field Semantics (CRITICAL)

Before using ANY financial field (ARR, Amount, Revenue, ACV) in analysis, verify it represents **deal-level value**, not account-level. This is a common Salesforce pattern: a field called `AARR__c` on the Opportunity object may actually contain the Account's total ARR (via formula or rollup), not the specific deal's ARR. Using account-level fields for deal-level pricing analysis produces fundamentally wrong results.

**Mandatory validation for every financial field:**
1. Query 3-5 Opportunities from the SAME Account. If the field shows identical values across all of them, it is account-level, not deal-level. Flag it and do NOT use it for pricing, revenue, or deal size analysis.
2. Check the field's `calculated` property from the describe API. Formula fields on Opportunity that return the same value across sibling opps are almost certainly account-level.
3. When multiple value fields exist (Amount, ARR, estimated_ARR, etc.), compare them on 10 won deals. If any field is consistently >2x the Amount field, it is likely account-level. Report the discrepancy before proceeding.
4. **Default to Amount** as the deal-level value field unless a deal-specific ARR field is explicitly confirmed. Amount is the standard Salesforce deal value field and is almost always deal-level.

**If an account-level field is detected:** Remove it from the `get_arr()` priority order. Note in methodology: "Field {name} was identified as account-level (same value across sibling opportunities) and excluded from deal-level analysis. Amount used instead."

### Causal Language

Do not claim X causes Y from observational data. Correlations in CRM data almost always have confounding variables.

- **Use:** "X is associated with higher win rates" / "X is a signal of Y" / "Deals with X win at {N}% vs {M}% without"
- **Do not use:** "X drives wins" / "X causes losses" / "Doing more X will improve win rates"
- **Specifically for engagement metrics** (call counts, email touches, meeting counts): Higher engagement on won deals usually reflects buyer seriousness, not sales effort causing wins. Frame as: "Low engagement (1 call, no follow-up) is a disqualification signal indicating the buyer was not in a real evaluation" rather than "More calls lead to more wins."
- **For any finding that could inform a 'do more of X' recommendation:** State the confounding variable explicitly. Example: "Partner-sourced deals win at 34.6%. This could be because partners bring better-qualified leads, because partners multi-thread naturally, or because partners only refer deals they believe will close. The data cannot distinguish between these explanations."

### Segmented by Default

**Never present a company-wide metric without immediate segmentation.** Every core metric must be shown by at least 2 segment dimensions. Aggregates hide the real story.

- **Win rate:** Always show by tier AND source AND deal type (new vs. existing). A 57% overall win rate that is really 33% new business and 100% renewals is misleading.
- **Deal velocity:** Always show by tier AND outcome (won vs. lost). A 21-day median that is really 6 days for SMB and 31 days for enterprise tells a different story.
- **Pricing:** Always show win rate by price band AND by tier. A $10K ceiling might only apply to SMB; enterprise might have a different ceiling.
- **Loss reasons:** Always show by tier AND by competitor. "Bad Timing" might mean something different for a $5K SMB deal vs. a $75K enterprise deal.
- **Competitor analysis:** Always show by tier AND by source channel. Losing to Recharge at Tier 2 is a different problem than losing to Recharge at Tier 3.
- **Revenue metrics:** Always separate new business from existing business. Renewals inflate every aggregate.

---

## Step 0: Load Configuration and Check Prerequisites

### 0a: Read Company Config

Read `~/.claude/skills/strategic-analysis/companies.json` and look up the `--company` value (case-insensitive match).

### 0b: Company Not Found

If the company is not in companies.json, stop and say:

> "{Company}" isn't configured yet. To add it:
>
> **For Salesforce:**
> 1. Connect: `sf org login web --alias your-org-alias`
> 2. Add an entry to `~/.claude/skills/strategic-analysis/companies.json` (copy the example entry as a template, set `crm.type` to `"salesforce"`, set `crm.field_mapping` to `"auto"` for first-run discovery)
>
> **For HubSpot:**
> 1. Ensure HubSpot MCP is configured with a private app token
> 2. Add an entry with `crm.type` set to `"hubspot"` and `crm.field_mapping` set to `"auto"`
>
> Then run `/strategic-analysis --company {Company}` again.

### 0c: Check CRM Connection

**Salesforce:** Run `sf org list --json` and verify the company's `crm.sf_org` value appears. If not, stop with auth instructions.

**HubSpot:** Verify HubSpot MCP tools are available by attempting a test query. If MCP is unavailable, stop with setup instructions.

### 0d: Check Call Recorder

**Gong:** Run `source ~/.env` and check `GONG_API_KEY` and `GONG_API_SECRET` exist. If missing and `--skip-calls` is not set, warn and auto-enable `--skip-calls`.

**Fathom:** Run `source ~/.env` and check `FATHOM_API_KEY` exists. If missing and `--skip-calls` is not set, warn and auto-enable `--skip-calls`.

**Manual:** No check needed. User will paste transcripts during Agent 2.

**Gong + Salesforce: Detect integration method.** If `call_recorder.type` is `"gong"` and `crm.type` is `"salesforce"`, probe for the Gong managed package:

```bash
sf sobject list --target-org {config.crm.sf_org} 2>&1 | grep -q "Gong__Related_Opportunity__c"
```

If found, note which Gong objects exist (`Gong__Gong_Call__c`, `Gong__Related_Opportunity__c`, etc.) and describe `Gong__Gong_Call__c` to identify the field that stores the Gong-native call ID (commonly `Gong__Call_ID__c` but may vary by managed package version). Store the field name for use in Step 2b.

If the managed package is NOT found, the skill proceeds with the Gong REST API path only (current behavior).

### 0e: Check Portfolio Path

Verify `portfolio_path` from config exists. Check for `knowledge/` and `deliverables/` subdirectories. Create `deliverables/` if missing.

### 0f: Check Python Dependencies (if --skip-ml not set)

**Always check DuckDB for transcript intelligence unless `--skip-transcript-intel` is set.** DuckDB is free/open-source and runs locally as an embedded analytical database.

```bash
python3 -c "import duckdb; print('DuckDB OK')" 2>/dev/null
```

If this fails, attempt install:
```bash
pip3 install --user duckdb 2>/dev/null || pip3 install --break-system-packages duckdb 2>/dev/null
```

If install fails, auto-set `--skip-transcript-intel` and note: "Skipping transcript intelligence (DuckDB not available). Later modules will use raw transcript search and recorder summaries only."

```bash
python3 -c "import xgboost, sklearn, pandas, numpy; print('ML dependencies OK')" 2>/dev/null
```

If this fails, attempt install:
```bash
pip3 install --user xgboost scikit-learn pandas numpy 2>/dev/null || pip3 install --break-system-packages xgboost scikit-learn pandas numpy 2>/dev/null
```

If install fails, auto-set `--skip-ml` and note: "Skipping predictive modeling (XGBoost not available). The rest of the analysis proceeds without it."

### 0g: Check Snowflake (if --enrich set)

Verify Snowflake MCP tools are available by checking for `mcp__snowflake__run_snowflake_query`. If unavailable, warn: "Snowflake MCP not connected. Skipping Revelio Labs enrichment." and disable enrichment.

### 0h: Check Prospeo API (if --personas enabled and --skip-prospeo not set)

```bash
source ~/.env && [ -n "$PROSPEO_API_KEY" ] && echo "Prospeo API key found" || echo "MISSING"
```

If `PROSPEO_API_KEY` is missing:
- Warn: "Prospeo API key not found in `~/.env`. Persona enrichment will use CRM Contact title fields only (lower accuracy). To enable LinkedIn-based persona classification, add `PROSPEO_API_KEY=your_key` to `~/.env`. Get a key at prospeo.io (Starter plan: $39/mo, 1,000 credits)."
- Auto-set `--skip-prospeo` to true.

If key exists, verify it works:

```bash
source ~/.env && curl -s -o /dev/null -w "%{http_code}" -X POST https://api.prospeo.io/enrich-person \
  -H "X-KEY: $PROSPEO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"data": {"linkedin_url": "https://www.linkedin.com/in/test"}}'
```

If response is 401, warn about invalid key and auto-set `--skip-prospeo`. A 400 (NO_MATCH) is expected and means the key works.

---

## Step 0.5: Feature Intel Check (Strongly Recommended)

Before pulling CRM data, check for a cached feature-intel taxonomy and prompt the user:

> **Strongly recommended:** Run `/feature-intel` before proceeding. This generates a structured feature taxonomy that grounds the feature-persona matrix (Module 4J). Without it, features are auto-discovered from transcripts and CRM fields (moderate confidence). With it, every feature mention maps to a verified taxonomy (high confidence).
>
> "Do you want to run `/feature-intel` to generate the feature taxonomy? This significantly improves the feature-persona matrix by providing a structured list of capabilities to map buyer preferences against. It also augments competitive intelligence and feature gap accuracy. Adds ~10-15 minutes but the taxonomy is cached for future runs."

**If yes:**

1. Identify the top 2-3 competitors from the user (or note you'll confirm after CRM data is pulled in Step 4D)
2. Run: `/feature-intel [portfolio company] vs [competitor1] vs [competitor2] --portfolio [portfolio company]`
3. The report and per-company feature caches are saved to `~/Desktop/feature-intel/<slug>/`
4. A knowledge file is also saved to `{portfolio_path}/knowledge/feature-comparison-YYYY-MM.md`
5. After completion, read that knowledge file — it becomes an additional data source for Steps 4D and 4G

**If no:** Skip and proceed to Step 1. The analysis will rely solely on CRM fields, call transcripts, and interview evidence for feature gap and competitive claims. Module 4J will auto-generate a feature taxonomy from deal evidence.

**Taxonomy mode reporting:** The report will note which mode was used:
- If /feature-intel taxonomy exists: "Feature taxonomy: from /feature-intel (high confidence)"
- If auto-generated: "Feature taxonomy: auto-generated from deal evidence (moderate confidence, consider running /feature-intel for a more complete picture)"

**If feature intel was run previously (cached):** Check whether `{portfolio_path}/knowledge/feature-comparison-*.md` already exists. If it does, ask: "I found a feature comparison from [date]. Do you want to use this or run a fresh scrape?" Use the existing file unless the user wants a refresh.

---

## Step 1: Field Discovery

**Only runs if `crm.field_mapping` is `"auto"` or empty.** If a mapping already exists in companies.json, skip to Step 2.

### Salesforce Field Discovery

```bash
sf sobject describe Opportunity --target-org {config.crm.sf_org} --json
```

Parse the response `fields` array. For each analysis concept, look for matching field names using these heuristics:

| Concept | Search patterns |
|---------|----------------|
| amount | `Amount`, `*ARR*`, `*ACV*`, `*Revenue*` |
| loss_reason | `*Lost*Reason*`, `*Closed*Lost*`, `*Loss*Reason*` |
| competitor | `*Competitor*`, `*Lost*To*` |
| deal_source | `*Source*`, `*Lead*Source*`, `*Channel*` |
| vertical_or_industry | `*Industry*`, `*Vertical*`, `*Segment*` |
| deal_tier | `*Tier*`, `*Segment*` (if not matched to vertical) |
| employee_count | `*Employee*`, `*Company*Size*` |
| product_gap | `*Gap*`, `*Feature*`, `*Product*Reason*` |
| stage_date_fields | `Date_Entered_*` pattern |
| won_reason | `*Won*Reason*`, `*Why*Won*`, `*Differentiator*`, `*Key*Factor*`, `*Closed*Won*AI*` |

**Gong managed package signal:** If any Opportunity field with prefix `Gong__` is found during discovery (e.g., `Gong__Gong_Count__c`, `Gong__MainCompetitors__c`), this confirms the Gong Salesforce managed package is installed. Note this for Step 2b so the managed package path is used for CRM associations instead of the Gong REST API.

Present the discovered mapping to the user:

> I found these field mappings for {company}'s Salesforce:
>
> | Concept | Field | Sample Values |
> |---------|-------|---------------|
> | Loss reason | Closed_Lost_Reason__c | "Competitor", "Price", "No Decision" |
> | ...
>
> Does this look right? Any corrections?

After confirmation, save the mapping to `companies.json`.

### HubSpot Field Discovery

Use HubSpot MCP to list all deal properties. Apply the same heuristic matching. Additionally:
- Query pipeline stage definitions to map stage IDs to human-readable names
- Note that HubSpot deal stages use internal IDs, not text values

### Post-Discovery Checks

1. **Stage date fields:** If no `Date_Entered_*` or equivalent fields found, note: "No stage date fields found. Deal velocity analysis will be skipped."
2. **Loss reason type:** Check if loss_reason values are from a picklist (structured) or free text. If >20 unique values and average length >20 characters, flag as free text: "Loss reasons appear to be free text. I'll cluster them into categories during analysis."
3. **Sparse fields:** For each mapped field, check population rate. Flag any field with <50% fill rate.

### Financial Field Validation (MANDATORY)

**This step must run before saving the field mapping.** It prevents the most damaging analytical error: using account-level fields for deal-level analysis.

For every field mapped to `amount`, `arr`, or any financial concept:

**Step A: Sibling test.** Find an Account with 3+ Opportunities and compare the field values:

```bash
sf data query --query "
SELECT AccountId, COUNT(Id) FROM Opportunity 
WHERE {financial_field} != null 
GROUP BY AccountId 
HAVING COUNT(Id) >= 3
ORDER BY COUNT(Id) DESC
LIMIT 1
" --target-org {config.crm.sf_org} --json
```

Then query those sibling opps:

```bash
sf data query --query "
SELECT Id, Name, Amount, {financial_field}
FROM Opportunity
WHERE AccountId = '{account_id}'
AND {financial_field} != null
ORDER BY CreatedDate DESC
LIMIT 5
" --target-org {config.crm.sf_org} --json
```

If all 5 opps show the SAME value for `{financial_field}` but DIFFERENT values for `Amount`, the field is **account-level**. Flag it: "WARNING: {field} appears to be account-level (identical values across sibling opportunities). Demoting to secondary. Amount will be used as the primary deal-level value."

**Step B: Magnitude test.** On 10 won deals, compare each financial field to Amount:

```bash
sf data query --query "
SELECT Amount, {arr_field}, {revops_field}
FROM Opportunity
WHERE StageName IN ({won_values}) AND Amount > 0
ORDER BY CloseDate DESC
LIMIT 10
" --target-org {config.crm.sf_org} --json
```

If any field consistently exceeds Amount by >3x, flag it as likely account-level.

**Step C: Set the value field priority.** Based on Steps A and B, establish the priority order for deal-level value:

1. `Amount` (always deal-level in standard Salesforce)
2. Any confirmed deal-level ARR field (passed sibling test)
3. Account-level fields are EXCLUDED from deal-level analysis but noted for context

Report the validated priority order: "Deal-level value field: Amount (primary), {field} (confirmed deal-level, secondary). Excluded: {field} (account-level)."

### Contact & Persona Field Discovery (if --personas enabled)

**Salesforce Contact fields:**

```bash
sf sobject describe --sobject Contact --target-org {config.crm.sf_org} --json
```

Search for persona-relevant fields:

| Concept | Search patterns |
|---------|----------------|
| title | `Title`, `*Job*Title*` |
| department | `*Department*` |
| role_in_deal | `*Role*` (on OpportunityContactRole) |
| linkedin_url | `*LinkedIn*`, `*Linkedin*URL*`, `*Social*` |
| seniority | `*Level*`, `*Seniority*` |

**Salesforce OpportunityContactRole fields:**

```bash
sf sobject describe --sobject OpportunityContactRole --target-org {config.crm.sf_org} --json
```

OpportunityContactRole links Contacts to Opportunities. Key fields: `ContactId`, `OpportunityId`, `Role`, `IsPrimary`. The `Role` field typically contains values like "Decision Maker", "Economic Buyer", "Influencer", "Technical Evaluator", "Executive Sponsor".

**HubSpot contact association:**

Use HubSpot MCP to list contact associations for deals. HubSpot uses the Associations API to link contacts to deals. Key properties to request: `jobtitle`, `hs_linkedinbio`, `department`.

---

## Step 2: Parallel Data Collection

Launch up to 4 agents simultaneously. Each returns structured data.

### Agent 1: CRM Deals

Build a query dynamically from `crm.field_mapping`. Include only fields that have non-empty mappings.

**Salesforce:**

Construct the SELECT clause from all mapped field names. Build the WHERE clause from `won_values`, `lost_values`, `dq_values`, `nurture_values`:

```bash
sf data query --query "
SELECT {all_mapped_fields_comma_separated}
FROM Opportunity
WHERE StageName IN ({all_stage_values_quoted})
AND (CloseDate >= LAST_N_DAYS:{DAYS} OR CreatedDate >= LAST_N_DAYS:{DAYS})
ORDER BY CloseDate DESC NULLS LAST
" --target-org {config.crm.sf_org} --json
```

If result set >2000 records, add `--bulk` flag.

**HubSpot:**

Use HubSpot MCP to search deals with filters:
- `dealstage` IN (mapped stage values)
- `closedate` >= lookback date
- Request all mapped properties
- Paginate (max 100 per request)

**Important:** For HubSpot, first query pipeline stage definitions to build a stage ID-to-name map. Use this map to translate stage IDs throughout the analysis.

**Data Validation (both paths):**

After pulling deals, report:

> Found **{N}** closed deals ({won} won, {lost} lost, {dq} disqualified, {nurture} nurture) from {date_range}.
>
> Quick sanity check:
> - Total deal value: ${total}
> - Average deal size (won): ${avg_won}
> - Win rate: {win_rate}% (won / (won + lost))
>
> Does this look roughly right?

If <30 deals, warn: "Small sample size ({N} deals). Analysis will have limited statistical power. Consider expanding the lookback window with `--days 365`."

### Agent 2: Call Transcripts

**If `--skip-calls` is set:** Skip this agent. Read fallback files from `config.interview_paths.gong_exports` glob pattern instead. Note in methodology.

#### Gong Path (if call_recorder.type is "gong")

**Authentication:**
```bash
source ~/.env
AUTH=$(echo -n "${GONG_API_KEY}:${GONG_API_SECRET}" | base64)
```

**2a: Get calls in date range:**
```bash
curl -s -X POST "${GONG_BASE_URL:-https://us-55616.api.gong.io}/v2/calls/extensive" \
  -H "Authorization: Basic ${AUTH}" \
  -H "Content-Type: application/json" \
  -d '{
    "filter": {
      "fromDateTime": "{START_ISO}",
      "toDateTime": "{END_ISO}"
    },
    "contentSelector": {
      "exposedFields": {
        "content": { "topics": true, "trackers": true, "pointsOfInterest": true },
        "parties": true
      }
    }
  }'
```

Handle pagination: if `records.cursor` is present, make additional requests with that cursor.

**2b: Get call-to-deal associations:**

Try sources in priority order. Stop at the first one that returns results.

**Source 1: Salesforce managed package (if detected in Step 0d):**

Query the junction object that links Gong calls to Opportunities:

```bash
sf data query --query "
SELECT Gong__Gong_Interaction__c, Gong__Related_Entity_ID__c,
  Gong__Opportunity_Name__c, Gong__Started__c
FROM Gong__Related_Opportunity__c
WHERE Gong__Related_Entity_ID__c != null
AND Gong__Started__c >= {START_ISO}
" --target-org {config.crm.sf_org} --json
```

Then resolve the Gong-native call IDs (needed for the transcript API) from the Gong Call object using whichever field was identified in Step 0d:

```bash
sf data query --query "
SELECT Id, {gong_call_id_field}
FROM Gong__Gong_Call__c
WHERE Id IN (
  SELECT Gong__Gong_Interaction__c
  FROM Gong__Related_Opportunity__c
  WHERE Gong__Related_Entity_ID__c != null
  AND Gong__Started__c >= {START_ISO}
)
" --target-org {config.crm.sf_org} --json
```

Build mapping: `{ sfOpportunityId: [gongNativeCallId, ...] }`.

**Source 2: Gong REST API (if managed package not detected, or returned empty):**

```bash
curl -s "${GONG_BASE_URL:-https://us-55616.api.gong.io}/v2/calls/manual-crm-associations" \
  -H "Authorization: Basic ${AUTH}"
```

Build mapping from `crmObjectId` values.

**If both return empty:** Report: "Found {N} Gong calls but 0 could be linked to Opportunities. Checked: SF managed package ({found/not found}), Gong REST API (empty). Transcript analysis will rely on topic/tracker aggregates only, not deal-level evidence. To investigate: check Gong admin settings for CRM integration status."

**2c: Get transcripts for linked calls (batches of up to 100):**
```bash
curl -s -X POST "${GONG_BASE_URL:-https://us-55616.api.gong.io}/v2/calls/transcript" \
  -H "Authorization: Basic ${AUTH}" \
  -H "Content-Type: application/json" \
  -d '{ "filter": { "callIds": ["{CALL_ID_1}", "{CALL_ID_2}"] } }'
```

**2d: Get AI summaries:**
```bash
curl -s -X POST "${GONG_BASE_URL:-https://us-55616.api.gong.io}/v2/calls/ai-content" \
  -H "Authorization: Basic ${AUTH}" \
  -H "Content-Type: application/json" \
  -d '{ "filter": { "callIds": ["{CALL_ID_1}", "{CALL_ID_2}"] } }'
```

**Rate limits:** 3 requests/second, 10,000/day. Add 350ms delay between requests. Handle 429 with `Retry-After` header.

#### Fathom Path (if call_recorder.type is "fathom")

**List recent meetings:**
```bash
source ~/.env && curl -s "https://api.fathom.ai/external/v1/meetings?created_after=$(date -v-{DAYS}d +%Y-%m-%dT00:00:00Z)" \
  -H "X-Api-Key: $FATHOM_API_KEY"
```

Response is in `.items[]`. Paginate with `next_cursor`.

**Select representative sample:** Pick 10-15 calls that appear to be sales/prospect calls (filter by attendee domains, exclude internal). Prioritize calls associated with closed deals if association data exists.

**Get transcripts:** For each selected meeting, fetch `/recordings/{id}/transcript`.

**Get summaries:** For broader pattern detection, fetch `/recordings/{id}/summary` for all calls.

**Linking to deals:** Fathom has no native CRM association. Match calls to deals by:
1. Attendee email domain matching to account domain
2. Attendee company name matching to deal account name
3. Note lower linking confidence in methodology

#### Manual Path (if call_recorder.type is "manual")

> No call recorder connected. You can still provide transcripts:
>
> **Option A:** Paste transcript text directly into chat
> **Option B:** Provide file paths to transcript files (.txt, .md, .pdf)
>
> Ideally provide 3-5 transcripts from a mix of won and lost deals. For each, tell me whether the deal was won or lost.

**All paths return:** Call ID/title, date, duration, participant names and roles, transcript text, AI summary (if available), linked deal ID (if known).

### Agent 2.5: Transcript Intelligence Layer

**Skip if `--skip-transcript-intel` is set or no transcripts are available.**

**Purpose:** Convert large volumes of raw call transcripts into a durable, queryable evidence store before analysis. Later modules should use this structured evidence first, not repeatedly search raw transcripts.

**Default architecture:** Use structured extraction as the source of truth. Use embeddings only in a future version for semantic retrieval, clustering, and quote search. Do not use embeddings as the primary analytical engine.

#### 2.5a: Create Cache Directory and DuckDB Store

Create:

```text
{portfolio_path}/strategic-analysis/cache/
  transcript_intelligence.duckdb
  raw_calls.jsonl
  transcript_chunks.jsonl
  extracted_signals.jsonl
  deal_evidence.jsonl
  rollups/
```

DuckDB file:

```text
{portfolio_path}/strategic-analysis/cache/transcript_intelligence.duckdb
```

DuckDB is local and free. There is no hosted service, account, or usage-based cost.

#### 2.5b: Normalize Call Records

Normalize Gong, Fathom, and manual transcripts to one schema:

```json
{
  "call_id": "string",
  "recorder_source": "gong|fathom|manual",
  "title": "string",
  "date": "ISO-8601",
  "duration_seconds": 0,
  "linked_deal_id": "string|null",
  "link_confidence": "high|moderate|low|none",
  "participants": [],
  "transcript_text": "string",
  "speaker_turns": [],
  "ai_summary": "string|null",
  "topics": [],
  "trackers": [],
  "points_of_interest": []
}
```

Write normalized calls to both DuckDB and `raw_calls.jsonl`.

#### 2.5c: Assign Processing Tier

Not every call deserves the same processing cost. Assign each call one of three tiers.

**Tier 1: Full processing**
- Closed won/lost deal-linked calls
- Late-stage opportunity calls
- Calls with competitor mentions from recorder metadata
- Pricing, procurement, security, implementation, or technical evaluation calls
- Enterprise/high-ACV calls

**Tier 2: Lightweight processing**
- Open opportunity calls
- Early discovery calls
- Lower-ACV calls with no risk/competitor/pricing markers

**Tier 3: Metadata/summary only**
- Unlinked calls
- Internal calls
- Very short calls
- Low-confidence account/domain matches

Processing by tier:

| Tier | Processing |
|------|------------|
| Tier 1 | Chunk transcript, extract structured signals, create per-call and per-deal summaries |
| Tier 2 | Use transcript or recorder AI summary to extract high-level themes only |
| Tier 3 | Store metadata, topics, trackers, and summary only; no chunk extraction |

Default `--transcript-scope closed-linked` means full processing is limited to calls linked to closed won/lost deals. If the user requests broader theme mining, use `all-linked` or `all`.

#### 2.5d: Chunk Transcripts

Chunk Tier 1 transcripts by conversation structure, not arbitrary character count.

Rules:
- Preserve speaker labels.
- Preserve timestamps.
- Target 500-1,500 tokens per chunk.
- Prefer Gong topics, points of interest, or speaker-turn topic boundaries.
- If no topic metadata exists, accumulate speaker turns until target size.
- Include 1-2 previous speaker turns as context overlap only when a topic crosses a chunk boundary.
- Attach each chunk to call ID, deal ID, deal outcome, stage, ACV band, segment, persona, competitor, source, and call date.

Chunk schema:

```json
{
  "chunk_id": "string",
  "call_id": "string",
  "deal_id": "string|null",
  "start_timestamp": "string|null",
  "end_timestamp": "string|null",
  "chunk_type": "discovery|demo|pricing|technical_eval|implementation|procurement|competitive|objection_handling|unknown",
  "speaker_turns": [],
  "text": "string",
  "token_count": 0,
  "metadata": {}
}
```

Write chunks to DuckDB and `transcript_chunks.jsonl`.

#### 2.5e: Extract Structured Signals

Run extraction on each Tier 1 chunk. Extract atomic evidence only. Do not ask for strategy or recommendations at this stage.

Signal types:

```text
buyer_question
pricing_objection
feature_request
feature_gap
competitor_mention
integration_requirement
implementation_concern
security_concern
roi_value_driver
buying_criteria
decision_process
stakeholder_signal
loss_risk
win_signal
```

Extraction schema:

```json
{
  "signal_id": "string",
  "signal_type": "string",
  "theme": "string",
  "speaker": "prospect|rep|customer|unknown",
  "persona": "Economic Buyer|Champion/User|Technical Evaluator|Executive Sponsor|Unknown",
  "sentiment": "positive|negative|neutral|mixed|unknown",
  "evidence_quote": "string",
  "confidence": "high|moderate|low",
  "chunk_id": "string",
  "call_id": "string",
  "deal_id": "string|null",
  "metadata": {
    "competitor": "string|null",
    "feature": "string|null",
    "stage": "string|null",
    "deal_outcome": "won|lost|dq|nurture|open|unknown"
  }
}
```

Validation rules:
- Every strategic signal needs a direct quote unless it is metadata-only.
- Keep quotes short and exact.
- Prospect/customer statements carry more analytical weight than rep statements.
- Separate "buyer said X" from "rep positioned X."
- Do not infer causality during extraction.
- Do not count repeated mentions from the same call as separate deal-level evidence unless different stakeholders raised the issue independently.

Write signals to DuckDB and `extracted_signals.jsonl`.

#### 2.5f: Build Per-Deal Evidence Summaries

For every deal with linked calls, build a compact evidence record:

```text
Deal: {account_name}
Outcome: {won/lost/dq/open}
Segment: {tier/vertical/source}
ACV: ${amount}
Evidence coverage: CRM + {N} calls + {M} interviews

Key buying criteria:
- {criterion}

Primary objections:
- {objection}

Competitor mentions:
- {competitor}: {summary}

Representative quotes:
- "{quote}" [Call: {title}, {date}]
```

Store in `deal_evidence` table and `deal_evidence.jsonl`.

#### 2.5g: Generate Analytical Rollups

Create rollup tables in DuckDB and export CSV/Markdown files under `cache/rollups/`:

```text
objections_by_segment
questions_by_persona
competitors_by_stage
feature_gaps_by_outcome
pricing_concerns_by_acv_band
implementation_concerns_by_deal_size
win_signals_by_source
loss_risks_by_competitor
```

Each rollup must include:
- Deal count
- Call count
- Signal count
- Won count
- Lost count
- Win rate where applicable
- Median ACV
- ARR exposure
- Representative quotes
- Evidence confidence

Apply the Analytical Principles:
- A pattern requires 5+ deals and persistence across at least 2 quarters.
- Smaller samples are labeled "Emerging signal."
- Segment by default.
- Separate new business from existing business.
- Avoid causal language.
- Report evidence strength.

#### 2.5h: Recommended DuckDB Schema

Create or update these tables:

```sql
CREATE TABLE IF NOT EXISTS analysis_runs (
  run_id VARCHAR,
  company VARCHAR,
  generated_at TIMESTAMP,
  days INTEGER,
  transcript_scope VARCHAR,
  extractor_version VARCHAR,
  chunker_version VARCHAR
);

CREATE TABLE IF NOT EXISTS calls (
  call_id VARCHAR,
  recorder_source VARCHAR,
  title VARCHAR,
  call_date TIMESTAMP,
  duration_seconds INTEGER,
  linked_deal_id VARCHAR,
  link_confidence VARCHAR,
  transcript_text VARCHAR,
  ai_summary VARCHAR,
  processing_tier INTEGER
);

CREATE TABLE IF NOT EXISTS transcript_chunks (
  chunk_id VARCHAR,
  call_id VARCHAR,
  deal_id VARCHAR,
  start_timestamp VARCHAR,
  end_timestamp VARCHAR,
  chunk_type VARCHAR,
  text VARCHAR,
  token_count INTEGER,
  metadata JSON
);

CREATE TABLE IF NOT EXISTS extracted_signals (
  signal_id VARCHAR,
  signal_type VARCHAR,
  theme VARCHAR,
  speaker VARCHAR,
  persona VARCHAR,
  sentiment VARCHAR,
  evidence_quote VARCHAR,
  confidence VARCHAR,
  chunk_id VARCHAR,
  call_id VARCHAR,
  deal_id VARCHAR,
  metadata JSON
);

CREATE TABLE IF NOT EXISTS deal_evidence (
  deal_id VARCHAR,
  account_name VARCHAR,
  outcome VARCHAR,
  segment VARCHAR,
  amount DOUBLE,
  evidence_coverage VARCHAR,
  summary VARCHAR,
  representative_quotes JSON
);
```

#### 2.5i: Cache Invalidation

Reuse cached transcript intelligence only when all of these match:
- `call_id`
- transcript last updated timestamp, if provided by the recorder
- `extractor_version`
- `chunker_version`
- analysis date window
- transcript scope

If the transcript text changed or the extractor/chunker version changed, reprocess that call.

#### 2.5j: Embeddings Future Version

Do not block v1 on embeddings. Add embeddings later for:
- semantic quote retrieval
- duplicate question clustering
- objection clustering
- related evidence search
- ad hoc analyst questions over transcripts

When implemented, embed both chunks and extracted signals. Embeddings support retrieval; they do not replace structured extraction or rollups.

### Agent 3: Knowledge Files

Read all qualitative data from the company's portfolio folder at `{config.portfolio_path}`.

**Files to read (in order of priority):**

1. Paths from `config.interview_paths.knowledge_files` (synthesized knowledge files)
2. `config.interview_paths.churn_interviews` directory
3. `config.interview_paths.market_research` directory
4. Files matching `config.interview_paths.gong_exports` glob
5. Standard locations: `knowledge/product.md`, `knowledge/competitive-landscape.md`, `knowledge/icp.md`

**Extract from each file:**
- Company names mentioned
- Churn/loss reasons cited
- Competitor mentions (positive, negative, neutral)
- Product gaps or feature requests
- Direct quotes explaining why a customer left or chose a competitor
- Market perception themes

**Return:** Structured index organized by company evidence and thematic evidence. If no interview files exist, note: "No interview data available. Analysis based on CRM and call data only."

### Agent 4: Revelio Labs Enrichment (if --enrich set)

**Only runs if `--enrich` flag is set and Snowflake MCP is available.**

After Agent 1 returns the deal list, take the unique account/company names and match them to Revelio Labs data.

**Step 4a: Match companies to Revelio RCIDs**

For each unique account name from CRM deals, query Snowflake:

```sql
SELECT RCID, PRIMARY_NAME, COMPANY_DISPLAY_NAME, URL,
  RAW_COUNT, YEAR_FOUNDED, NAICS_DESC, NAICS_DESC_2D,
  SAAS_COMPANY, HQ_COUNTRY, HQ_STATE
FROM REVELIO_LABS.YOUR_SCHEMA.COMPANY_REF_LATEST
WHERE LOWER(PRIMARY_NAME) LIKE LOWER('%{account_name}%')
   OR LOWER(COMPANY_DISPLAY_NAME) LIKE LOWER('%{account_name}%')
LIMIT 5
```

If multiple matches, pick the best match by name similarity. If no match, skip that account.

**Step 4b: Pull growth metrics for matched companies**

For each matched RCID:

```sql
SELECT RCID, COMPANY, CATEGORY, CATEGORY_TYPE,
  COUNT, RAW_COUNT, PERCENT_OF_COMPANY,
  GROWTH_1M, GROWTH_3M, GROWTH_6M, GROWTH_1Y
FROM REVELIO_LABS.YOUR_SCHEMA.GROWTH_METRICS_LATEST
WHERE RCID = {rcid}
AND CATEGORY_TYPE = 'all'
```

**Step 4c: Build enrichment dataset**

For each deal, attach (where available):
- `revelio_headcount`: actual employee count from Revelio (fills CRM gaps)
- `revelio_growth_1y`: 1-year headcount growth rate (growing vs. shrinking company)
- `revelio_industry`: NAICS industry classification (more standardized than CRM)
- `revelio_year_founded`: company age
- `revelio_is_saas`: SaaS flag

These enriched fields become additional columns in the XGBoost feature matrix (Step 4H) and provide backup data when CRM fields are empty.

**Report:** "Enriched {X} of {Y} accounts with Revelio Labs data (headcount, growth, industry)."

### Agent 5: Account-Level Retention Data (MANDATORY for Salesforce)

**This agent is REQUIRED, not optional.** Retention by segment is necessary for any segment recommendation (see Step 5.5). Without it, recommendations are labeled "Conditional."

**Purpose:** Calculate quarterly retention (logo and GDR) by tier/segment from Account-level data. The Opportunity object only has new business and competitive losses. Churn data lives on the Account object.

**Salesforce — discover retention fields:**

First, discover which Account fields exist for retention analysis:

```bash
sf sobject describe --sobject Account --target-org {config.crm.sf_org} --json
```

Search for fields matching these patterns:
- Status: `*Status*`, `*Stage*`, `*Active*`, `*Live*`, `*Customer*`
- Churn: `*Churn*`, `*Cancel*`, `*Churned*`
- Revenue: `*ARR*`, `*MRR*`, `*Revenue*`, `*Projected*`
- Dates: `*Live_Date*`, `*Go_Live*`, `*Churn_Date*`
- Tier: `*Tier*`, `*Segment*`
- Churn detail: `*Churned_Reason*`, `*New_Subscription_Vendor*`, `*Lost_To*`

**Salesforce — pull Account data:**

```bash
sf data query --query "
SELECT {tier_field}, {status_field}, {arr_field}, 
  {live_date_field}, {churn_date_field}, {churn_arr_field},
  {churn_reason_field}, {new_vendor_field}, Name
FROM Account 
WHERE {live_date_field} != null AND {tier_field} != null
ORDER BY {live_date_field} ASC
" --target-org {config.crm.sf_org} --json
```

**Calculate quarterly retention cohorts:**

For each quarter in the analysis period and each tier:
1. **Starting cohort**: accounts live before quarter-start AND not yet churned
2. **Churned in quarter**: accounts from starting cohort that churned during the quarter
3. **Logo retention**: (starting - churned) / starting
4. **GDR**: (starting ARR - churned ARR) / starting ARR

**Output required:**

```
Quarter | Tier   | Start | Churned | Logo Ret | Start ARR | Churned ARR | GDR
Q1'25   | Tier 1 |    66 |       5 |   92.4%  | $2.82M    | $799K       | 71.7%
Q1'25   | Tier 2 |    67 |       3 |   95.5%  | $531K     | $49K        | 90.9%
...
```

Plus annualized retention by tier, churn reasons by tier, and "churned to" by tier.

**REQUIRED output tables (all must be present or logged as gaps):**

1. **Logo retention by tier (quarterly):**

| Quarter | Tier | Starting Accounts | Churned | Logo Retention % |
|---------|------|-------------------|---------|-----------------|

2. **Dollar retention by tier (quarterly):**

| Quarter | Tier | Starting ARR | Churned ARR | GDR % |
|---------|------|-------------|------------|-------|

3. **Churn concentration (full period):**

| Tier | Total Churned Logos | Total Churned $ | % of All Churned $ | % of All Churned Logos | Avg $ per Churn |
|------|--------------------|-----------------|--------------------|----------------------|----------------|

4. **Churn reasons by tier:**

| Tier | Top Reason | Count | ARR Impact |
|------|------------|-------|-----------|

If any table cannot be populated, log as CRITICAL data gap. The churn concentration table is the most important: it reveals whether churn dollars are distributed evenly across tiers or concentrated in one segment. This directly impacts segment recommendations.

**If Account-level retention fields don't exist:** Log as CRITICAL data gap: "Cannot calculate retention by segment. Account object lacks churn date, status, or ARR fields. This prevents validating any segment-level recommendation." Ask the user if this data exists elsewhere (e.g., a separate churn tracking system, a BI tool, or a finance template).

**HubSpot:** Use HubSpot company object with lifecycle stage and deal associations. Calculate retention from company stage transitions (Customer → Former Customer). Revenue data may require summing associated deal values.

**Report:** "Retention by tier: T1 GDR {X}%, T2 GDR {Y}%, T3 GDR {Z}%. {N} accounts churned in period, ${ARR} total ARR lost. Churn concentration: T1 = {X}% of churned $, T2 = {Y}%, T3 = {Z}%."

### Agent 6: Buyer Persona Identification (if --personas enabled)

**Runs in parallel with Agents 1-5.** Depends on Agent 1's deal list for Opportunity IDs, so starts after Agent 1 returns (or runs in a second wave after Agent 1).

**Purpose:** Identify the primary buyer persona for each deal by enriching contacts via Prospeo and classifying by title/seniority/department.

**Step 6a: Query CRM contacts for each deal**

**Salesforce:**

```bash
sf data query --query "
SELECT OpportunityId, ContactId, Role, IsPrimary,
  Contact.FirstName, Contact.LastName, Contact.Title, Contact.Department,
  Contact.LinkedIn_URL__c, Contact.Email
FROM OpportunityContactRole
WHERE OpportunityId IN ('{OPP_ID_1}', '{OPP_ID_2}', ...)
ORDER BY IsPrimary DESC, Role ASC
" --target-org {config.crm.sf_org} --json
```

If OpportunityContactRole has no records, fall back to querying Contacts associated with the Account:

```bash
sf data query --query "
SELECT Id, FirstName, LastName, Title, Department, Email, LinkedIn_URL__c
FROM Contact
WHERE AccountId IN (SELECT AccountId FROM Opportunity WHERE Id IN ('{OPP_ID_1}', ...))
AND Title != null
" --target-org {config.crm.sf_org} --json
```

**HubSpot:**

Use HubSpot MCP to get associated contacts for each deal:
1. For each deal ID, query associations: `deals/{dealId}/associations/contacts`
2. For each associated contact, get properties: `jobtitle`, `hs_linkedinbio`, `department`, `firstname`, `lastname`, `email`

**Step 6b: Enrich via Prospeo (if --skip-prospeo is not set)**

For each contact, attempt enrichment in priority order:

1. **LinkedIn URL available in CRM:** Use it directly:

```bash
source ~/.env && curl -s -X POST https://api.prospeo.io/enrich-person \
  -H "X-KEY: $PROSPEO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "data": {
      "linkedin_url": "{contact_linkedin_url}"
    }
  }'
```

2. **No LinkedIn URL but has name + company:** Use name-based matching:

```bash
source ~/.env && curl -s -X POST https://api.prospeo.io/enrich-person \
  -H "X-KEY: $PROSPEO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "data": {
      "first_name": "{contact_first_name}",
      "last_name": "{contact_last_name}",
      "company_name": "{account_name}"
    }
  }'
```

3. **Neither available:** Fall back to CRM Title field only.

**Batch strategy:** Use the bulk endpoint for efficiency when >10 contacts:

```bash
source ~/.env && curl -s -X POST https://api.prospeo.io/bulk-enrich-person \
  -H "X-KEY: $PROSPEO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "data": [
      {"linkedin_url": "{url_1}"},
      {"linkedin_url": "{url_2}"},
      ...up to 50 per batch
    ]
  }'
```

**Rate limiting:** Prospeo allows 150 requests/minute. Add 500ms delay between batch requests. Handle 429 with exponential backoff (1s, 2s, 4s, max 3 retries).

**Credit budget:** Before starting enrichment, calculate expected credit usage: `{contact_count} * 1 credit`. If >500 credits, warn: "Prospeo enrichment will use ~{N} credits (${cost} at current plan). Proceed?" Allow user to set a credit cap.

**Step 6c: Classify contacts into persona framework**

For each contact, use the enriched data (or CRM fallback) to assign a persona.

**Classification input priority:**
1. Prospeo `person.current_job_title` + `person.job_history[0].seniority` + `person.job_history[0].departments` (highest confidence)
2. CRM `Contact.Title` + `Contact.Department` (moderate confidence)
3. Gong/Fathom call participant names matched to contacts (low confidence, title unknown)

**Persona taxonomy (standard B2B SaaS buyer framework):**

| Persona | Title patterns (case-insensitive) | Seniority | Departments |
|---------|----------------------------------|-----------|-------------|
| Economic Buyer | CEO, CFO, VP Finance, COO, Owner, Founder, President, Managing Director | C-Suite, VP | Finance, Executive, Operations |
| Champion / User | Manager, Director (non-IT/Finance), Head of, Team Lead, Operations Manager, Account Manager | Manager, Director | Operations, Marketing, Sales, Customer Success, Account Management |
| Technical Evaluator | CTO, VP Engineering, IT Director, Solutions Architect, DevOps, Engineering Manager, Technical Lead | Any | Engineering, IT, Technology, Product |
| Executive Sponsor | CMO, CRO, VP Sales, VP Marketing, VP Customer Success, General Manager | C-Suite, VP | Marketing, Sales, Customer Success, Revenue |

**Classification rules:**
1. Match against title patterns first (most specific)
2. If no pattern match, use Prospeo `seniority` + `departments` fields
3. If multiple contacts per deal, identify the **primary persona** (highest seniority or `IsPrimary=true` from OpportunityContactRole) and **secondary personas**
4. If no contact data at all for a deal, mark as `persona_unknown`

**Auto-discovery layer:** After framework classification, also capture the raw title clusters. Group all unique titles into 8-15 clusters using string similarity (e.g., all "Director of Operations" / "VP Operations" / "Head of Ops" cluster together). Report both the framework mapping AND the raw clusters so the user sees what titles actually appear in their data.

**Step 6d: Output**

For each deal, return:
- `primary_persona`: Economic Buyer | Champion/User | Technical Evaluator | Executive Sponsor | Unknown
- `primary_persona_title`: the actual job title (e.g., "VP of Finance")
- `primary_persona_seniority`: C-Suite | VP | Director | Manager | Individual Contributor
- `primary_persona_department`: the department/function
- `secondary_personas`: array of other contacts and their personas
- `persona_confidence`: high (Prospeo enriched) | moderate (CRM title) | low (inferred from call participants) | none
- `enrichment_source`: prospeo | crm_title | call_participant | none

**Report:** "Identified buyer personas for {X} of {Y} deals. {A} via Prospeo enrichment, {B} from CRM title fields, {C} from call participants. {D} deals have no persona data. Used {N} Prospeo credits."

---

## Step 3: Data Linking

After all agents return, link the data together.

### 3a: Deal-to-Call Linking

**Gong:** Use the mapping built in Step 2b, regardless of whether it came from the SF managed package or REST API. The mapping format is the same: `{ sfOpportunityId: [gongCallId, ...] }`. Match Opportunity IDs to the deal list from Agent 1.

If no mapping was built (both sources empty), fall back to email-domain matching: extract external participant emails from Gong call metadata and match against SF Contact emails associated with Opportunities. Flag these as lower confidence.

**Fathom:** Match by attendee email domain or company name (fuzzy). Flag matches as lower confidence.

**Manual:** User specified which deal each transcript relates to.

### 3b: Deal-to-Interview Linking

Match interview evidence to deals using case-insensitive substring matching on company/account names. If no match, the evidence still contributes to thematic analysis but isn't linked to a specific deal.

### 3c: Deal-to-Enrichment Linking (if --enrich)

Match Revelio data to deals via the account name-to-RCID mapping from Agent 4.

### 3d: Build Unified Records

For each deal, create a unified record containing:
- All CRM fields
- Array of linked call transcripts + summaries
- Array of linked interview evidence
- Revelio enrichment data (if available)
- Persona data: primary persona, title, seniority, department, secondary personas, confidence (if --personas enabled)
- Evidence coverage flag: `crm_only`, `crm+calls`, `crm+interviews`, `crm+calls+interviews`, append `+persona` if persona data exists

**Report:** "Linked {X} deals to {Y} call transcripts, {Z} to interview evidence. {W} deals have evidence from all sources."

### 3e: Deal-to-Persona Linking (if --personas enabled)

Attach Agent 6's persona data to each deal's unified record. Each deal gets:
- `primary_persona`: the framework classification
- `primary_persona_title`: raw title
- `primary_persona_seniority`: seniority level
- `primary_persona_department`: department/function
- `secondary_personas`: array
- `persona_confidence`: high/moderate/low/none

**Cross-validation with call transcripts:** If a deal has both persona data (from CRM/Prospeo) AND call transcripts, check if the identified persona's name appears in the call attendees. This confirms the persona was actively involved in the evaluation (not just a CRM record). Flag deals where the persona was on the CRM record but never appeared on a call as "Persona identified but engagement unconfirmed."

---

## Step 4: Analysis Modules

Run these modules sequentially. Each builds on prior outputs.

**Throughout all modules:** Track data gaps. Every time a module encounters a missing field, sparse data, or insufficient sample, log it as a gap entry for Step 6.

### Module 4A: Temporal Foundation

**Purpose:** Establish the time dimension before any segmentation.

1. Group all deals by quarter (or month if <6 months of data)
2. Calculate overall win rate per period (Won / (Won + Lost), exclude DQ and Nurture)
3. Identify inflection points: any quarter where win rate changed >10 percentage points from prior quarter
4. Calculate additional per-period metrics: total revenue won, total revenue lost, average deal size, DQ rate
5. **For the most recent quarter**: show monthly breakdown (deals, win rate, bookings) to surface within-quarter momentum

**Report:**

> **Timeline Overview (BOTH count and dollar views required):**
> | Quarter | Deals | Won | Lost | DQ | Win Rate | Revenue Won | Revenue Lost | Net |
> |---------|-------|-----|------|----|----------|-------------|--------------|-----|
> | {quarter} | {N} | {won} | {lost} | {dq} | {rate}% | ${X} | ${Y} | ${Won-Lost} |
>
> **Most Recent Quarter Monthly Breakdown:**
> | Month | Deals | Won | Lost | Win Rate | Bookings | Trend vs Prior Month |
> |-------|-------|-----|------|----------|----------|---------------------|
>
> {If inflection point:} **Something changed in {quarter}** -- win rate shifted from {X}% to {Y}%. We'll investigate what drove this in the segment analysis.

### Module 4B: Segmented Analysis

For each requested segment dimension (`--segment` parameter), calculate per-segment metrics.

**ACV Tier segmentation:**
- Use `deal_tier` field first if it exists and is populated
- If not, auto-detect boundaries from the deal amount/ARR distribution:
  - Calculate 25th, 50th, 75th percentiles
  - Tier 1 (Enterprise): top quartile
  - Tier 2 (Mid-Market): middle 50%
  - Tier 3 (SMB): bottom quartile
  - Report the boundaries found

**Other segments:** Use the mapped field directly:
- Vertical/Industry: group by `vertical_or_industry` values
- Source: group by `deal_source`
- Competitor: group by `competitor` (which competitor was the deal lost to)
- Owner: group by `owner`
- Lifecycle: group by `deal_type` combined with stage (New Business Won, New Business Lost, Existing Business Won, Existing Business Lost/Churn, DQ, Nurture)

**Per-segment metrics:**

| Metric | Calculation |
|--------|-------------|
| Deal count | Total, Won, Lost |
| Win rate (overall) | Won / (Won + Lost) for entire lookback |
| Win rate (last 2 quarters) | Same calculation, recent data only |
| Win rate trend | Improving (recent > overall by >5pp), Declining (recent < overall by >5pp), or Stable |
| Revenue won | Sum of ARR/amount for won deals |
| Revenue lost | Sum of ARR/amount for lost deals |
| **Net revenue impact** | Revenue won minus revenue lost (a segment can have a high WR but negative net if it loses big deals) |
| Avg deal size (won vs. lost) | Mean ARR/amount by outcome |
| Top loss reasons | Distribution of loss_reason values within this segment |
| Recency flag | If last 2 quarters diverge from trailing average by >10pp, flag it |

**MANDATORY dual-axis views:** For pipeline and bookings analysis, ALWAYS produce BOTH:
1. **Opps count view** (deals by segment, source, etc.)
2. **Dollar value view** ($ by segment, source, etc.)

These often tell dramatically different stories. Example: Enterprise may represent 5% of opps but 40% of pipeline dollars. Present both views side by side or as companion tables. If you only show one, you are hiding the other story.

**Flag any segment where:**
- Win rate is >10pp above or below the overall average
- Revenue lost exceeds revenue won
- The trend direction has reversed recently

**Cross-segment analysis (required):**
For each primary segment dimension, cross-tabulate with at least one other dimension:
- Tier x Source: win rate for each tier within each source (e.g., "Partner channel: Tier 3 wins at 66.7%, Tier 1-2 wins at 27.3%")
- Tier x Competitor: which competitors you face at each tier
- Source x Competitor: which competitors show up in which channels
- Source secondary breakdown: if a source has sub-types (e.g., Partner has Agency vs. Tech partners in Deal_Source_Secondary__c), always break it down. The sub-segment is often more actionable than the top-level segment.

**Outlier handling (required):**
For each segment with <10 deals, check if a single deal represents >30% of the segment's revenue. If so, report both the with-outlier and without-outlier metrics. Lead with the deal count story, not the revenue story.

### Module 4C: Loss Reason Analysis

**If loss_reason field exists and is structured (picklist):**
1. Group lost deals by loss reason
2. Rank by frequency AND by revenue impact
3. For each top-10 reason:
   - Count of deals lost
   - Total ARR/revenue lost
   - Which segments are most affected
   - Trend: compare reason's share in the last 2 quarters vs. trailing average
   - When it emerged: flag any reason that was <10% of losses 6+ months ago but >25% recently (emerging threat)

**If loss_reason is free text:**
1. Read all unique loss reason values
2. Cluster similar reasons into 8-12 categories (use judgment to group synonyms: "too expensive" / "pricing" / "budget constraints" all map to "Price")
3. Present proposed categories for confirmation
4. Then run the same ranked analysis as structured data

**If loss_reason field doesn't exist:**
Log a Critical data gap: "No loss reason field. Cannot determine why deals are lost from CRM data. Relying on call transcripts and interviews only. Recommend adding a 'Closed Lost Reason' picklist to your CRM."

**Persona breakdown (if --personas enabled):**

For each top-5 loss reason, show which persona was the primary buyer:

| Loss Reason | Economic Buyer | Champion/User | Technical Evaluator | Total |
|-------------|---------------|--------------|--------------------|----|
| {reason} | {N} deals (${X}) | {N} deals (${X}) | {N} deals (${X}) | {N} |

Flag any loss reason that is concentrated in one persona (>60% of deals). Example: "Price Sensitivity" losses are 80% Economic Buyer, suggesting pricing objections come from finance, not from the user champion. This changes the mitigation strategy (ROI case vs. feature value).

### Module 4D: Competitive Intelligence

**If feature intel was run in Step 0.5:** Read `{portfolio_path}/knowledge/feature-comparison-*.md` before writing any competitive analysis. Use the radar scores, unique advantages, and critical gaps to ground claims. Specifically:
- Where call/interview evidence says "competitor X is stronger on analytics" — cross-check against the feature intel radar score. Is the gap real and quantified, or is it perception?
- Where CRM identifies a competitor as a deal-loss driver — check whether the feature intel identifies a specific differentiator (e.g., BigQuery integration, native loyalty engine) vs. broad category weakness. The specific gap is more actionable.
- Flag any feature gap claimed from CRM/calls that the feature intel shows the portfolio company already has (false gap — exists but isn't marketed or discovered in demos).

**From CRM data:**
- Group lost deals by `competitor` field
- Rank competitors by: (1) frequency, (2) ARR lost to them, (3) segments where they appear
- Calculate your win rate when each competitor is present vs. absent
- Identify incumbent providers you successfully displace vs. fail to displace (from `current_provider` field)

**From transcript intelligence evidence:**
Use the `extracted_signals` table first. Filter for `signal_type = 'competitor_mention'` and join to deals so competitor claims can be segmented by outcome, tier, source, persona, stage, and ACV band.

If `--skip-transcript-intel` was set, fall back to raw transcript search. For each competitor identified in CRM data, search call transcripts for mentions. Extract:
- Direct quotes about the competitor (positive and negative)
- How your reps position against them
- What prospects say they like about the competitor
- What prospects say your product does better

**From interview evidence:**
Cross-reference competitor mentions from knowledge files and interview transcripts.

**Competitor trend analysis:**
- Compare each competitor's share of losses in recent 2 quarters vs. trailing average
- Flag emerging threats: competitors appearing in >25% of recent losses but <10% historically
- Flag fading competitors: declining by >50% across the lookback period

**Repeatability check per competitor (required):**
For each competitor, before making claims about displacement difficulty:
- Show displacement rate by tier (don't aggregate across tiers)
- Show displacement rate by source channel
- Flag if the sample is driven by 1-2 large deals (revenue concentration check)
- Separate "deals where the prospect was actively leaving" from "deals where we tried to displace them" (if distinguishable from CRM fields)
- If total deals against a competitor is <5, label as "Insufficient sample, not actionable"

**Output per competitor:**
- Deals lost: {N} (${ARR} lost)
- Segments most affected: {tier, vertical}
- Displacement rate by tier: Tier 1: X%, Tier 2: Y%, Tier 3: Z%
- Trend: growing / stable / fading
- Your win rate against them: {X}% (with outlier check if N<10)
- Their advantage (from calls/interviews): "{quote}"
- Your advantage (from calls/interviews): "{quote}"
- Repeatability: Is this a persistent pattern (5+ deals, 2+ quarters) or an emerging signal?

**Persona breakdown (if --personas enabled):**

For each top-3 competitor, show which persona drives the competitive loss:

| Competitor | Persona Where We Lose | Win Rate by Persona | Their Advantage for This Persona |
|-----------|----------------------|--------------------|---------------------------------|
| {competitor} | {persona, % of losses} | EB: {X}%, Tech: {Y}% | {specific advantage that persona cares about} |

This reveals whether competitive losses are driven by the same persona across competitors or different ones. If Technical Evaluators drive losses to Competitor A but Economic Buyers drive losses to Competitor B, the competitive response is different for each.

### Module 4E: Deal Velocity Analysis

**Skip if no `stage_date_fields` exist in field_mapping.** Log as a High-priority data gap.

Calculate days between consecutive stage date timestamps for each deal. Use median (not mean) to reduce outlier impact.

**Stage transitions to measure (adapt to whatever stage_date_fields exist):**
1. CreatedDate to first stage date
2. Each consecutive stage pair
3. Last stage date to close (Closed Won or Closed Lost)
4. Total: CreatedDate to final outcome

**Velocity output:**

| Stage Transition | Won (median days) | Lost (median days) | DQ (median days) | Delta (Won vs Lost) |
|-----------------|-------------------|--------------------|--------------------|---------------------|
| {transition} | {X} | {Y} | {Z} | {+/- N days} |
| Total Cycle | {X} | {Y} | {Z} | {+/- N days} |

**Stall point identification:** For lost deals, find the stage with the longest median dwell time. This is where deals die. Cross-reference with Gong transcripts from calls during that stage.

**Velocity cuts (required, not optional):**
All velocity metrics must be segmented. Never present a single company-wide median without the breakdown.
- Won vs. Lost vs. DQ
- By ACV tier (required)
- By competitor (if data exists)
- By deal source (if data exists)
- By deal size band (if pricing ceiling exists, show velocity above and below it)

### Module 4F: Pricing Analysis

**From CRM deal amounts:**
1. Calculate win rate by price band (split deals into quintiles or deciles by amount/ARR)
2. Identify pricing sweet spot: the band with the highest win rate weighted by volume
3. Identify pricing ceiling: the price point above which win rate drops sharply (>15pp decline)
4. Compare average deal size for won vs. lost deals (are we winning smaller deals and losing bigger ones?)

**From contract type (if field exists):**
- Win rate by contract structure (annual vs. monthly, etc.)
- Average deal size by contract type

**From transcript intelligence evidence:**
Use `extracted_signals` first. Filter for `signal_type IN ('pricing_objection', 'roi_value_driver', 'buying_criteria')` and segment by tier, amount band, persona, outcome, and source.

If `--skip-transcript-intel` was set, fall back to raw transcript search for pricing-related terms: "price", "cost", "budget", "expensive", "cheaper", "discount", "ROI", "value". Extract relevant quotes and categorize sentiment (price seen as fair, high, or low).

**Pricing by segment (required):**
- Show win rate by price band within each tier (the ceiling may differ by tier)
- Show pricing sweet spot separately for new business vs. existing business
- If a price band has <5 deals, merge with adjacent band rather than drawing conclusions from thin data

**Output:**
- Pricing sweet spot by tier: Tier 3: ${X}-${Y} ({Z}% win rate, {N} deals). Tier 2: ${X}-${Y}. Tier 1: ${X}-${Y}.
- Pricing ceiling by tier (the point where win rate drops >15pp)
- Contract preference: {type} wins at {X}% vs. {Y}% for other types
- Prospect sentiment on pricing: {summary with quotes}
- Outlier check: does a single large deal distort any price band?

**Persona breakdown (if --personas enabled):**

Show pricing sensitivity by buyer persona:

| Persona | Median Deal Size (Won) | Median Deal Size (Lost) | Price Ceiling | Win Rate Above Ceiling |
|---------|----------------------|------------------------|--------------|----------------------|
| Economic Buyer | ${X} | ${X} | ${X} | {Y}% |
| Champion/User | ${X} | ${X} | ${X} | {Y}% |

Flag if Economic Buyers have a different price ceiling than other personas (they usually do). If the Economic Buyer's ceiling is lower than the Champion/User's ceiling, it suggests the person evaluating value is different from the person controlling budget.

### Module 4G: Feature Gap Analysis

**From CRM (product_gap field, if exists):**
- Group lost deals by product gap reason
- Rank by: (1) count of deals affected, (2) total ARR lost
- Show which segments and competitors are associated with each gap

**From transcript intelligence evidence:**
- Use `extracted_signals` first. Filter for `signal_type IN ('feature_request', 'feature_gap', 'integration_requirement', 'buying_criteria')`
- Segment feature evidence by outcome, tier, competitor, persona, source, and ACV band
- Categorize into product gap themes
- Match to CRM product_gap categories where possible

If `--skip-transcript-intel` was set, fall back to raw transcript search for feature requests, "I wish it could...", "we need...", "does it support..." mentions.

**From interview evidence:**
- Pull product gap mentions from knowledge files and churn interviews
- Add interview quotes to each gap

**Revenue quantification:**
For each top-5 gap:
- Deals lost due to this gap: {N}
- ARR lost: ${X}
- Evidence strength: Strong / Moderate / Emerging
- Evidence block: SF data + Gong quote + Interview quote

**Roadmap cross-reference:**
If `{portfolio_path}/knowledge/product.md` exists, read it and note which gaps are on the roadmap vs. not. Flag gaps with high revenue impact that are NOT on the roadmap.

**Persona breakdown (if --personas enabled):**

For each top-5 feature gap, show which persona cited it:

| Feature Gap | Economic Buyer | Champion/User | Technical Evaluator | ARR Lost | On Roadmap? |
|-------------|---------------|--------------|--------------------|---------|----|
| {gap} | {N} deals (${X}) | {N} deals (${X}) | {N} deals (${X}) | ${total} | {Y/N} |

This reveals whose requirements are driving product gaps. A gap cited by Champions (daily users) has different urgency than one cited by Economic Buyers (who may be repeating what they heard from the champion vs. having a personal opinion).

**Prioritization signal:** Feature gaps cited by multiple personas across multiple tiers are the highest priority. Gaps cited by only one persona in one tier are niche.

**Feature intel cross-reference (if Step 0.5 ran):**
For each gap identified from CRM/calls, check the feature intel taxonomy before reporting it:
- **Confirmed real gap:** Feature is `missing` or `implied` in the scraped taxonomy → report with full evidence
- **False gap (exists but isn't discovered):** Feature shows `explicit` in help center crawl but wasn't mentioned in demos or marketing pages → report as "exists, not marketed" rather than a product gap. This is a sales/positioning fix, not a product fix.
- **Depth gap:** Feature is `explicit` but with only 1-2 thin help articles vs. competitor's 10+ → report as "feature exists but is shallow vs. competitor's mature implementation"

This prevents the analytical mistake of recommending product investment in features that already ship.

### Module 4H: Predictive Analysis (XGBoost)

**Skip if `--skip-ml` is set.** Note in data gaps.

**Step A: Prepare the feature matrix**

Convert each deal into a feature row. Use all available numeric and categorical fields:

| Feature | Source | Encoding |
|---------|--------|----------|
| estimated_arr | CRM | Numeric (log-transform if skewed) |
| acv_tier | CRM | One-hot encode |
| industry | CRM | One-hot top 5, collapse rest to "Other" |
| deal_source | CRM | One-hot encode |
| contract_type | CRM | One-hot encode |
| current_provider | CRM | One-hot encode |
| employee_count | CRM or Revelio | Numeric |
| deal_type | CRM | One-hot encode |
| total_cycle_days | Calculated | Numeric |
| gong_call_count | CRM | Numeric |
| has_competitor | CRM | Binary |
| has_gong_calls | Linked data | Binary |
| revelio_headcount | Enrichment | Numeric (if --enrich) |
| revelio_growth_1y | Enrichment | Numeric (if --enrich) |
| revelio_is_saas | Enrichment | Binary (if --enrich) |
| company_age_years | Enrichment | Numeric (if --enrich) |

Target variable: `is_won` (1 = Won, 0 = Lost). Exclude DQ and Nurture.

**Step B: Correlations**

```python
import pandas as pd
import numpy as np

numeric_cols = [all numeric features from matrix]
correlations = {}
for col in numeric_cols:
    valid = df[[col, 'is_won']].dropna()
    if len(valid) > 10:
        corr = valid[col].corr(valid['is_won'])
        correlations[col] = round(corr, 3)

sorted_corr = sorted(correlations.items(), key=lambda x: abs(x[1]), reverse=True)
```

Interpret: |r| > 0.3 = meaningful, |r| > 0.5 = strong, |r| < 0.1 = negligible.

**Step C: XGBoost**

```python
import xgboost as xgb
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import LabelEncoder
from sklearn.metrics import accuracy_score, roc_auc_score

X = df[feature_cols].copy()
for col in X.select_dtypes(include='object').columns:
    X[col] = LabelEncoder().fit_transform(X[col].astype(str))
X = X.fillna(-1)
y = df['is_won'].astype(int)

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42, stratify=y)

model = xgb.XGBClassifier(
    n_estimators=100, max_depth=4, learning_rate=0.1,
    random_state=42, verbosity=0, eval_metric='logloss'
)
model.fit(X_train, y_train)

y_pred = model.predict(X_test)
y_proba = model.predict_proba(X_test)[:, 1]
accuracy = accuracy_score(y_test, y_pred)
auc = roc_auc_score(y_test, y_proba)

importance_df = pd.DataFrame({
    'feature': feature_cols,
    'importance': model.feature_importances_
}).sort_values('importance', ascending=False)
```

**Step D: Interpret**

Report top 10 features. For each:
- Feature name and importance score
- Direction (higher = more likely to win or lose?)
- Does it align with segment analysis findings?
- If surprising (high importance but not in segment findings), flag as "hidden driver"

**Model quality:**
- AUC < 0.6: "Weak predictive power. Structured fields don't strongly predict outcomes. Qualitative factors likely dominate."
- AUC 0.6-0.75: "Moderate predictive power. Feature rankings are directional."
- AUC > 0.75: "Strong predictive power. Feature importance rankings are reliable."

**Interaction effects (if >100 deals):**
Check if deal source effectiveness varies by tier, or if competitor impact varies by vertical. Flag any interaction where a variable's win rate impact differs by >15pp across segments.

### Module 4I: ICP Synthesis

Synthesize all prior modules into an Ideal Customer Profile.

**Descriptive ICP ("who your best customer IS today"):**

*Firmographic profile* (from segment win rates + XGBoost + enrichment):
- Optimal ACV tier: highest win rate, weighted by volume (80% win rate on 5 deals is less reliable than 60% on 50 deals)
- Best-fit verticals: top 2-3 by recency-weighted win rate
- Company size sweet spot: employee/subscriber count range where win rate peaks
- Revenue range: P25, median, P75 of won deal amounts
- Include "last 2 quarters" vs. "overall" for each, flag divergence

*Behavioral signals* (from deal patterns):
- Best deal sources (highest recent win rates)
- Contract preferences (which type wins most)
- Provider displacement targets (which incumbent providers you successfully displace)
- Engagement level (Gong call count correlation with winning, if available)

*Competitive positioning*:
- Who you beat consistently
- Who beats you consistently
- Top 3 differentiators (from call transcripts)
- Top 3 weaknesses (from call transcripts + loss reasons)

**Prescriptive ICP ("who your best customer SHOULD BE"):**

*Under-invested winners:*
Segments with high win rate (>average + 10pp) but low pipeline volume (<20% of total deals). Calculate: "If you doubled pipeline in this segment, at the current {X}% win rate, that's ${Y} additional revenue per quarter."

*Emerging segments:*
Small sample (<15 deals) but strong recent performance (win rate >60% in last 2 quarters). Flag as "worth monitoring, too early to bet on."

*Fixable losers:*
Segments with addressable loss reasons (specific product feature, pricing, sales process). Quantify: "${X} in lost revenue from {N} deals, addressable by {specific action}."

*Abandon candidates:*
Segments with consistently low win rates (<25% recent) and structural disadvantages. Calculate: "You spent resources on {N} deals in this segment and won {M}. Reallocating that effort to {winning segment} would yield ${X} more."

**Anti-ICP** (who to avoid):
- Segments with <25% recent win rate
- Deal characteristics that predict losing (from XGBoost)
- Customer profiles from churn interviews

**Velocity benchmarks:**
- Median cycle time for ICP deals vs. non-ICP
- Expected time per stage for ICP
- Red flag: if a deal exceeds 75th percentile dwell at any stage, it's at risk

**Repeatability filter for all recommendations:**
Before recommending investment in any segment:
1. Confirm the pattern holds across sub-segments (e.g., Partner channel works because of agency partners specifically, not all partners)
2. Confirm the pattern is not driven by a one-time event (e.g., a platform shutting down, a migration wave that won't recur)
3. Report how many unique deals support the recommendation (minimum 10 for a strong recommendation, 5-9 for a directional recommendation, <5 is "insufficient data")
4. If a recommendation depends on a specific sub-segment, say so explicitly: "Invest in agency partner referrals for Tier 3 deals" not "Invest in the Partner channel"
5. For every recommendation, ask: "Would this pattern hold next quarter if the conditions that created it didn't recur?"

**Comparison with existing ICP:**
If `{portfolio_path}/knowledge/icp.md` exists, compare:
- What the data confirms
- What the data contradicts
- What's new (patterns not in the current ICP)

### Module 4J: Feature-Persona Intelligence (if --personas enabled)

**Skip if `--personas` is false.** Note in data gaps if skipped.

**Purpose:** Map which features and capabilities matter most to each buyer persona, for both won and lost deals. Output serves product teams (what to build for whom) and marketing teams (what to say to whom).

#### 4J-1: Feature Taxonomy Construction

**If /feature-intel taxonomy exists** (`{portfolio_path}/knowledge/feature-comparison-*.md`):

Read the cached taxonomy. Extract the hierarchical feature list (categories > specific features). This becomes the canonical list against which all feature mentions are mapped.

Report: "Feature taxonomy: from /feature-intel (high confidence). {N} categories, {M} specific features."

**If no /feature-intel taxonomy:**

Auto-generate a taxonomy from available evidence:

1. **From CRM product_gap field:** Extract all unique values from `product_gap` field (won and lost deals). These are the product team's own feature categories.
2. **From call transcripts:** Extract feature mentions using these search patterns:
   - Positive: "we liked", "what sold us", "the reason we chose", "really impressed by", "key differentiator", "stood out", "what we needed"
   - Negative: "we need", "missing", "doesn't have", "wish it could", "deal breaker", "gap", "limitation"
   - Neutral: "does it support", "can it do", "how does it handle", "what about"
3. **From interview evidence:** Extract feature mentions from knowledge files.
4. **Cluster into hierarchy:** Group related mentions into 8-15 capability categories. For each category, list the specific features mentioned.

Present the auto-generated taxonomy for quick confirmation:

> I've identified {N} capability categories from your deal evidence:
>
> | Category | Specific Features | Mentions |
> |----------|-------------------|----------|
> | {category} | {feature1}, {feature2} | {N} |
>
> Does this look right? Any categories to add, merge, or rename?

Report: "Feature taxonomy: auto-generated from deal evidence (moderate confidence, consider running /feature-intel for a more complete picture). {N} categories, {M} specific features."

#### 4J-1.5: Buyer Title Discovery

**Purpose:** Before analyzing what features matter to each persona, show WHO is actually buying. This grounds the persona analysis in real titles from the data.

**Process:**
1. From Agent 6's output, collect all unique titles across all deals with contact data
2. Group similar titles into 8-15 clusters using string similarity (e.g., "Director of Operations" / "VP Operations" / "Head of Ops" cluster together)
3. Map each cluster to the persona framework
4. Show the distribution: how many deals does each title cluster appear in, and what's the win rate?

**Output table (include in BOTH Medium Dive and Deep Dive):**

| Title Cluster | Example Titles | Persona Mapping | Deals | Win Rate |
|--------------|---------------|----------------|-------|----------|
| {cluster} | {title1}, {title2}, ... | {Economic Buyer / Champion / etc.} | {N} | {X}% |

**Why this comes first:** The raw titles reveal whether the persona framework mapping is accurate. If "Director of Ecommerce" gets mapped to Champion/User but actually behaves like a Technical Evaluator (low win rate, Recharge losses), the mapping should be adjusted before running the rest of Module 4J.

**Validation step:** After producing the title clusters, check if any cluster has a win rate that diverges >15pp from its assigned persona's average. If so, flag it: "Title '{title}' is mapped to {persona} but wins at {X}% vs. the persona average of {Y}%. Consider reclassifying."

#### 4J-2: Win Driver Analysis by Persona

For each persona that appears in 5+ won deals, identify what features drove the win.

**From CRM won_reason field (if exists):**
- Group won_reason values by primary persona
- Rank by frequency within each persona

**From call transcripts (won deals):**
- For each won deal with persona data AND call transcripts:
  - Search for positive feature signals: "what sold them", "the reason we went with", "what stood out", "we really liked", "deciding factor", "why we chose"
  - Map each mention to the feature taxonomy
  - Tag with the persona who was on the call (or the deal's primary persona)

**From interview evidence:**
- Search knowledge files for win-related feature mentions
- Map to taxonomy and tag with persona where identifiable

**Output table:**

| Persona | Top Win Drivers | Mentions | Revenue Won | Evidence |
|---------|----------------|----------|-------------|----------|
| Economic Buyer | {feature_category}: {specific_features} | {N} deals | ${X} | {Strong/Moderate/Emerging} |
| Champion/User | {feature_category}: {specific_features} | {N} deals | ${X} | {Strong/Moderate/Emerging} |
| Technical Evaluator | {feature_category}: {specific_features} | {N} deals | ${X} | {Strong/Moderate/Emerging} |

**Per persona, report:**
- Top 3 features that drove wins (ranked by frequency, then revenue)
- Representative quote from a call or interview
- Whether this persona's win drivers differ from the overall company win drivers (divergence flag)

#### 4J-3: Gap Impact Analysis by Persona

For each persona that appears in 5+ lost deals, identify which feature gaps caused the loss.

**From CRM product_gap and loss_reason fields:**
- Group product_gap and loss_reason values by primary persona
- Rank by revenue lost within each persona

**From call transcripts (lost deals):**
- For each lost deal with persona data AND call transcripts:
  - Search for negative feature signals: "we needed", "missing", "doesn't support", "deal breaker", "went with [competitor] because", "gap"
  - Map each mention to the feature taxonomy
  - Tag with persona

**From interview evidence:**
- Pull feature gap mentions from churn interviews and knowledge files
- Map to taxonomy and tag with persona

**Output table:**

| Persona | Top Feature Gaps | Deals Lost | ARR Lost | Competitor Where Gap Matters | Evidence |
|---------|-----------------|------------|----------|----------------------------|----------|
| Economic Buyer | {gap}: {specifics} | {N} | ${X} | {competitor} | {Strong/Moderate/Emerging} |

**Per persona, report:**
- Top 3 feature gaps causing losses (ranked by ARR lost)
- Which competitor benefits from each gap
- Whether the gap is on the product roadmap (cross-reference with knowledge/product.md)
- Feature intel cross-reference (if Step 0.5 ran): is the gap confirmed real, false (exists but not marketed), or a depth gap?

#### 4J-4: Feature-Persona Importance Matrix

The canonical output artifact. Combines win drivers and gap analysis into a single matrix showing feature importance by persona.

**Construction:**

For each feature category in the taxonomy and each persona:
1. **Win signal count:** How many won deals with this persona mentioned this feature positively
2. **Loss signal count:** How many lost deals with this persona cited this feature as a gap
3. **Total revenue at stake:** Won ARR where this feature was a win driver + Lost ARR where this feature was a gap
4. **Importance score:** (win_signals + loss_signals) weighted by revenue, normalized 0-100

**Output matrix:**

| Feature Category | Economic Buyer | Champion/User | Technical Evaluator | Executive Sponsor | Total Revenue at Stake |
|-----------------|---------------|--------------|--------------------|--------------------|----------------------|
| {category_1} | {score} ({N} signals) | {score} ({N} signals) | {score} ({N} signals) | {score} ({N} signals) | ${X} |
| {category_2} | ... | ... | ... | ... | ... |

**Reading the matrix:**
- High score for one persona, low for others = persona-specific messaging opportunity
- High score across all personas = table-stakes feature (must have, lead with universally)
- High score only from loss signals = product gap (build it)
- High score only from win signals = current differentiator (protect and market it)

**Per-feature detail (for top 5 features by total revenue):**

For each, show:
- Which persona cares most (and by how much vs. others)
- Win driver quote: "[Call: {title}, {date}] {quote}"
- Gap quote: "[Call: {title}, {date}] {quote}"
- Product/marketing implication: one-sentence recommendation

#### 4J-5: Segment x Persona Cross-Tabs

Cross-tabulate persona data with existing segmentation dimensions to surface whether different segments have different buyer personas and feature priorities.

**Tier x Persona:**

| ACV Tier | Top Persona | Win Rate by Persona | Feature Priority Differences |
|----------|-------------|--------------------|-----------------------------|
| Tier 1 (Enterprise) | {persona, % of deals} | EB: {X}%, Tech: {Y}% | Enterprise EBs care about {X}; SMB EBs care about {Y} |
| Tier 2 (Mid-Market) | {persona, % of deals} | ... | ... |
| Tier 3 (SMB) | {persona, % of deals} | ... | ... |

**Source x Persona:**

| Source | Top Persona | Note |
|--------|-------------|------|
| {source} | {persona, % of deals} | {Does channel attract different buyer types?} |

**Competitor x Persona:**

| Competitor | Persona Where We Lose Most | Feature Gap That Persona Cares About |
|-----------|---------------------------|-------------------------------------|
| {competitor} | {persona} | {feature gap specific to that persona} |

**Flag divergences:** If enterprise Economic Buyers care about fundamentally different features than SMB Economic Buyers, flag this explicitly: "The Economic Buyer persona has different feature priorities by tier. Enterprise EBs prioritize {X}, SMB EBs prioritize {Y}. Messaging should be tier-specific, not persona-generic."

#### 4J-6: Questions Asked by Persona

**Purpose:** Surface what each persona asks about during the sales process. This directly informs messaging, demo scripts, and content strategy. The questions a persona asks reveal what they're evaluating, not just what they buy.

**Extraction from transcript intelligence evidence:**

Use `extracted_signals` first. Filter for `signal_type = 'buyer_question'`, `speaker IN ('prospect', 'customer')`, and deals with persona data.

For each question signal:
1. Keep only prospect/customer questions.
2. Tag with the deal's primary persona and, when available, the speaker's actual persona.
3. Categorize questions into themes using the feature taxonomy from 4J-1.
4. Join to outcome, segment, source, stage, and ACV band.

If `--skip-transcript-intel` was set, fall back to raw transcript extraction:
1. Extract all question sentences (ending in `?`, or starting with question patterns: "how does", "can you", "what about", "do you support", "what happens when", "is there", "how do we", "what's the", "does it").
2. Tag each question with the speaker (prospect vs. rep) — only keep prospect questions.
3. Tag with the deal's primary persona.
4. Categorize questions into themes using the feature taxonomy from 4J-1.

**Clustering:**
Group questions into 5-10 themes per persona. Example themes:
- **Migration/Switching:** "How long does migration take?", "What happens to our existing subscribers?"
- **Technical/Integration:** "Do you support BigQuery?", "How does the API handle X?"
- **ROI/Business Case:** "What churn reduction do your customers see?", "What's the typical payback period?"
- **Feature Depth:** "Can it do X?", "How does your analytics compare to Y?"
- **Pricing/Contract:** "What's the pricing structure?", "Is there a minimum commitment?"

**Output table:**

| Persona | Top Question Themes | Example Questions | Frequency (% of calls) |
|---------|--------------------|--------------------|----------------------|
| Economic Buyer | {theme1}, {theme2} | "{question1}", "{question2}" | {X}% |
| Champion/User | {theme1}, {theme2} | "{question1}", "{question2}" | {X}% |
| Technical Evaluator | {theme1}, {theme2} | "{question1}", "{question2}" | {X}% |

**Actionable output per persona:**
- Top 3 question themes (what they ask about most)
- "Must-answer" questions (questions that appear in >50% of calls for that persona)
- Questions that predict winning vs. losing (if a persona asks about X, win rate is higher/lower)

**Report placement:** Include in both Medium Dive (summary table) and Deep Dive (full question list with quotes).

---

## Step 5: Evidence Cross-Referencing

For every significant finding (appears in 3+ deals OR represents >15% of a segment):

1. **CRM evidence:** Cite specific deals and their relevant field values
2. **Call evidence:** Use `extracted_signals` and `deal_evidence` first to find relevant transcript quotes. Cite as: `[Call: "{title}", {date}, "{quote}"]`. If transcript intelligence was skipped, fall back to raw transcript search.
3. **Interview evidence:** Find relevant quotes from knowledge files. Cite as: `[Interview: {source_file}, "{quote}"]`

**Evidence strength rating:**
- **Strong**: All 3 sources confirm (CRM + Call + Interview)
- **Moderate**: 2 of 3 sources confirm
- **Emerging**: 1 source only, flag as "needs further investigation"

Tag every finding with its evidence strength.

---

## Step 5.5: Recommendation Validation

**This step is MANDATORY before writing any strategic recommendation.** Every finding that leads to an "invest in X" or "stop doing Y" recommendation must pass this checklist. If any item fails, either get the data or label the recommendation as **"Conditional on [missing data]."**

### Full-Funnel Check (REQUIRED for any segment recommendation)

Before recommending focus on ANY segment (tier, vertical, channel, customer profile):

- [ ] **Acquisition data present**: Win rate, deal count, and bookings $ for the recommended segment
- [ ] **Retention data present**: GDR, NDR, and logo retention for the recommended segment, broken out by tier. If not available by segment, flag as **Critical data gap** and label the recommendation as conditional. Do NOT present a segment recommendation without retention data and call it definitive.
- [ ] **Expansion data present**: Upsell/cross-sell rate for the recommended segment. If not available, note it.
- [ ] **LTV math checked**: Does the segment's retention rate support the acquisition cost? A 55% win rate on $6K ACV with 70% annual retention has a fundamentally different LTV profile than a 30% win rate on $30K ACV with 82% retention. Run the math explicitly. Show it.
- [ ] **Churn concentration checked**: Show what % of churned DOLLARS comes from the recommended segment vs. others. A segment with high win rates but disproportionate churn dollars may not be the right focus.

### Enforcement (MANDATORY)

Before writing the final report, verify EACH segment recommendation against this matrix. Copy this table into the report:

| Recommendation | Acquisition? | Retention? | Churn Concentration? | LTV Math? | Persona Data? | Status |
|---|---|---|---|---|---|---|
| [each segment rec] | Yes/No | Yes/No | Yes/No | Yes/No | Yes/No/N/A | Validated / Conditional |

If Persona Data is "No" for a recommendation that claims persona-specific messaging or product priority, label it:

> **Conditional Recommendation:** Persona-specific claims are based on CRM title data only / insufficient persona sample size. Run with Prospeo enrichment and confirm against transcript evidence before acting on persona-level recommendations.

If ANY segment recommendation has "No" for Retention or Churn Concentration, it MUST be labeled:

> **Conditional Recommendation:** This recommendation is based on acquisition data only. Retention by segment has not been validated. The recommendation may change if segment-level retention reveals different patterns (e.g., if SMB churns at 50% logo retention, the "focus on SMB" argument weakens despite high win rates).

Do NOT present conditional recommendations as definitive. This enforcement exists because segment recommendations are not credible without retention data such as logo retention, GDR, and NDR by segment.

### Dual-Axis Views (REQUIRED for any chart-based finding)

- [ ] Every pipeline chart shown in BOTH opps count AND dollar value. The stories often differ materially (e.g., enterprise may be small in opps but dominant in $).
- [ ] Every bookings view shown by segment in dollars, not just win rates
- [ ] Every source/channel view shown in both opps and dollars
- [ ] If only one view is shown, explicitly state why and what the other view would show

### Denominator Check (REQUIRED for any "this is working" claim)

- [ ] For any segment claimed as "working," show both wins AND losses (count + dollars)
- [ ] Calculate **net impact**: won $ minus lost $. A channel with 67% WR but negative net $ impact tells a different story.
- [ ] For onboarding/displacement claims (e.g., "Recharge displacement is working"), show total attempts (won + lost), not just wins. State the overall displacement win rate.
- [ ] For any "X accounts did Y" claim, state what fraction of the total that represents

### Trend Validation (REQUIRED for any "improving" or "declining" claim)

- [ ] Show **monthly breakdown within the most recent quarter** (not just quarterly snapshots)
- [ ] Compare Q-on-Q AND versus the same quarter prior year (if available)
- [ ] State explicitly: **"This is a trend (3+ quarters of consistent direction)"** vs. **"This is a single-quarter signal"** vs. **"This is within-quarter momentum (monthly trajectory)"**
- [ ] For the most recent quarter, state whether the monthly trajectory is accelerating, decelerating, or flat

### Current-State Verification (REQUIRED for any operational recommendation)

- [ ] For "hire X" recommendations: verify if the role is already filled (check team knowledge file, recent meeting notes, or ask the user)
- [ ] For "stop doing X" recommendations: verify the current resource allocation
- [ ] For "invest in X" recommendations: verify current spend level
- [ ] For "fix X" recommendations: verify whether the fix is already in progress

### Counter-Argument Test (REQUIRED for every recommendation)

- [ ] For each recommendation, explicitly state the strongest argument against it
- [ ] Example: For "focus on SMB" — "The counter-argument is that SMB retention is 70% (per Cohorts v3), meaning 30% of acquired SMB customers churn within 12 months. The LTV of a $6K SMB deal at 70% retention is $X, versus a $30K MM deal at 82% retention which has LTV of $Y."
- [ ] If the counter-argument is strong enough to change the recommendation, change it or add conditions

### Persona Validation (REQUIRED if --personas enabled)

- [ ] **Persona sample size:** For any persona-specific recommendation ("lead with X for Economic Buyers"), confirm the persona has 5+ deals. <5 deals = "Insufficient data for persona-specific recommendation."
- [ ] **Persona consistency across tiers:** Before recommending "Technical Evaluators care about X," check if this holds across tiers. Enterprise Technical Evaluators may care about different things than SMB ones.
- [ ] **Feature-persona matrix confidence:** For any feature cited in the matrix, confirm it has evidence from at least 2 sources (CRM + transcript, or CRM + interview). Single-source feature claims are labeled "Emerging signal."
- [ ] **Win driver vs. gap balance:** For each persona, confirm the analysis includes BOTH win drivers and gaps. If only gaps are available (no won_reason field, no won-deal transcripts), label the persona analysis as "Loss-side only. Win drivers unknown."

---

## Step 6: Data Gaps Analysis

Collect all gaps logged throughout the analysis. Deduplicate and prioritize.

**Priority logic:**
- **Critical**: Prevents answering a core strategic question (e.g., no loss reasons = can't explain losses)
- **High**: Significantly weakens a major analysis (e.g., no competitor field = competitive analysis relies only on call mentions)
- **Medium**: Limits a supplementary analysis (e.g., no stage dates = no velocity analysis)
- **Low**: Nice-to-have (e.g., XGBoost skipped = no predictive modeling)

**Data quality issues to surface:**
- Fields with >40% blank/"Other" values
- Stale data (e.g., Gong exports >3 months old)
- Sample size warnings per segment (<10 deals)
- Free-text fields that should be picklists
- Missing CRM fields that would significantly improve the analysis

**Output table:**

| Data Gap | What You Can't Know | How to Fix | Priority |
|----------|-------------------|------------|----------|
| {gap} | {what it prevents} | {specific action} | {Critical/High/Medium/Low} |

---

## Step 7: Generate Layered Report

Save to: `{portfolio_path}/deliverables/strategic-analysis-{YYYY-MM-DD}.md`

### Layer 1: Executive Summary (Shallow Dive)

A CEO reads this in 3 minutes and has the full picture.

```markdown
# Strategic Analysis: {Company Name}
**Period:** {start_date} to {end_date} | **Generated:** {today}
**Data:** {N} closed deals, {M} call transcripts, {P} interview sources

---

## Executive Summary

### Headline Metrics

| Metric | Value | By Segment | Note |
|--------|-------|------------|------|
| Win Rate | {X}% | New Biz: {Y}%, Existing: {Z}% | New biz is the real number |
| Revenue Won | ${X} | New Biz: ${Y}, Existing: ${Z} | |
| Revenue Lost | ${X} | Tier 3: ${Y}, Tier 1-2: ${Z} | |
| Avg Deal Size (Won) | ${X} median | Tier 3: ${Y}, Tier 1: ${Z} | Use median, not mean |
| Avg Cycle Time (Won) | {X} days median | Tier 3: {Y}d, Tier 1: {Z}d | |
| Biggest Competitor | {name} (${X} lost) | Tier breakdown: {details} | |

### Top 5 Findings

Each finding must specify the segment, sample size, and repeatability.

1. **{Finding}** {one sentence with segment, sample size, and a number}. *{Repeatable Pattern / Directional Signal / Insufficient Data}. Evidence: {Strong/Moderate/Emerging}*
2. **{Finding}** ...
3. **{Finding}** ...
4. **{Finding}** ...
5. **{Finding}** ...

### Your ICP Today
{2-3 sentence data-driven description: tier, vertical, size, source, win rate}

### Where You Should Be Investing
{2-3 sentences on the prescriptive ICP: under-invested segments, revenue upside, specific recommendation}

### Top Buyer Personas (if --personas enabled)

| Persona | % of Deals | Win Rate | Top Feature Priority | Key Insight |
|---------|-----------|----------|---------------------|-------------|
| {persona} | {X}% | {Y}% | {feature_category} | {one-line insight} |

**Messaging implication:** {one-sentence summary: "Lead with X for Economic Buyers, Y for Technical Evaluators"}

### What You Can't Know (Data Gaps)
1. {Gap}: {what it prevents, one line}
2. {Gap}: ...
3. {Gap}: ...

### Recommended Actions
1. **This week:** {highest-impact, lowest-effort action}
2. **This month:** {medium-effort change}
3. **This quarter:** {strategic shift}
```

### Layer 2: Medium Dive

A VP of Sales or Marketing reads this in 15 minutes and knows what to do.

```markdown
---

## Analysis Detail

### Win/Loss by Segment

#### By ACV Tier
| Tier | Deals | Win Rate (Overall) | Win Rate (Last 2Q) | Trend | Revenue Won | Revenue Lost | Avg Size |
|------|-------|--------------------|--------------------|----|-------------|--------------|----------|

#### By Source
| Source | Deals | Win Rate (Overall) | Win Rate (Last 2Q) | Trend | Revenue Won | Avg Size |
|--------|-------|--------------------|--------------------|----|-------------|----------|

#### By Vertical
| Vertical | Deals | Win Rate (Overall) | Win Rate (Last 2Q) | Trend | Revenue Won |
|----------|-------|--------------------|--------------------|----|-------------|

#### By Competitor
| Lost To | Deals Lost | ARR Lost | Your Win Rate Against | Trend | Worst Segments |
|---------|------------|----------|----------------------|-------|----------------|

### Loss Drivers (by Revenue Impact)

| Rank | Reason | Deals | ARR Lost | % of Losses | Trend | Evidence |
|------|--------|-------|----------|-------------|-------|----------|
| 1 | {reason} | {N} | ${X} | {Y}% | {growing/stable/fading} | {Strong/Moderate} |

### Deal Velocity

| Stage Transition | Won (median days) | Lost (median days) | Delta |
|-----------------|-------------------|--------------------|-------|

**Stall point:** Deals die at {stage}. Median dwell time for lost deals: {X} days vs. {Y} days for won deals.

### Pricing

- **Sweet spot:** ${X}-${Y} ({Z}% win rate)
- **Ceiling:** Above ${X}, win rate drops to {Y}%
- **What prospects say:** "{quote}" [Call: {title}, {date}]

### Feature Gaps (by Revenue Impact)

| Rank | Gap | Deals | ARR Lost | On Roadmap? | Evidence |
|------|-----|-------|----------|-------------|----------|

### Feature-Persona Matrix (if --personas enabled)

#### Buyer Persona Distribution
| Persona | Deals | Win Rate | Avg Deal Size | Top Win Driver | Top Gap |
|---------|-------|----------|---------------|---------------|---------|
| Economic Buyer | {N} | {X}% | ${Y} | {feature} | {gap} |
| Champion/User | {N} | {X}% | ${Y} | {feature} | {gap} |
| Technical Evaluator | {N} | {X}% | ${Y} | {feature} | {gap} |
| Executive Sponsor | {N} | {X}% | ${Y} | {feature} | {gap} |
| Unknown | {N} | {X}% | ${Y} | N/A | N/A |

#### Buyer Title Clusters
| Title Cluster | Example Titles | Persona | Deals | Win Rate |
|--------------|---------------|---------|-------|----------|
| {cluster} | {titles} | {persona} | {N} | {X}% |

#### Feature Importance by Persona
| Feature Category | Economic Buyer | Champion/User | Tech Evaluator | Exec Sponsor | Revenue at Stake |
|-----------------|---------------|--------------|---------------|-------------|-----------------|
| {category} | {score/100} | {score/100} | {score/100} | {score/100} | ${X} |

**Key Finding:** {The most actionable insight from the matrix, e.g., "Economic Buyers and Technical Evaluators have opposite feature priorities: EBs care about ROI reporting while TEs care about API depth. Your current messaging leads with API features, which resonates with TEs but not the person who signs the check."}

#### Persona-Specific Recommendations
**For Economic Buyers:** {What to lead with in messaging, what product investments matter}
**For Champions/Users:** {What to lead with, what UX investments matter}
**For Technical Evaluators:** {What to lead with, what integration/API investments matter}

#### Questions by Persona
| Persona | Top Question Themes | Must-Answer Questions | Frequency |
|---------|--------------------|-----------------------|-----------|
| {persona} | {themes} | "{question}" | {X}% of calls |

### Retention by Segment

| Tier | Logo Retention | GDR | Avg $ at Churn | % of Total Churned $ |
|------|---------------|-----|---------------|---------------------|

**Churn concentration:** [State which tier produces the most churned dollars relative to its share of churned logos. A tier with 25% of churned logos but 80% of churned dollars has concentrated risk.]

**Retention validates acquisition:** [Explicitly state whether the segment recommended for focus also has better retention. If it doesn't, flag the contradiction and reconsider the recommendation. Example: "SMB wins at 55% and retains at 90% GDR. Enterprise wins at 8% and retains at 82% GDR. The full-funnel case for SMB is validated."]

### Quarter-over-Quarter Trends

| Quarter | Won | Lost | DQ | Win Rate | Avg ARR Won | Top Loss Reason | Top Competitor |
|---------|-----|------|----|----------|-------------|-----------------|----------------|

### Data Gaps

| Gap | What You Can't Know | How to Fix | Priority |
|-----|---------------------|------------|----------|

### Methodology
- **CRM:** {type}, {N} records, queried {date}
- **Calls:** {type}, {N} transcripts + {M} summaries, {date range}
- **Transcript intelligence:** {Enabled: DuckDB evidence store with {chunk_count} chunks, {signal_count} extracted signals, scope {transcript_scope} | Skipped: raw transcript search only}
- **Interviews:** {N} files from {date range}
- **Enrichment:** {Revelio Labs: {N} accounts matched | Not enabled}
- **ML model:** {Accuracy {X}%, AUC {Y} | Skipped}
- **Lookback:** {days} days ({start} to {end})
```

### Layer 3: Deep Dive

An analyst or consultant reads this for the complete evidence. This layer must include a Pattern Reliability section.

```markdown
---

## Deep Dive

### Pattern Reliability

Rate every finding from the analysis. This is the analytical credibility check.

| Finding | Deals | Quarters | Outlier-Adjusted? | Repeatable? | Rating |
|---------|-------|----------|-------------------|-------------|--------|
| {finding} | {N} | {Q} | {Yes: metric holds without top deal / No / N/A} | {Yes/No} | {Repeatable Pattern / Directional Signal / Insufficient Data} |

**Rating criteria:**
- **Repeatable Pattern**: 5+ deals, persists across 2+ quarters, holds after removing outliers
- **Directional Signal**: 5+ deals but only 1 quarter of data, OR pattern weakens when outliers removed
- **Insufficient Data**: <5 deals, regardless of how strong the signal looks

### Predictive Model Results
{Only if XGBoost ran}

**Model performance:** Accuracy {X}%, AUC {Y} (on {N} test observations)

#### Top Predictors of Win vs. Loss
| Rank | Feature | Importance | Direction | Aligns with Segments? |
|------|---------|------------|-----------|----------------------|

#### Correlations
| Variable | Correlation | Interpretation |
|----------|-------------|----------------|

#### Interaction Effects
{Variables that behave differently across segments}

#### Hidden Drivers
{Features ranking high in XGBoost but not prominent in segment analysis}

### Ideal Customer Profile (Complete)

#### Firmographic Profile
| Dimension | Ideal Range | Win Rate | Sample Size | XGBoost Rank |
|-----------|-------------|----------|-------------|--------------|

#### Behavioral Signals
{Deal source, contract type, engagement patterns with specific win rates}

#### Anti-ICP
| Characteristic | Win Rate | Why (Evidence) |
|---------------|----------|----------------|

#### Velocity Benchmarks
| Metric | ICP Deals | Non-ICP Deals | Delta |
|--------|-----------|---------------|-------|

#### vs. Existing ICP
{What data confirms, contradicts, and adds vs. knowledge/icp.md}

### Segment Deep Dives

#### {Tier 1 / Enterprise}
{Full data table, top loss reasons, competitors, evidence block with direct quotes}

{Repeat for each tier, vertical, lifecycle type}

### Competitive Intelligence (Detailed)

#### {Competitor Name}
- **Deals lost:** {N} (${X} ARR)
- **Segments:** {where they win}
- **Their advantage:** {from evidence}
- **Your advantage:** {from evidence}
- **Call evidence:** "{quote}" [Call: {title}, {date}]
- **Interview evidence:** "{quote}" [Interview: {source}]

{Repeat for each major competitor}

### Feature-Persona Intelligence (Detailed)

#### Feature Taxonomy
**Source:** {/feature-intel (high confidence) | auto-generated (moderate confidence)}

| Category | Specific Features | Source |
|----------|-------------------|--------|
| {category} | {feature1}, {feature2}, ... | {CRM / Transcript / Interview / feature-intel} |

#### Persona Profiles

##### Economic Buyer
- **Titles observed:** {list of actual titles from data}
- **% of deals:** {X}% ({N} deals)
- **Win rate:** {X}% (vs. {Y}% overall)
- **Avg deal size:** ${X} (vs. ${Y} overall)
- **Win drivers:** {ranked list with evidence}
  - {feature}: {N} deals, ${X} won. Evidence: "[Call: {title}, {date}] {quote}"
- **Feature gaps:** {ranked list with evidence}
  - {gap}: {N} deals, ${X} lost. Evidence: "[Call: {title}, {date}] {quote}"
- **Pricing sensitivity:** Ceiling at ${X} ({Y}% win rate above)
- **Competitor exposure:** Loses most to {competitor} ({N} deals)

{Repeat for each persona with 5+ deals}

#### Raw Title Clusters

See Module 4J-1.5 for methodology.

| Cluster | Titles | Framework Mapping | Count |
|---------|--------|-------------------|-------|
| {cluster_name} | {title1}, {title2}, ... | {Economic Buyer / Champion / etc.} | {N} |

#### Full Feature-Persona Matrix (with evidence)
| Feature | Persona | Signal Type | Deals | Revenue | Quote | Source |
|---------|---------|------------|-------|---------|-------|--------|
| {feature} | {persona} | Win Driver / Gap | {N} | ${X} | "{quote}" | {CRM/Call/Interview} |

#### Enrichment Methodology
- **Prospeo enrichment:** {N} contacts enriched, {M} matched, {P} credits used
- **CRM title fallback:** {N} contacts classified from CRM Title field
- **Unknown personas:** {N} deals with no persona data
- **Persona confidence distribution:** High: {N}, Moderate: {N}, Low: {N}, None: {N}

#### Questions by Persona (Detailed)

| Persona | Theme | Question | Call | Outcome | Frequency |
|---------|-------|----------|------|---------|-----------|
| {persona} | {theme} | "{question}" | [Call: {title}, {date}] | Won/Lost | {X}% |

**Must-Answer Questions (appear in >50% of calls for persona):**
- Economic Buyer: "{question1}", "{question2}"
- Champion/User: "{question1}", "{question2}"
- Technical Evaluator: "{question1}", "{question2}"

**Questions That Predict Winning:**
| Question Theme | Win Rate When Asked | Win Rate When Not | Delta | Persona |
|---------------|--------------------|--------------------|-------|---------|
| {theme} | {X}% | {Y}% | {+/-}pp | {persona} |

#### Incumbent Provider Displacement
| Current Provider | Won (Displaced) | Lost (Failed) | Displacement Rate |
|-----------------|-----------------|---------------|-------------------|

### Product Gap Details

#### {Gap Name}
- **Deals affected:** {N}
- **ARR lost:** ${X}
- **Segments:** {most affected}
- **On roadmap:** {Yes/No}
- **CRM evidence:** {specific deals and field values}
- **Call evidence:** "{quote}" [Call: {title}, {date}]
- **Interview evidence:** "{quote}" [Interview: {source}]

{Repeat for each top gap}

### Evidence Appendix

#### Source Coverage
| Deal | CRM | Calls | Interviews | Enrichment |
|------|-----|-------|------------|------------|

#### Calls Analyzed
{List: title, date, linked deal, duration}

#### Transcript Intelligence Artifacts
{If enabled, list DuckDB path, JSONL exports, rollup exports, chunk count, signal count, and transcript scope. If skipped, state why.}

#### Interview Sources
{List: file name, company, date, type}

#### Data Freshness
- CRM: queried {date}, {N} records
- Calls: {API pull date or export date}, {N} calls
- Interviews: {N} files, {date range}
- Enrichment: {Revelio snapshot date}
```

---

## Step 7.5: Question Catalog (if --skip-questions not set)

**Skip if `--skip-questions` is set.**

Before generating the ICP file, run the question catalog to extract and classify prospect questions from the transcripts already collected.

### 7.5a: Save unified records for question-catalog

Write the unified deal records (from Step 3d) to a temporary data file that question-catalog can read:

Save to `{portfolio_path}/deliverables/strategic-analysis-data/unified-records.json`:
```json
{
  "deals": [ ...unified records from Step 3d... ],
  "transcripts": [ ...raw transcript data from Step 2... ],
  "transcript_intelligence": {
    "duckdb_path": "{portfolio_path}/strategic-analysis/cache/transcript_intelligence.duckdb",
    "signals_path": "{portfolio_path}/strategic-analysis/cache/extracted_signals.jsonl",
    "deal_evidence_path": "{portfolio_path}/strategic-analysis/cache/deal_evidence.jsonl",
    "rollups_path": "{portfolio_path}/strategic-analysis/cache/rollups/",
    "enabled": true
  },
  "speakers": [ ...speaker identification data... ],
  "personas": [ ...persona data from Agent 6... ],
  "config": {
    "company": "{company}",
    "portfolio_path": "{path}",
    "crm_type": "{type}",
    "call_recorder": "{recorder}"
  }
}
```

### 7.5b: Invoke question-catalog

Run: `/question-catalog {company} --from-strategic-analysis --months {days/30} --skip-gdoc`

(Skip the standalone Google Doc — the question catalog output will be included in the strategic-analysis Google Doc instead.)

### 7.5c: Include question catalog in strategic-analysis report

After question-catalog completes, read `{portfolio_path}/question-catalog/question-catalog.md` and include the following sections in the strategic-analysis report:

**In Layer 2 (Medium Dive), after the "Questions by Persona" table (around line 1700):**

Add:
```markdown
#### Question Catalog Summary

_Full catalog: `{portfolio_path}/question-catalog/`_

**Top Question Categories by Call Penetration:**

| Category | Penetration | Top Persona | Avg Response Quality | Win Rate Delta |
|----------|------------|-------------|---------------------|---------------|
| {from question-catalog.md Category Summary table} |

**Questions That Predict Winning:**

| Category | Win Rate (Asked) | Win Rate (Not Asked) | Delta |
|----------|-----------------|---------------------|-------|
| {from question-catalog.md} |

{If content gap analysis ran:}
**Top 5 Content Gaps:**

| Priority | Theme | Penetration | Coverage | Recommendation |
|----------|-------|------------|----------|----------------|
| {from question-catalog.md Content Gap Analysis, top 5 only} |
```

**In Layer 3 (Deep Dive), after the "Questions by Persona (Detailed)" table (around line 1861):**

Add:
```markdown
#### Full Question Catalog

See `{portfolio_path}/question-catalog/` for the complete catalog including:
- `question-catalog.json` — structured data ({N} questions)
- `questions-by-persona.md` — persona-level drill-down
- `questions-by-segment.md` — segment-level drill-down
- `questions-by-stage.md` — stage progression analysis
- `methodology.md` — data sources and confidence notes
```

### 7.5d: Include in Google Doc (Step 9)

When building the HTML for the Google Doc in Step 9a, include the Question Catalog Summary section from Layer 2 in the HTML output. Do NOT include the full catalog (too large for a Google Doc — link to local files instead).

**Report:** "Question catalog generated: {N} questions, {M} categories. Full catalog at `{portfolio_path}/question-catalog/`."

---

## Step 8: Generate Standalone ICP File

Save to: `{portfolio_path}/knowledge/icp-from-analysis-{YYYY-MM-DD}.md`

```markdown
# Data-Driven ICP: {Company Name}
**Generated:** {date}
**Based on:** {N} deals ({date range}), {N} calls, {N} interview sources
**Model quality:** XGBoost AUC = {score} | {or "No predictive model"}

## Best-Fit Customer Profile

### Firmographics
{Each dimension: specific range, win rate, sample size, confidence}

### Behavioral Signals
{Deal source, contract type, engagement patterns}

### Anti-ICP
{Characteristics to avoid, with data}

### Velocity Benchmarks
{Expected cycle times for ICP vs. non-ICP deals}

## Prescriptive Recommendations
{Under-invested winners with revenue upside}
{Fixable losers with specific actions}
{Abandon candidates with reallocation math}

## vs. Existing ICP
{Confirms / Contradicts / Adds}

## Key Evidence
{Top 3-5 Gong quotes and interview quotes that validate the profile}

## Methodology
{Data sources, sample sizes, model quality, lookback window}
```

---

## Step 9: Google Doc Export

**Skip if `--skip-gdoc` is set.**

### 9a: Convert Report to HTML

Build an HTML file at `/tmp/strategic-analysis-{company}-{date}.html`.

**Base template:**
```html
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8">
<style>
body { font-family: Arial, sans-serif; font-size: 11pt; line-height: 1.25; color: #000; }
h1 { font-size: 26pt; font-weight: normal; margin-top: 0; margin-bottom: 4pt; }
h2 { font-size: 20pt; font-weight: bold; margin-top: 28pt; margin-bottom: 8pt; }
h3 { font-size: 14pt; font-weight: bold; margin-top: 20pt; margin-bottom: 6pt; }
p { margin-top: 0; margin-bottom: 8pt; }
table { border-collapse: collapse; width: 100%; margin-top: 8pt; margin-bottom: 24pt; font-size: 10pt; }
th { background-color: #f3f3f3; font-weight: bold; text-align: left; padding: 8px 10px; border: 1px solid #d9d9d9; }
td { padding: 8px 10px; border: 1px solid #d9d9d9; vertical-align: top; }
.subtitle { font-size: 13pt; color: #666; margin-bottom: 8pt; }
table.callout-table { margin-top: 2pt; margin-bottom: 8pt; font-size: 11pt; }
.caption { font-size: 9pt; color: #666; font-style: italic; }
ul { margin-top: 4pt; margin-bottom: 12pt; }
li { margin-bottom: 4pt; }
</style>
</head>
<body>
<!-- Convert the .md report content to HTML -->
</body>
</html>
```

**Formatting rules:**
- `<h1>` for doc title, `<h2>` for layer headers and major sections, `<h3>` for subsections
- All data tables use `<table>` with gray headers
- Evidence strength: <span style="color: #1a7a1a">Strong</span>, <span style="color: #b36b00">Moderate</span>, <span style="color: #cc1111">Emerging</span>
- Executive Summary findings in callout boxes: `<table class="callout-table"><tr><td style="background-color: #e8f5e8; border: none; padding: 12px 16px;">`
- Layer separators: `<h2>` with extra top margin

### 9b: Upload to Google Drive

```bash
CLIENT_ID=$(python3 -c "import json; d=json.load(open('$HOME/.config/gws/credentials.json')); print(d['client_id'])")
CLIENT_SECRET=$(python3 -c "import json; d=json.load(open('$HOME/.config/gws/credentials.json')); print(d['client_secret'])")
REFRESH_TOKEN=$(python3 -c "import json; d=json.load(open('$HOME/.config/gws/credentials.json')); print(d['refresh_token'])")

ACCESS_TOKEN=$(curl -s -X POST https://oauth2.googleapis.com/token \
  -d "client_id=$CLIENT_ID" \
  -d "client_secret=$CLIENT_SECRET" \
  -d "refresh_token=$REFRESH_TOKEN" \
  -d "grant_type=refresh_token" | python3 -c "import json,sys; print(json.load(sys.stdin).get('access_token',''))")

curl -s -X POST "https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -F "metadata={\"name\": \"Strategic Analysis: {Company} ({date})\", \"mimeType\": \"application/vnd.google-apps.document\"};type=application/json" \
  -F "file=@/tmp/strategic-analysis-{company}-{date}.html;type=text/html"
```

Extract the `id` from the response. The Google Doc URL is: `https://docs.google.com/document/d/{id}/edit`

If the upload fails (expired token, missing credentials), keep the .md file and say: "Google Doc creation failed. The markdown report is saved at {path}. To create the Google Doc manually, re-authenticate your Google Workspace CLI credentials."

---

## Step 10: Present Findings

> Strategic analysis complete for **{Company}**.
>
> **Top findings:**
> 1. **{Finding 1}** {with specific number}
> 2. **{Finding 2}** {with specific number}
> 3. **{Finding 3}** {with specific number}
>
> **Your ICP today:** {one-sentence data-driven description}
>
> **Where to invest:** {one-sentence prescriptive recommendation with revenue upside}
>
> **Top buyer personas:** {persona1} ({X}% of deals, {Y}% win rate), {persona2} ({X}%, {Y}%)
>
> **Feature-persona insight:** {one-sentence key finding from the matrix}
>
> **Biggest data gap:** {what's missing and what it prevents knowing}
>
> **Reports saved:**
> - Full analysis: `{portfolio_path}/deliverables/strategic-analysis-{date}.md`
> - ICP document: `{portfolio_path}/knowledge/icp-from-analysis-{date}.md`
> - Google Doc: {URL or "skipped"}

---

## Error Handling

| Scenario | Action |
|----------|--------|
| CRM auth expired (SF) | Stop. "Run `sf org login web --alias {org}`" |
| CRM auth expired (HubSpot) | Stop. "Check HubSpot MCP private app token" |
| Gong API 401/403 | Warn, switch to `--skip-calls` |
| Gong API 429 | Wait per `Retry-After`, retry |
| Fathom API failure | Try API key fallback, then manual mode |
| No deals returned | Expand `--days`, check stage mappings |
| <30 deals | Warn small sample, proceed |
| Interview files missing | Proceed with CRM + calls, note in evidence appendix |
| Python missing (ML) | Auto-set `--skip-ml`, note in data gaps |
| XGBoost install fails | Auto-set `--skip-ml`, note in data gaps |
| Field not found | Skip dimension, note in data gaps |
| HubSpot rate limit | Wait 100ms between requests, retry with backoff |
| SOQL too long | Split into multiple queries |
| Google Doc upload fails | Keep .md, warn, suggest `re-authenticate your Google Workspace CLI` |
| Segment <3 deals | Include but flag "Small sample (N={count})" |
| Free text loss reasons | Cluster via Claude, present for confirmation |
| Snowflake MCP unavailable | Disable enrichment, note in data gaps |
| Revelio match rate low | Report: "Matched X of Y accounts. Unmatched accounts won't have enrichment data." |
| Prospeo API 401 | Warn invalid key, auto-set `--skip-prospeo`, use CRM titles |
| Prospeo API 429 | Wait per backoff (1s, 2s, 4s), max 3 retries, then skip remaining |
| Prospeo credit limit | Stop enrichment, report how many were enriched, proceed with partial data |
| No contacts on deals | Log as Medium data gap: "No contacts linked to opportunities. Persona analysis relies on call participant names only." |
| Contact Title field empty >50% | Warn: "Title field is {X}% populated. Persona classification will be incomplete for {Y} deals." |
| LinkedIn URL field doesn't exist | Auto-set `--skip-prospeo`, use CRM Title only, note in methodology |
| <5 deals per persona | Label all findings for that persona as "Insufficient data (N={count})" |
| --personas false | Skip Agent 6, Module 4J, all persona cross-tabs. Note in data gaps: "Persona analysis not run. Use `--personas` to enable." |
| Question catalog fails | Warn: "Question catalog generation failed: {error}. Proceeding without. Run `/question-catalog {company}` separately." Continue pipeline. |
| --skip-questions true | Skip Step 7.5. Note in methodology: "Question catalog not run. Use default flags or run `/question-catalog` separately." |

---

## Adding a New Company

### Salesforce Company

1. **Connect Salesforce:**
   ```bash
   sf org login web --alias your-org-alias
   ```

2. **Add to companies.json:**
   ```json
   "{CompanyName}": {
     "crm": {
       "type": "salesforce",
       "sf_org": "your-org-alias",
       "field_mapping": "auto"
     },
     "call_recorder": {
       "type": "gong"
     },
     "portfolio_path": "/path/to/company/workspace",
     "interview_paths": {
       "churn_interviews": "source-docs/churn-interviews/",
       "market_research": "source-docs/",
       "knowledge_files": [],
       "gong_exports": "source-docs/gong-*.md"
     }
   }
   ```

3. **Run:** `/strategic-analysis --company {CompanyName}` -- field discovery auto-runs on first use.

### HubSpot Company

1. **Set up HubSpot MCP** with a private app token.

2. **Add to companies.json:**
   ```json
   "{CompanyName}": {
     "crm": {
       "type": "hubspot",
       "field_mapping": "auto"
     },
     "call_recorder": {
       "type": "fathom"
     },
     "portfolio_path": "/path/to/company/workspace",
     "interview_paths": {
       "churn_interviews": "",
       "market_research": "",
       "knowledge_files": [],
       "gong_exports": ""
     }
   }
   ```

3. **Run:** `/strategic-analysis --company {CompanyName}` -- field discovery auto-runs.

### Call Recorder Options

- **Gong:** Set `GONG_API_KEY` and `GONG_API_SECRET` in `~/.env`
- **Fathom:** Set `FATHOM_API_KEY` in `~/.env`
- **Manual:** Set `"call_recorder": {"type": "manual"}` -- you'll be prompted to paste transcripts

### Enrichment

To enable Revelio Labs enrichment, ensure Snowflake MCP is connected and run with `--enrich`. No per-company config needed; enrichment works by matching account names to the Revelio Labs company database.
