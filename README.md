# c10gdn-claude-skills

Personal Claude Code plugin marketplace — reusable skills distilled from real
projects, grouped by domain so each project only installs what it needs.

## Plugins

| Plugin | Skills | Use on projects that... |
|---|---|---|
| `aws-ops` | `terraform-two-phase-deploy`, `aws-cost-reconciliation`, `aws-local-first-testing` | Deploy to AWS via Terraform (Lambda/ECS/etc.) |
| `scraping-toolkit` | `bot-protection-bypass`, `multi-source-scraper-architecture` | Scrape sites, especially bot-protected ones, from one or more sources |
| `trading-research` | `strategy-cost-model`, `benchmark-noise-floor`, `broker-api-preflight`, `money-path-safety` | Automate or evaluate trading strategies against a real broker |

## Install (per project — not global)

This marketplace is designed to be installed **per project**, so a repo only picks
up the skill domains it actually needs — see the `aws-ops`/`scraping-toolkit` split
above.

From inside the target project's Claude Code session:

```
/plugin marketplace add c10gdn-dev/c10gdn-claude-skills
/plugin install aws-ops@c10gdn-claude-skills
/plugin install scraping-toolkit@c10gdn-claude-skills
```

(Install only the plugin(s) relevant to that project — most projects will want one,
not both.)

### Testing changes locally before pushing

```
claude plugin marketplace add /path/to/c10gdn-claude-skills
claude plugin validate .
```

## Adding a new skill

1. Decide which existing plugin it belongs to, or whether it's the start of a new
   domain (new plugin dir + `.claude-plugin/plugin.json`, and register it in
   `.claude-plugin/marketplace.json`).
2. `plugins/<plugin>/skills/<skill-name>/SKILL.md` with YAML frontmatter
   (`name`, `description`) — the `description` is what Claude reads to decide when
   to invoke it, so write it as a trigger condition, not a summary.
3. `claude plugin validate .` from the repo root before committing.

## Updating an installed plugin

```
/plugin marketplace update c10gdn-claude-skills
```
