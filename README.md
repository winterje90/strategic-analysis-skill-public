# Strategic Analysis for B2B SaaS

A Claude Code skill that pulls CRM data, call transcripts, and interview evidence for a B2B SaaS company, links them together, and produces a layered strategic report. Designed for B2B SaaS teams and consultants working with Salesforce or HubSpot CRM.

## What It Answers

- Why you win and lose deals (by segment, not in aggregate)
- Who your ICP actually is vs. who you think it is
- Which competitors beat you, where, and why
- What feature gaps cost you in real revenue
- Where your pricing ceiling is by tier
- How fast deals move and where they stall
- What data you're missing and what it prevents you from knowing

## How It Works

```
/strategic-analysis --company ExampleCo --days 180
```

The skill pulls data from up to 4 sources in parallel:

| Source | What It Gets | Required? |
|--------|-------------|-----------|
| **CRM** (Salesforce or HubSpot) | All closed deals with field data | Yes |
| **Call Recorder** (Gong, Fathom, or manual paste) | Transcripts and AI summaries linked to deals | No |
| **Knowledge Files** (portfolio folder) | Interview transcripts, competitive intel, product docs | No |
| **Revelio Labs** (Snowflake) | Headcount, growth rates, industry for each account | No (use `--enrich`) |

It links these together by account, converts call transcripts into a structured evidence store, runs analysis modules, and produces three outputs:

1. **Layered .md report** (exec summary / medium dive / deep dive)
2. **Standalone ICP file** saved to `knowledge/`
3. **Google Doc** (auto-created from the report)

## Transcript Intelligence

Large transcript volumes are handled through a structured evidence layer rather than raw transcript search.

The default flow is:

1. Normalize Gong/Fathom/manual transcripts into one call schema.
2. Assign each call a processing tier.
3. Fully process closed won/lost deal-linked calls by default.
4. Chunk transcripts by speaker turns and topic boundaries.
5. Extract structured signals: buyer questions, pricing objections, feature gaps, competitor mentions, implementation concerns, ROI drivers, buying criteria, win signals, and loss risks.
6. Store calls, chunks, signals, deal evidence, and rollups in a local DuckDB database.
7. Use the evidence store for competitive, pricing, feature, persona, and question-catalog analysis.

DuckDB is free and local. The cache is stored under:

```text
{portfolio_path}/strategic-analysis/cache/transcript_intelligence.duckdb
```

JSONL and CSV/Markdown rollup exports are also written so the evidence remains inspectable.

## Output Layers

| Layer | Who Reads It | Time | What's In It |
|-------|-------------|------|-------------|
| Executive Summary | CEO | 3 min | Headline metrics by segment, top 5 findings with sample sizes, ICP, data gaps, 3 recommended actions |
| Medium Dive | VP Sales/Marketing | 15 min | All segment tables with cross-tabs, loss drivers by tier, competitors, velocity, pricing, feature gaps, trends |
| Deep Dive | Analyst | 30 min | Pattern reliability ratings, full ICP with evidence, per-competitor evidence blocks, product gap details, evidence appendix |

## Analytical Principles

Two rules govern every analysis module:

**Repeatability over outliers.** Minimum 5 deals to draw a conclusion. Every finding shows with-and-without-outlier metrics. Patterns require 5+ deals across 2+ quarters. One-time events (platform migrations, competitor shutdowns) are flagged, not treated as trends.

**Segmented by default.** No aggregate number without the segment breakdown next to it. Win rate is always shown by tier, source, and deal type. A 57% overall win rate that's really 33% new business and 100% renewals is misleading, and the skill won't let that slide.

## Parameters

| Parameter | Default | What It Does |
|-----------|---------|-------------|
| `--company Name` | *(required)* | Which company from `companies.json` |
| `--days N` | 180 | How far back to look |
| `--depth shallow\|medium\|deep\|all` | all | Which report layers to generate |
| `--segment acv\|vertical\|source\|competitor\|owner\|all` | all | Which dimensions to segment by |
| `--enrich` | off | Pull Revelio Labs data (headcount, growth, industry) via Snowflake |
| `--skip-calls` | off | Use exported call data instead of live API |
| `--skip-transcript-intel` | off | Skip structured transcript intelligence and fall back to raw transcript search |
| `--transcript-scope closed-linked\|all-linked\|all` | closed-linked | Which calls receive full transcript processing |
| `--skip-ml` | off | Skip XGBoost predictive modeling |
| `--skip-gdoc` | off | Skip Google Doc creation |

## Analysis Modules

