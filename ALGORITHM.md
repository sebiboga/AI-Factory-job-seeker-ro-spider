# AI Factory — Autonomous Scraper Derivation Algorithm

**Audience:** AI agents (opencode, big-pickle model) executing inside a GitHub Actions workflow.

**Goal:** Given a Romanian company name (and optionally CIF), autonomously create or recreate a Node.js scraper repo from the EPAM template, configure it, and verify it runs.

**Root constraint:** Fully autonomous — NO user interaction. You must decide, act, and verify alone.

---

## Environment

All set up by the workflow — you can rely on:

| Tool / Secret | Available as |
|---|---|
| `gh` CLI | `$GH_TOKEN` — full repo, workflow, admin scopes |
| SOLR | `$SOLR_AUTH` — `user:password` format |
| Git | Configured with `user.email` / `user.name` |
| Chrome DevTools | MCP server at `http://127.0.0.1:9222` |
| Filesystem | External-directory MCP for paths outside project |
| Node.js 20 + npm | Installed globally |
| `opencode` | Running you (the big-pickle model) |
| Working directory | AI-Factory-job-seeker-ro-spider checkout root |

**Constraints:**
- All derived repos are under `sebiboga/` org
- EPAM template: `sebiboga/epam-systems-international-srl-nodejs-scraper`
- EPAM is already a GitHub template (`is_template: true`)
- Use the EPAM `--template` flag to get the "Generated from" badge

---

## Decision tree

When execution starts, you receive a company name (and optionally CIF). Follow this:

```
1. Search ANAF to get CIF (if not provided)
2. Search ALL sebiboga repos for company.json containing this CIF
3. Evaluate:
   a. Found repos where company.json contains this CIF → these belong to this company
      - If one matches → UPDATE flow (do NOT recreate, just update the scraper)
      - If none → CREATE flow (fresh from template)
   b. Found repo where company.json contains DIFFERENT CIF → belongs to another company
      - If we're asked for ANOTHER company → ERROR (never overwrite another company's repo)
      - Clean isolation: repos must never be repurposed
4. After create/update, update EPAM's Derived Scrapers table
5. Verify: trigger the scraper, wait for green CI
```

---

## Step 0 — Search ANAF for company details (if CIF not provided)

Use Chrome to navigate to `https://demoanaf.ro/api/search?q=<BRAND>`.

Parse the JSON response. If multiple matches, pick the one whose name best matches the input brand. Extract:
- `cui` → CIF
- `denumire` → legal name
- `adresa` → address

Verify the result by fetching `https://demoanaf.ro/api/company/<CIF>` for full details.

If the API search fails, try the ANAF site directly via Chrome.

---

## Step 1 — CIF-based repo scan

List ALL sebiboga repos and check each for the target CIF:

```bash
gh repo list sebiboga --limit 100 --json name,description | jq -r '.[].name' | while read repo; do
  CIF_IN_REPO=$(gh api "repos/sebiboga/$repo/contents/company.json" -q '.content' 2>/dev/null | base64 -d | jq -r '.summary.cif // .cif // ""')
  if [ "$CIF_IN_REPO" = "$TARGET_CIF" ]; then
    echo "MATCH: $repo"
  fi
done
```

Also check `config/company.json` in each repo for the CIF.

**Important:** The EPAM template repo itself has CIF `33159615`. Always skip updating that one — it IS the template.

---

## Step 2 — CREATE flow (no existing repo for this CIF)

### 2.1 Create repo from EPAM template

```bash
REPO_NAME="<slug>-nodejs-scraper"   # e.g. "company-name-srl-nodejs-scraper"
gh repo create sebiboga/$REPO_NAME \
  --template sebiboga/epam-systems-international-srl-nodejs-scraper \
  --public \
  --description "Scraper automat pentru locurile de muncă <LEGAL_NAME> (CIF: <CIF>) — extrage de pe <CAREER_URL> și publică pe peviitor.ro"
```

**Verify badge:**
```bash
gh api repos/sebiboga/$REPO_NAME -q '.template_repository.full_name'
# Must return: sebiboga/epam-systems-international-srl-nodejs-scraper
```

