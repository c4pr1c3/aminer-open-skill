---
name: aminer-data-search
version: 1.0.8
author: AMiner
contact: report@aminer.cn
description: >
  Full-featured AMiner skill with 28 APIs and 6 workflows. Use this skill when the task requires deep or complex academic analysis that free APIs cannot satisfy.
  Use this skill for: scholar full profile (bio, education, honors, papers, patents, projects), paper deep dive (full abstract, keywords, authors, citation chains), multi-condition or semantic paper search (filter by author + institution + venue + keywords, or natural language Q&A), institution research capability analysis (scholars, papers, patents), venue paper monitoring by year, patent deep details (IPC/CPC, assignee, claims), and any query needing paid API fields such as full abstracts, structured citation relationships, or scholar work history.
  Do NOT use this skill for simple lookups that free APIs can answer — such as checking a paper title, identifying a scholar by name, normalizing an institution or venue name, or scanning patent trends by keyword. For those, use aminer-free-search instead.
  Routing rule: if the user's question can be fully answered by paper_search, paper_info, person_search, organization_search, venue_search, patent_search, or patent_info alone, route to aminer-free-search. Otherwise use this skill.
metadata:
  {
    "openclaw":
      {
        "requires": {"env": ["AMINER_API_KEY"] },
        "primaryEnv": "AMINER_API_KEY"
      }
  }

---

# AMiner Open Platform Academic Data Query

AMiner is a globally leading academic data platform that provides comprehensive academic data covering scholars, papers, institutions, journals, patents, and more.
This skill covers all 28 open APIs and organizes them into 6 practical workflows.
Before use, please generate a token in the console and set it as the environment variable `AMINER_API_KEY` for automatic script access.

- **API Documentation**: https://open.aminer.cn/open/docs
- **Console (Generate Token)**: https://open.aminer.cn/open/board?tab=control

---

## High-Priority Mandatory Rules (Critical)

The following four rules are the **highest priority** and must be followed in any query task:

1. **Token Security**: Only check whether `AMINER_API_KEY` exists; never expose the token in plain text in any location (including terminal output, logs, example results, or debug information).
2. **Cost Control**: Always prefer the optimal combined query; never perform indiscriminate full-detail retrieval. When many results are matched and the user has not specified a count, default to fetching details for only the top 10 results.
3. **Free-First**: Prefer free APIs unless the user explicitly requires deeper fields or higher precision; only upgrade to paid APIs when free ones cannot meet the need.
4. **Result Links**: Whenever this skill is used and the result contains entities (papers/scholars/patents/journals), an accessible URL must be appended after each entity, regardless of the scenario or output format.

Entity URL templates (mandatory):
- Paper: `https://www.aminer.cn/pub/{paper_id}`
- Scholar: `https://www.aminer.cn/profile/{scholar_id}`
- Patent: `https://www.aminer.cn/patent/{patent_id}`
- Journal: `https://www.aminer.cn/open/journal/detail/{journal_id}`

> Enforcement note: This rule applies to all returned results (including summaries, lists, details, comparative analyses, workflow outputs, and raw output transcriptions). Whenever an entity appears and a usable ID is available, a link must be attached.

> Violating any of the above rules is considered a process non-compliance; execution must be immediately halted and corrected before continuing.

---

## Step 1: Check Environment Variable Token (Required)

Before making any API call, you must first check whether the environment variable `AMINER_API_KEY` exists (request headers should default to `Authorization: ${AMINER_API_KEY}` and include `X-Platform: openclaw`).
Only determine whether it "exists / does not exist"; never output, echo, or log the token in plain text (including logs, terminal output, or example results).

**Standard check (recommended for direct use):**
```bash
if [ -z "${AMINER_API_KEY+x}" ]; then
    echo "AMINER_API_KEY does not exist"
else
    echo "AMINER_API_KEY exists"
fi
```

- **If the token already exists in the environment variable**: proceed with the subsequent query workflow.
- **If no token is in the environment variable**: check whether the user has explicitly provided `--token`.
- **If neither the environment variable nor `--token` is available**: stop immediately; do not call any API or enter any subsequent workflow; guide the user to obtain a token first.

