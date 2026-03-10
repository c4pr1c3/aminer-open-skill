# aminer-open-skill

Turn AMiner's academic data capabilities into a ready-to-use query and analysis Skill.  
Just provide a research question (look up a scholar / paper / institution / journal / patent), and it will automatically run the corresponding API workflow and return structured results.

## What This Skill Does in One Line

`aminer-data-search` is designed for academic information retrieval and lightweight analysis. It integrates 28 APIs under the hood and wraps them into 6 common workflows, reducing the overhead of manually assembling API calls and aligning fields.

## What Problems It Solves

- Look up a scholar: bio, research interests, papers, patents, projects
- Look up a paper or papers: details, citation relationships, keyword expansion
- Look up an institution: scholar size, paper output, patent distribution
- Look up a journal: papers from a specific year and topic tracking
- Ask academic questions in natural language: e.g., "latest advances in Transformer"
- Look up patents in a technology domain: and chain to scholar/institution patent relationships

## Get Started in 3 Minutes

### 1) Prepare a Token (Required)

Generate a Token in the AMiner Console:  
https://open.aminer.cn/open/board?tab=control

### 2) Configure Environment Variable (Recommended)

```bash
export AMINER_API_KEY="<YOUR_TOKEN>"
```

### 3) Run Examples

```bash
# Look up a scholar profile
python skills/aminer-data-search/scripts/aminer_client.py \
  --action scholar_profile --name "Andrew Ng"

# Deep-dive a paper by title (with citation chain)
python skills/aminer-data-search/scripts/aminer_client.py \
  --action paper_deep_dive --title "Attention is all you need"

# Analyze institutional research capability
python skills/aminer-data-search/scripts/aminer_client.py \
  --action org_analysis --org "Tsinghua University"

# Search papers with natural language Q&A
python skills/aminer-data-search/scripts/aminer_client.py \
  --action paper_qa --query "latest advances in transformer architecture"
```

## Common Usage Patterns

- **Task-based workflow**: suitable for "give me complete results" needs (e.g., scholar_profile, paper_deep_dive)
- **Fine-grained API calls**: suitable for "call just one API" needs (`--action raw` + `--api` + `--params`)
- **Cost-control strategy**: use free/low-cost APIs to locate targets first, then call expensive detail APIs

## Directory Structure

- `skills/aminer-data-search/SKILL.md`: Full capability description, workflow design, and call constraints
- `skills/aminer-data-search/scripts/aminer_client.py`: Python client and command-line entry point
- `skills/aminer-data-search/references/api-catalog.md`: Quick reference for all 28 API parameters and paths
- `skills/aminer-data-search/evals/evals.json`: Evaluation cases and test samples

## Notes

- Do not continue calling APIs without a Token (use `AMINER_API_KEY` or `--token`)
- The client has built-in timeout retry and partial fallback strategies to improve request stability
- Some APIs are billed; confirm the scenario before scaling up calls

## References

- AMiner Open Platform Documentation: https://open.aminer.cn/open/docs
- Skill Detailed Documentation: `skills/aminer-data-search/SKILL.md`