If no badge → delete and retry (the `--template` flag didn't apply properly).

### 2.2 Clone locally (outside project tree)

```bash
git clone https://github.com/sebiboga/$REPO_NAME.git /tmp/$REPO_NAME
cd /tmp/$REPO_NAME
```

---

## Step 3 — UPDATE flow (repo already exists for this CIF)

If an existing repo already has this CIF, do NOT delete it. Update it in place.

```bash
git clone https://github.com/sebiboga/$REPO_NAME.git /tmp/$REPO_NAME
cd /tmp/$REPO_NAME
```

Then proceed through all the customization steps below (Steps 4–10), changing only what's needed.

---

## Step 4 — Delete stale template artefacts

```bash
rm -f company.json          # ANAF cache with EPAM identity
rm -f docs/jobs.md          # EPAM's last scraped jobs
```

---

## Step 5 — Identify target scraping method

Use Chrome DevTools to navigate to the company's career page and determine:

| Pattern | Signs | Approach |
|---|---|---|
| JSON API | `application/json`, paginated | GET + loop pages |
| HTML / cheerio | Static HTML with job listings | GET, cheerio selectors |
| POST AJAX → HTML | Form submission returns HTML | POST with params, cheerio |
| Teamtailor | `team-tailor` classes, single-page app | GET, cheerio |

Inspect network requests in Chrome DevTools to find API endpoints, required headers, and payload format.

If the company has jobs listed on ANOFM (Agentia Nationala pentru Ocuparea Fortei de Munca), include ANOFM scraping too:

```js
// ANOFM API query by CIF
const payload = {
  current: 1,
  rowCount: 250,
  sort: { created_at: "desc" },
  employer_tax_code: cif
};
// POST to https://www.anofm.ro/api/entity/vw_public_job_posting
```

---

## Step 6 — Configure `config/company.json`

This is the single source of truth. Edit it with the new company's identity:

```json
{
  "cif": "<CIF>",
  "legalName": "<COMPANY LEGAL NAME>",
  "brand": "<Commercial brand>",
  "website": "https://...",
  "careerUrl": "https://...",
  "apiBase": "https://...",
  "apiEndpoint": "<e.g. /api/jobs>",
  "defaultLocation": "<city>",
  "scraperFile": "https://raw.githubusercontent.com/sebiboga/<REPO_NAME>/main/.github/workflows/job-seeker-ro-spider.yml"
}
```

Also overwrite `docs/company.json` with the same content.

---

## Step 7 — Rewrite scraper logic (`index.js`)

Only the `fetchJobs*()` and `parse*Jobs()` functions are company-specific. The rest (mapping, SOLR upsert, markdown) is generic.

Write the scraper for the company's careers site based on your analysis from Step 5.

---

## Step 8 — Update tests

### 8.1 `tests/unit/index.test.js`
Rewrite the parser test block with actual test cases for the new company's parsing logic.

### 8.2 `tests/unit/company.test.js`
Rename `EPAM_ANAF_RECORD` → `<COMPANY>_ANAF_RECORD`, update mocked ANAF data (run `curl https://demoanaf.ro/api/company/<CIF>`).

### 8.3 `tests/unit/solr.test.js`
If CIF is 7 digits, relax the regex check from `/^\d{8}$/` to `/^\d{6,9}$/`.

### 8.4 `tests/integration/workflow.test.js`
Replace `EPAM_CIF` with `companyConfig.cif` imported from `../../config/company.js`.

### 8.5 `tests/e2e/scraper.test.js`
Rewrite fully for new company's scraping method. Increase `beforeAll` timeout to 60s.

### 8.6 `tests/consistency/repo.test.js`
Make brand assertions dynamic: import and use `companyConfig.brand`.

### 8.7 `tests/validate-epam-jobs.js`
Rename to `tests/validate-<brand>-jobs.js` and update references in workflow files.

### 8.8 `tests/package.json`
Rename `name` from `epam-scraper-tests` to `<slug>-scraper-tests`.

---

## Step 9 — Documentation sweep

Use `sed` for bulk rename of "EPAM" → new brand, then MANUALLY VERIFY:

1. **README.md** — restore the "Derivat din EPAM template" link (sed breaks it)
2. **CONTRIBUTING.md** — replace "Deriving a New Scraper" section with slim derived-scraper intro
3. **AGENTS.md** — change "📐 This Repo Is a Template" → "🌱 This Repo Is a Derived Scraper"
4. **ROBOTS.md** — analyze target site's robots.txt
5. **CHANGELOG.md** — replace with fresh `1.0.0` entry
6. **package.json** — set `name` to `scraper-<brand>-ro`, `version` to `1.0.0`
7. **docs/index.html** — replace "EPAM Careers API" / "EPAM Romania" with new brand
8. **delete_request.json** — set `cif` to new CIF
9. **DELETE `AI-DERIVATION-GUIDE.md`** — this file belongs only in the EPAM template

### sed pitfalls (READ BEFORE RUNNING)

```bash
# WRONG — can create identifier pollution:
sed -i 's/epam.com/jobs-newcompany.ro/g' file.js

# BETTER — use word boundaries:
sed -i 's/\bepam\b/newbrand/g' file.js

# Always verify after sed:
grep -rnE '\b[a-z]+\.[a-z]+[a-z]+\b' --include="*.js" .
node -c file.js
npm run test:unit
```

---

## Step 10 — CI configuration

### 10.1 Verify workflow ordering (critical)

Both `.github/workflows/*.yml` workflows must have:

```yaml
- name: Sync with remote
  if: github.event_name != 'pull_request'
  run: git pull origin main --rebase -X theirs
- name: Install dependencies
  run: npm install
```

The `Sync with remote` step MUST come BEFORE `Install dependencies`.

### 10.2 Update workflow label references

```bash
grep -rn "EPAM SYSTEMS\|EPAM Careers" .github/workflows/
# Should not find any matches after sed
```

---

## Step 11 — GitHub repo settings

```bash
# Topics
gh repo edit sebiboga/$REPO_NAME \
  --add-topic job-seeker-ro-spider \
  --add-topic peviitor-ro

# Homepage URL
gh repo edit sebiboga/$REPO_NAME \
  --homepage "https://sebiboga.github.io/$REPO_NAME/"

# Enable GitHub Pages from /docs on main
gh api -X POST repos/sebiboga/$REPO_NAME/pages \
  -f source[branch]=main \
  -f source[path]=/docs

# SOLR_AUTH secret (vital — the scraper will not run without it)
# Use the same value as the EPAM template has
gh secret set SOLR_AUTH --repo sebiboga/$REPO_NAME \
  --body "$SOLR_AUTH"
```

---

## Step 12 — Commit and push

```bash
cd /tmp/$REPO_NAME
git add -A
git commit -m "feat: convert template into <BRAND> scraper

Derived from sebiboga/epam-systems-international-srl-nodejs-scraper."
git push
```

---

## Step 13 — Trigger CI and verify

```bash
gh workflow run job-seeker-ro-spider.yml --repo sebiboga/$REPO_NAME

# Wait for completion (poll with timeout)
for i in $(seq 1 30); do
  STATUS=$(gh run list --repo sebiboga/$REPO_NAME --workflow job-seeker-ro-spider.yml --limit 1 --json conclusion,status -q '.[0]')
  echo "Status: $STATUS"
  echo "$STATUS" | grep -q "conclusion.*success" && break
  echo "$STATUS" | grep -q "conclusion.*failure" && { echo "CI FAILED"; break; }
  sleep 15
done
```

**Watch for known failures:**
- "Sync with remote" fails → CI ordering wrong
- "Run Integration Tests" fails → likely sed mangling
- "Run E2E Tests" times out → increase `beforeAll` to 60s
- "Consistency tests" fail → Pages not deployed or homepage URL not set

---

## Step 14 — Update EPAM template's "Derived Scrapers" list

After CI is green:

1. Clone the EPAM template:
```bash
git clone https://github.com/sebiboga/epam-systems-international-srl-nodejs-scraper.git /tmp/epam-template
cd /tmp/epam-template
```

2. Add a row to the Derived Scrapers table in `README.md`:
```markdown
| [<REPO_NAME>](https://github.com/sebiboga/<REPO_NAME>) | <Legal Name> | <CIF> | <Method> | ✅ Live |
```

3. If `CONTRIBUTING.md` has a "Validated in production" list that mentions derivatives, add it there too.

4. Commit and push:
```bash
git add -A
git commit -m "docs: add <BRAND> to Derived Scrapers list"
git push
```

---

## Known pitfalls (read all before starting)

### P1 — Bulk sed creates mangled identifiers
After any bulk `sed`, run: `grep -rnE '\b[a-z]+\.[a-z]+[a-z]+\b' --include="*.js" .`

### P2 — ANAF returns multiple matches
Always find by CIF, not by array position: `results.find(c => c.cui.toString() === CIF)`

### P3 — SOLR may uppercase brand on store
Use `.toLowerCase()` on both sides of brand comparisons.

### P4 — `lastScraped` format can be ISO or date-only
Use regex: `/^\d{4}-\d{2}-\d{2}(T.*)?$/`

### P5 — E2E timeout on Azure GH runners
Set `beforeAll` timeout to `60000`.

### P6 — CIF length varies (7-9 digits)
Use `/^\d{6,9}$/` not `/^\d{8}$/`.

### P7 — Stale `company.json` cache from template
Always `rm -f company.json` early (Step 4).

### P8 — "Generated from" badge requires `--template` flag
Verify with `gh api ... -q '.template_repository.full_name'` — null means no badge.

### P9 — Forgot to update EPAM template's derived list
Check Step 14 — this is the last step and easy to skip.

### P10 — Stale `docs/jobs.md` from template creates fake URLs
Always `rm -f docs/jobs.md` early (Step 4).

### P11 — SOLR `_version_` causes 409 on re-upsert
Always strip `_version_` from existing jobs before re-upserting:
```js
delete job._version_;
```

### P12 — ANOFM jobs lost after delete-by-CIF crash
Include ANOFM scraping (ANAF CIF filter) in the scraper.

---

## Error handling (autonomous mode)

| Situation | Action |
|---|---|
| ANAF unreachable | Retry 3 times with 2s exponential backoff |
| Repo creation fails | Check if repo name already exists → if yes, use UPDATE flow |
| CI fails on first run | Read the error, fix the issue, commit fix, re-trigger |
| SOLR_AUTH already set | Skip setting — `gh secret set` is idempotent |
| Pages already enabled | Skip — the API call will return a 422 error |
| Can't determine scraping method | Use Chrome DevTools to inspect page structure more carefully |
| Multiple repos matching CIF | Pick the one whose description matches the brand, error if ambiguous |

---

## When to stop and fail (hard errors)

| Situation | Action |
|---|---|
| The input company is the EPAM template itself | Error — EPAM is the template, not a derivation target |
| Found repo with DIFFERENT CIF for input company | Error — can't repurpose another company's repo |
| CI still red after 3 fix attempts | Error — manual intervention needed |
| ANOFM and career page both unreachable | Error — no data source available |

---

## Verification checklist

Before finishing, confirm:

- [ ] Derived repo exists and is public
- [ ] "Generated from" badge visible (`gh api ... -q '.template_repository.full_name'`)
- [ ] `config/company.json` has correct CIF, brand, legalName, URLs
- [ ] Root `company.json` deleted (no stale EPAM cache)
- [ ] `docs/jobs.md` deleted (will be regenerated on first scrape)
- [ ] `AI-DERIVATION-GUIDE.md` deleted (belongs only in EPAM template)
- [ ] All tests pass: `npm test` (unit + integration + consistency)
- [ ] Topics set: `job-seeker-ro-spider`, `peviitor-ro`
- [ ] Homepage URL set to GitHub Pages URL
- [ ] GitHub Pages enabled on `main` `/docs`
- [ ] `SOLR_AUTH` secret set on derived repo
- [ ] CI workflow triggered and green
- [ ] EPAM template's Derived Scrapers table updated
- [ ] No EPAM brand strings remain in derived repo