**Recommended token configuration (preferred):**
1. Go to the [AMiner Console](https://open.aminer.cn/open/board?tab=control), log in, and generate an API Token.
2. Set the token as an environment variable: `export AMINER_API_KEY="<TOKEN>"`
3. The script reads the environment variable `AMINER_API_KEY` by default (if `--token` is explicitly provided, it takes precedence).

**Guidance when no token is available:**
1. Clearly inform the user: "A token is currently missing; AMiner API calls cannot continue."
2. Direct the user to the [AMiner Console](https://open.aminer.cn/open/board?tab=control) to log in and generate an API Token.
3. For assistance, refer to the [Open Platform Documentation](https://open.aminer.cn/open/docs).
4. Prompt the user to continue after obtaining a token; they can reply directly: `Here is my token: <TOKEN>`

> The token can be generated in the console and reused within its validity period. Do not execute any data query steps before obtaining a token.

---

## Quick Start (curl)

Use direct `curl` calls by default. A Python client is optional, not required.

Recommended common headers:

- `Authorization: ${AMINER_API_KEY}` by default
- `X-Platform: openclaw`
- `Content-Type: application/json;charset=utf-8` for POST requests

Single-endpoint example:
```bash
curl -X GET \
  'https://datacenter.aminer.cn/gateway/open_platform/api/paper/search?page=1&size=5&title=BERT' \
  -H "Authorization: ${AMINER_API_KEY}" \
  -H 'X-Platform: openclaw'
```

Compose multi-step workflows by following the call chains in this skill.

**Direct-call guardrails (mandatory):**
1. Verify the endpoint signature before calling; parameter names and types must match exactly.
2. Parameter constraints are governed by `references/api-catalog.md`.
3. `paper_info` is batch-only and must use `{"ids": [...]}`.
4. `paper_detail` is single-paper only and must use one `id`.
5. When multiple paper details are needed, filter first and detail later.
6. Prefer the free path first, then upgrade only when deeper fields are required.

---

## Stability and Failure Handling Strategy (Must Read)

The client `scripts/aminer_client.py` has built-in request retry and fallback strategies to reduce the impact of network fluctuations and transient service errors on results.

- **Timeout and Retry**
  - Default request timeout: `30s`
  - Maximum retries: `3`
  - Backoff strategy: exponential backoff (`1s -> 2s -> 4s`) + random jitter
- **Retryable Status Codes**
  - `408 / 429 / 500 / 502 / 503 / 504`
- **Non-Retryable Scenarios**
  - Common `4xx` errors (e.g., parameter errors, authentication issues) are not retried by default; an error structure is returned directly.
- **Workflow Fallback**
  - `paper_deep_dive`: automatically falls back to `paper_search_pro` if `paper_search` yields no results.
  - `paper_qa`: automatically falls back to `paper_search_pro` if the `query` mode yields no results.
- **Traceable Call Chain**
  - Combined workflow output includes `source_api_chain`, marking which APIs were combined to produce the result.

---

## Paper Search API Selection Guide

When the user says "search for papers", first determine whether the goal is "find an ID", "filter results", "Q&A", or "generate an analysis report", then select the API:

| API | Focus | Use Case | Cost |
|---|---|---|---|
| `paper_search` | Title search, quickly get `paper_id` | Known paper title, locate the target paper first | Free |
| `paper_search_pro` | Multi-condition search and sorting (author/institution/journal/keywords) | Topic search, sort by citations or year | ¥0.01/call |
| `paper_qa_search` | Natural language Q&A / topic keyword search | User describes need in natural language; semantic search first | ¥0.05/call |
| `paper_list_by_search_venue` | Returns more complete paper info (suitable for analysis) | Need richer fields for analysis/reports | ¥0.30/call |
| `paper_list_by_keywords` | Multi-keyword batch retrieval | Batch thematic retrieval (e.g., AlphaFold + protein folding) | ¥0.10/call |
| `paper_detail_by_condition` | Retrieve details by year + journal dimension | Journal annual monitoring, venue selection analysis | ¥0.20/call |

Recommended routing (default):

1. **Known title**: `paper_search -> paper_detail -> paper_relation`
2. **Conditional filtering**: `paper_search_pro -> paper_detail`
3. **Natural language Q&A**: `paper_qa_search` (fall back to `paper_search_pro` if no results)
4. **Journal annual analysis**: `venue_search -> venue_paper_relation -> paper_detail_by_condition`

Key free-tier screening fields currently available:

- `paper_search`: `venue_name`, `first_author`, `n_citation_bucket`, `year`
- `paper_info`: `abstract_slice`, `year`, `venue_id`, `author_count`
- `person_search`: `interests`, `n_citation`, `org/org_id`
- `organization_search`: `aliases`
- `venue_search`: `aliases`, `venue_type`
- `patent_search`: `inventor_name`, `app_year`, `pub_year`
- `patent_info`: `app_year`, `pub_year`

Supplementary rules (strongly recommended):

1. **When searching by title only**, always use `paper_search` first (free) to quickly locate the paper ID.
2. **For complex semantic retrieval** (natural language, multi-condition, fuzzy expressions), prefer `paper_qa_search`.
3. When using `paper_qa_search`, first break the natural language need into structured conditions, then fill in the fields (e.g., year, topic keywords, author/institution, etc.).
4. `query` and `topic_high/topic_middle/topic_low` are **mutually exclusive**: choose one; do not pass both simultaneously.
5. When using `query` mode, fill in a natural language string directly; when using `topic_*` mode, first expand with synonyms/English variants before filling in.
6. Example: querying "AI-related papers from 2012":
   - `year` → `[2012]`
   - Option A: `query` → `"artificial intelligence"`
   - Option B: `topic_high` → `[["artificial intelligence","ai","Artificial Intelligence"]]` (with `use_topic` enabled)

---

## Handling Out-of-Workflow Requests (Required)

When the user's request **falls outside the 6 workflows above**, or existing workflows cannot directly cover it, the following steps must be executed:

1. First read `references/api-catalog.md` to confirm available interfaces, parameter constraints, and response fields.
2. Select the most appropriate API based on the user's goal and design the shortest viable call chain (locate ID first, then supplement details, then expand relationships).
3. When necessary, combine multiple APIs to complete the query, and annotate `source_api_chain` in the result to clearly describe the data source path.
4. If multiple combination approaches exist, prefer the one with lower cost, higher stability, and fields that satisfy the requirement.
5. Use the "optimal query combination" as much as possible; avoid indiscriminate full retrieval; perform low-cost search and filtering first, then fetch details for a small set of targets.
6. When results are large and the user has not specified a count, default to querying only the top 10 details and returning a summary first; for example, when 1000 papers are matched, do not call the detail API for all 1000 to reduce user costs.
7. For `raw` calls, parameter-level validation is required: e.g., `paper_info` uses `ids`, `paper_detail` uses `paper_id`; do not mix them up.
8. When the user has not explicitly requested deep information, prefer the free path (e.g., `paper_search` / `paper_info` / `venue_search`); only supplement with necessary paid APIs after confirming they are insufficient.
9. When returning the final entity list, the corresponding URL must be included; if entity IDs are missing, supplement them before outputting results.

> Do not give up on a query simply because "no existing workflow fits"; actively complete the API combination based on `api-catalog`.

---

## 6 Combined Workflows

### Workflow 1: Scholar Profile

**Use Case**: Understand a scholar's complete academic profile, including bio, research interests, published papers, patents, and research projects.

**Call Chain:**
```
Scholar search (name → person_id)
    ↓
Parallel calls:
  ├── Scholar details (bio/education/honors)
  ├── Scholar portrait (research interests/experience/work history)
  ├── Scholar papers (paper list)
  ├── Scholar patents (patent list)
  └── Scholar projects (research projects/funding info)
```

**curl Example:**
```bash
curl -X POST \
  'https://datacenter.aminer.cn/gateway/open_platform/api/person/search' \
  -H 'Content-Type: application/json;charset=utf-8' \
  -H "Authorization: ${AMINER_API_KEY}" \
  -H 'X-Platform: openclaw' \
  -d '{"name":"Yann LeCun","size":5}'
```

**Sample output fields:**
- Basic info: name, institution, title, gender
- Personal bio (bilingual)
- Research interests and domains (use free-search `interests` first for quick screening)
- Education history (structured)
- Work experience (structured)
- Paper list (ID + title)
- Patent list (ID + title)
- Research projects (title/funding amount/dates)

---

### Workflow 2: Paper Deep Dive

**Use Case**: Retrieve complete paper information and citation relationships based on a paper title or keywords.

**Call Chain:**
```
Paper search / Paper search pro (title/keyword → paper_id)
    ↓
Paper details (abstract/authors/DOI/journal/year/keywords)
    ↓
Paper citations (which papers this paper cites → cited_ids)
    ↓
(Optional) Batch retrieve basic info for cited papers
```

**curl Example:**
```bash
curl -X GET \
  'https://datacenter.aminer.cn/gateway/open_platform/api/paper/search?page=1&size=5&title=BERT' \
  -H "Authorization: ${AMINER_API_KEY}" \
  -H 'X-Platform: openclaw'
```

---

### Workflow 3: Org Analysis

**Use Case**: Analyze an institution's scholar size, paper output, and patent count; suitable for competitive research or partnership evaluation.

**Call Chain:**
```
Org disambiguation pro (raw string → org_id, handles alias/full-name differences)
    ↓
Parallel calls:
  ├── Org details (description/type/founding date)
  ├── Org scholars (scholar list)
  ├── Org papers (paper list)
  └── Org patents (patent ID list, supports pagination, up to 10,000)
```

> If multiple institutions share the same name, org search returns a candidate list; use org disambiguation pro for precise matching.

**curl Example:**
```bash
curl -X POST \
  'https://datacenter.aminer.cn/gateway/open_platform/api/organization/na/pro' \
  -H 'Content-Type: application/json;charset=utf-8' \
  -H "Authorization: ${AMINER_API_KEY}" \
  -H 'X-Platform: openclaw' \
  -d '{"org":"Massachusetts Institute of Technology, CSAIL"}'
```

---

### Workflow 4: Venue Papers

**Use Case**: Track papers published in a specific journal for a given year; useful for submission research or research trend analysis.

**Call Chain:**
```
Venue search (name → venue_id, free response includes `aliases` and `venue_type`)
    ↓
Venue details (ISSN/type/abbreviation)
    ↓
Venue papers (venue_id + year → paper_id list)
    ↓
(Optional) Batch paper detail query
```

**curl Example:**
```bash
curl -X POST \
  'https://datacenter.aminer.cn/gateway/open_platform/api/venue/search' \
  -H 'Content-Type: application/json;charset=utf-8' \
  -H "Authorization: ${AMINER_API_KEY}" \
  -H 'X-Platform: openclaw' \
  -d '{"name":"NeurIPS"}'
```

---

### Workflow 5: Paper QA Search

**Use Case**: Intelligently search for papers using natural language or structured keywords; supports SCI filtering, citation-based sorting, author/institution constraints.

**Core API**: `Paper QA Search` (¥0.05/call), supports:
- `query`: natural language question; system automatically breaks it into keywords
- `topic_high/middle/low`: fine-grained keyword weight control (nested array OR/AND logic)
- `sci_flag`: show SCI papers only
- `force_citation_sort`: sort by citation count
- `force_year_sort`: sort by year
- `author_terms / org_terms`: filter by author name or institution name
- `author_id / org_id`: filter by author ID or institution ID (recommended for disambiguation)
- `venue_ids`: filter by conference/journal ID list

**curl Example:**
```bash
curl -X POST \
  'https://datacenter.aminer.cn/gateway/open_platform/api/paper/qa/search' \
  -H 'Content-Type: application/json;charset=utf-8' \
  -H "Authorization: ${AMINER_API_KEY}" \
  -H 'X-Platform: openclaw' \
  -d '{"use_topic":false,"query":"deep learning methods for protein structure prediction","size":10,"sci_flag":true}'
```

---

### Workflow 6: Patent Analysis

**Use Case**: Search for patents in a specific technology domain, or retrieve a scholar's/institution's patent portfolio.

**Call Chain (standalone search):**
```
Patent search (query → patent_id, free response includes `inventor_name`, `app_year`, `pub_year`)
    ↓
Patent info / Patent details (basic year-number card / deeper structured fields)
```

**Call Chain (via scholar/institution):**
```
Scholar search → Scholar patents (patent_id list)
Org disambiguation → Org patents (patent_id list)
    ↓
Patent info / Patent details
```

**curl Example:**
```bash
curl -X POST \
  'https://datacenter.aminer.cn/gateway/open_platform/api/patent/search' \
  -H 'Content-Type: application/json;charset=utf-8' \
  -H "Authorization: ${AMINER_API_KEY}" \
  -H 'X-Platform: openclaw' \
  -d '{"query":"quantum computing chip","page":0,"size":10}'
```

---

## Individual API Quick Reference

> For complete parameter descriptions, read `references/api-catalog.md`

| # | Title | Method | Price | API Path (Base domain: datacenter.aminer.cn/gateway/open_platform) |
|---|------|------|------|------|
| 1 | Paper QA Search | POST | ¥0.05 | `/api/paper/qa/search` |
| 2 | Scholar Search | POST | Free | `/api/person/search` |
| 3 | Paper Search | GET | Free | `/api/paper/search` |
| 4 | Paper Search Pro | GET | ¥0.01 | `/api/paper/search/pro` |
| 5 | Patent Search | POST | Free | `/api/patent/search` |
| 6 | Org Search | POST | Free | `/api/organization/search` |
| 7 | Venue Search | POST | Free | `/api/venue/search` |
| 8 | Scholar Details | GET | ¥1.00 | `/api/person/detail` |
| 9 | Scholar Projects | GET | ¥3.00 | `/api/project/person/v3/open` |
| 10 | Scholar Papers | GET | ¥1.50 | `/api/person/paper/relation` |
| 11 | Scholar Patents | GET | ¥1.50 | `/api/person/patent/relation` |
| 12 | Scholar Portrait | GET | ¥0.50 | `/api/person/figure` |
| 13 | Paper Info | POST | Free | `/api/paper/info` |
| 14 | Paper Details | GET | ¥0.01 | `/api/paper/detail` |
| 15 | Paper Citations | GET | ¥0.10 | `/api/paper/relation` |
| 16 | Patent Info | GET | Free | `/api/patent/info` |
| 17 | Patent Details | GET | ¥0.01 | `/api/patent/detail` |
| 18 | Org Details | POST | ¥0.01 | `/api/organization/detail` |
| 19 | Org Patents | GET | ¥0.10 | `/api/organization/patent/relation` |
| 20 | Org Scholars | GET | ¥0.50 | `/api/organization/person/relation` |
| 21 | Org Papers | GET | ¥0.10 | `/api/organization/paper/relation` |
| 22 | Venue Details | POST | ¥0.20 | `/api/venue/detail` |
| 23 | Venue Papers | POST | ¥0.10 | `/api/venue/paper/relation` |
| 24 | Org Disambiguation | POST | ¥0.01 | `/api/organization/na` |
| 25 | Org Disambiguation Pro | POST | ¥0.05 | `/api/organization/na/pro` |
| 26 | Paper Search by Venue | GET | ¥0.30 | `/api/paper/list/by/search/venue` |
| 27 | Paper Batch Query | GET | ¥0.10 | `/api/paper/list/citation/by/keywords` |
| 28 | Paper Details by Year and Venue | GET | ¥0.20 | `/api/paper/platform/allpubs/more/detail/by/ts/org/venue` |

---

## References

- Full API parameter documentation: read `references/api-catalog.md`
- Optional Python client: `scripts/aminer_client.py`
- Test cases: `evals/evals.json`
- Official documentation: https://open.aminer.cn/open/docs
- Console: https://open.aminer.cn/open/board?tab=control
