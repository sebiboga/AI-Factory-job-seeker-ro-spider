# AI Factory — job_seeker_ro_spider

Autonomous scraper factory for Romanian companies.

Triggers a GitHub Actions workflow that uses `opencode` (big-pickle model) with Chrome DevTools and `gh` CLI to:
1. Search ANAF for the company CIF
2. Scan existing repos for CIF matches
3. Create or update a scraped repo from the EPAM template
4. Customise config, scraper logic, tests, and docs
5. Set topics, GitHub Pages, and `SOLR_AUTH` secret
6. Trigger the scraper CI and verify it passes
7. Update the EPAM template's Derived Scrapers table

No user interaction required.

## Usage

Go to **Actions → AI Factory — Autonomous Scraper Derivation → Run workflow**:

| Input | Required | Description |
|---|---|---|
| `company` | yes | Company name or brand (e.g. `GAMINVEST`, `TEC Agency`) |
| `cif` | no | CIF if already known (otherwise the AI searches ANAF) |

## Required secrets

| Secret | Description |
|---|---|
| `GH_TOKEN` | GitHub PAT with `repo`, `workflow`, `admin:org` scopes |
| `PAT_TOKEN` | Same or separate PAT for git push |
| `SOLR_AUTH` | SOLR credentials (`user:password`) — propagated to derived repos |

## How it works

1. The workflow sets up Node.js, Chrome (headless + remote debugging), and `opencode` CLI
2. `opencode run` executes with the company input, reading `ALGORITHM.md` as instructions
3. The AI model browses ANAF via Chrome, inspects career pages, runs `gh`, edits files, pushes repos
4. All decisions are made autonomously based on the algorithm
5. On completion, the derived scraper is live with CI and Pages configured