| Module | What It Does |
|--------|-------------|
| Temporal Foundation | Quarter-by-quarter trends, inflection point detection |
| Segmented Analysis | Win rates by tier/vertical/source/competitor/owner with cross-tabs (Tier x Source, Tier x Competitor, Source sub-types) |
| Loss Reason Analysis | Ranked by frequency AND revenue impact, with trends. Clusters free-text reasons automatically. |
| Competitive Intelligence | Per-competitor displacement rates by tier, evidence from calls/interviews, emerging vs. fading threats |
| Deal Velocity | Stage-by-stage median days (won vs. lost), stall point identification, velocity by tier/source/competitor |
| Pricing Analysis | Win rate by price band within each tier, sweet spot, ceiling, contract preferences, call evidence |
| Feature Gap Analysis | Gaps ranked by deals affected and ARR lost, cross-referenced with product roadmap |
| Predictive / XGBoost | Feature importance, correlations, interaction effects, hidden drivers (requires Python) |
| ICP Synthesis | Descriptive ("who IS your customer") + Prescriptive ("who SHOULD BE") + Anti-ICP, with repeatability filters |

## Setup

### 1. Add a Company

Edit `~/.claude/skills/strategic-analysis/companies.json`:

**For Salesforce:**
```json
"YourCompany": {
  "crm": {
    "type": "salesforce",
    "sf_org": "yourorg@example.com",
    "field_mapping": "auto"
  },
  "call_recorder": {
    "type": "gong"
  },
  "portfolio_path": "/path/to/company/folder",
  "interview_paths": {
    "churn_interviews": "source-docs/churn-interviews/",
    "market_research": "source-docs/",
    "knowledge_files": [],
    "gong_exports": "source-docs/gong-*.md"
  }
}
```

**For HubSpot:**
```json
"YourCompany": {
  "crm": {
    "type": "hubspot",
    "field_mapping": "auto"
  },
  "call_recorder": {
    "type": "fathom"
  },
  "portfolio_path": "/path/to/company/folder",
  "interview_paths": {
    "churn_interviews": "",
    "market_research": "",
    "knowledge_files": [],
    "gong_exports": ""
  }
}
```

Setting `field_mapping` to `"auto"` triggers field discovery on first run. The skill will scan your CRM fields, propose a mapping, and save it after you confirm.

### 2. Connect Your CRM

**Salesforce:** `sf org login web --alias yourorg@example.com`

**HubSpot:** Configure HubSpot MCP with a private app token.

### 3. Connect Your Call Recorder (Optional)

**Gong:** Add to `~/.env`:
```
GONG_API_KEY=your-key
GONG_API_SECRET=your-secret
```

**Fathom:** Add to `~/.env`:
```
FATHOM_API_KEY=your-key
```

**Manual:** Set `"call_recorder": {"type": "manual"}` and you'll be prompted to paste transcripts.

### 4. Python Dependencies

For transcript intelligence:

```bash
pip3 install duckdb
```

If DuckDB is not installed, the skill attempts to install it. If install fails, transcript intelligence is skipped and the analysis falls back to raw transcript search.

For XGBoost predictive modeling:

```bash
pip3 install xgboost scikit-learn pandas numpy
```

If these aren't installed, the skill auto-skips predictive modeling and notes it in the data gaps section.

## Files

```
~/.claude/skills/strategic-analysis/
  SKILL.md           # The skill definition (1300 lines)
  companies.json     # Per-company configuration
  README.md          # This file
```

**Outputs go to the company's portfolio folder:**
```
{portfolio_path}/
  deliverables/
    strategic-analysis-YYYY-MM-DD.md    # Full layered report
  knowledge/
    icp-from-analysis-YYYY-MM-DD.md     # Standalone ICP file
```

**Google Doc** is created in Google Drive with the title "Strategic Analysis: {Company} (YYYY-MM-DD)".

## Optional Companion Workflows

This skill references optional companion workflows for feature taxonomy generation and question catalog generation. If you do not have those workflows installed, the skill falls back to transcript-derived feature discovery and its built-in question extraction guidance.

## Example Output (ExampleCo, Sample Analysis)

From a sample 90-day analysis with 204 deals and 304 call transcripts:

**Executive Summary finding (before analytical rigor update):**
> "Partner-sourced leads convert at 55.6%"

**Same finding with repeatability and segmentation applied:**
> "Agency partner referrals for Tier 3 (SMB) deals win at 66.7% (16W/8L). Repeatable Pattern. Tech partner referrals win at 27.3% (3W/8L), below average. The aggregate 55.6% Partner number masks this split."

That's the difference the analytical principles make. The first version would have led to "invest in the Partner channel." The second leads to "invest specifically in agency partner relationships for SMB deals."

## Configuration

- Add your own company in `examples/companies.example.json` and copy it to your local config.

To add more companies, follow the setup instructions above.
