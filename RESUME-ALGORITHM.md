# AI Factory — Resume and Verify Derivation Algorithm

**Audience:** AI agents (opencode, big-pickle model) executing inside a GitHub Actions workflow.

**Goal:** Resume an incomplete scraper derivation. Given a repo name that was created from the EPAM template but not fully customized (timed out during the main workflow), clone it, verify every artifact, and complete any missing steps.

**Root constraint:** Fully autonomous — NO user interaction. You must decide, act, and verify alone.

---

## Environment

Same as the main ALGORITHM.md environment:

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

**Environment variables set by the workflow:**
- `$REPO_NAME` — the name of the incomplete repo (e.g. `west-co-impex-srl-nodejs-scraper`)
- `$COMPANY_NAME` — optional, user-provided company name
- `$COMPANY_CIF` — optional, user-provided CIF

---

## Overall strategy

```
1. Clone the incomplete repo
2. Read current state (what's customized, what's still EPAM template)
3. Infer company info from repo name or partial config
4. Search ANAF for CIF if not yet known
5. Run the verification checklist
6. For each missing item, execute the corresponding derivation step
7. Trigger CI and verify pass
8. Update EPAM template's Derived Scrapers table
9. Update scrapers index
```

---

## Step 0 — Clone and assess

### 0.1 Clone the target repo

```bash
git clone https://github.com/sebiboga/$REPO_NAME.git /tmp/$REPO_NAME
cd /tmp/$REPO_NAME
```

### 0.2 Determine what state the repo is in

Run these checks and record results:

```bash
# Check what company info already exists
if [ -f config/company.json ]; then
  echo "HAS_CONFIG=true"
  cat config/company.json | jq .
else
  echo "HAS_CONFIG=false"
fi

# Check for stale template artefacts
test -f company.json && echo "HAS_ROOT_COMPANY_JSON=true" || echo "HAS_ROOT_COMPANY_JSON=false"
test -f AI-DERIVATION-GUIDE.md && echo "HAS_DERIVATION_GUIDE=true" || echo "HAS_DERIVATION_GUIDE=false"
test -f docs/jobs.md && echo "HAS_JOBS_MD=true" || echo "HAS_JOBS_MD=false"

# Check if index.js still has EPAM references
grep -q "EPAM\|epam" index.js && echo "INDEX_HAS_EPAM=true" || echo "INDEX_HAS_EPAM=false"

# Check repo settings
gh api repos/sebiboga/$REPO_NAME -q '.has_pages' && echo "HAS_PAGES=true" || echo "HAS_PAGES=false"
gh secret list --repo sebiboga/$REPO_NAME | grep -q SOLR_AUTH && echo "HAS_SOLR_SECRET=true" || echo "HAS_SOLR_SECRET=false"
```

### 0.3 Extract or infer company info

Priority order for getting company info:

1. **From `config/company.json`** — already has the correct data
2. **From user-provided `$COMPANY_NAME` / `$COMPANY_CIF`** — fallback
3. **From repo name** — strip `-nodejs-scraper` suffix, attempt to reconstruct company name
4. **From ANAF API** — search by the brand portion of the repo name

Derive variables:
- `BRAND` — commercial brand (e.g. "WEST CO IMPEX")
- `LEGAL_NAME` — full legal name from ANAF
- `CIF` — fiscal code
- `WEBSITE` — company website
- `CAREER_URL` — careers page
- `SCRAPING_METHOD` — one of JSON API, HTML/cheerio, POST AJAX, Teamtailor, ANOFM

If `config/company.json` already has valid values, extract them and skip Step 2.

---

## Step 1 — Search ANAF for CIF (if not known)

If CIF is not yet known from any source, use Chrome to search ANAF:

```
Navigate to: https://demoanaf.ro/api/search?q=<BRAND>
```

Parse the JSON, pick the best match, then verify with:
```
https://demoanaf.ro/api/company/<CIF>
```

Extract: `cui`, `denumire`, `adresa`.

---

## Step 2 — Determine scraping method (if not configured)

If `config/company.json` already has a valid `careerUrl` and `scrapingMethod`, skip this step.

Otherwise, search Google or known sources for the company's career page. Use Chrome DevTools to inspect the career page and determine the scraping method (see Step 5 in ALGORITHM.md).

If no career page is found, default to ANOFM API scraping (Step 5 of ALGORITHM.md shows the ANOFM POST payload).

---

## Step 3 — Verification checklist

Execute each check. If the check FAILS, execute the corresponding fix step.

### [V1] Stale template artefacts — root `company.json`

**Check:**
```bash
test -f /tmp/$REPO_NAME/company.json && echo "EXISTS" || echo "CLEAN"
```

**Fix if EXISTS:** `rm -f company.json`

### [V2] Stale template artefacts — `docs/jobs.md`

**Check:**
```bash
test -f /tmp/$REPO_NAME/docs/jobs.md && echo "EXISTS" || echo "CLEAN"
```

**Fix if EXISTS:** `rm -f docs/jobs.md`

### [V3] `AI-DERIVATION-GUIDE.md` still present

**Check:**
```bash
test -f /tmp/$REPO_NAME/AI-DERIVATION-GUIDE.md && echo "EXISTS" || echo "CLEAN"
```

**Fix if EXISTS:** `rm -f AI-DERIVATION-GUIDE.md`

### [V4] `config/company.json` has correct company identity

**Check:**
```bash
CIF_IN_CONFIG=$(jq -r '.cif // ""' /tmp/$REPO_NAME/config/company.json 2>/dev/null)
if [ "$CIF_IN_CONFIG" = "$TARGET_CIF" ] && [ -n "$CIF_IN_CONFIG" ]; then
  echo "CORRECT"
else
  echo "WRONG_OR_MISSING"
fi
```

**Fix if WRONG_OR_MISSING:** Write `config/company.json` with:
```json
{
  "cif": "<CIF>",
  "legalName": "<LEGAL_NAME>",
  "brand": "<BRAND>",
  "website": "<WEBSITE>",
  "careerUrl": "<CAREER_URL>",
  "apiBase": "<API_BASE>",
  "apiEndpoint": "<API_ENDPOINT>",
  "defaultLocation": "<CITY>",
  "scraperFile": "https://raw.githubusercontent.com/sebiboga/<REPO_NAME>/main/.github/workflows/job-seeker-ro-spider.yml"
}
```

Also overwrite `docs/company.json` with the same content.

### [V5] `index.js` is customized (no EPAM references)

**Check:**
```bash
grep -q "EPAM\|epam\.com\|epam-systems" /tmp/$REPO_NAME/index.js && echo "HAS_EPAM" || echo "CUSTOMIZED"
```

**Fix if HAS_EPAM:** Rewrite the scraper logic (`fetchJobs*()`, `parse*Jobs()`) for the target company's career site. Use Chrome DevTools to inspect the career page and determine the correct selectors or API endpoints. Follow Step 7 of ALGORITHM.md.

### [V6] Tests are updated

**Check each test file for EPAM references:**

```bash
grep -rl "EPAM\|epam-systems" /tmp/$REPO_NAME/tests/ --include="*.js" 2>/dev/null || echo "ALL_CLEAN"
test -f /tmp/$REPO_NAME/tests/validate-epam-jobs.js && echo "NEEDS_RENAME" || echo "ALREADY_RENAMED"
```

**Fix:** Follow Step 8 of ALGORITHM.md for each test file:
- `tests/unit/index.test.js` — rewrite parser tests
- `tests/unit/company.test.js` — rename `EPAM_ANAF_RECORD` → `<BRAND>_ANAF_RECORD`, update mocked ANAF data
- `tests/unit/solr.test.js` — relax CIF regex if needed
- `tests/integration/workflow.test.js` — replace `EPAM_CIF` with `companyConfig.cif`
- `tests/e2e/scraper.test.js` — rewrite for new scraping method, set `beforeAll` to 60s
- `tests/consistency/repo.test.js` — make brand assertions dynamic
- `tests/validate-epam-jobs.js` → rename to `tests/validate-<brand>-jobs.js`, update workflow references
- `tests/package.json` — rename `name` from `epam-scraper-tests` to `<slug>-scraper-tests`

### [V7] Documentation is customized

**Check:**
```bash
grep -rl "EPAM\|epam-systems-international" /tmp/$REPO_NAME --include="*.md" --include="*.html" --include="*.json" \
  --exclude="node_modules" --exclude="package-lock.json" 2>/dev/null \
  | grep -v "CHANGELOG.md" | grep -v "Generated from" || echo "ALL_CLEAN"
```

**Fix if EPAM strings remain:**
1. Run bulk `sed` with word boundaries: `sed -i 's/\bEPAM\b/<BRAND>/g' file` (carefully, file by file)
2. Restore `README.md` "Derivat din EPAM template" link (sed destroys it)
3. Replace `CONTRIBUTING.md` "Deriving a New Scraper" section with slim derived-scraper intro
4. Update `AGENTS.md` "📐 This Repo Is a Template" → "🌱 This Repo Is a Derived Scraper"
5. Analyze target site's `robots.txt` and write `ROBOTS.md`
6. Replace `CHANGELOG.md` with fresh `1.0.0` entry
7. Set `package.json` `name` to `scraper-<brand>-ro`, `version` to `1.0.0`
8. Update `docs/index.html` — replace "EPAM Careers API" / "EPAM Romania"
9. Set `delete_request.json` `cif` to new CIF

**sed pitfalls:** Always verify after sed: `grep -rnE '\b[a-z]+\.[a-z]+[a-z]+\b' --include="*.js" .`

### [V8] CI workflow is properly configured

**Check:**
```bash
test -f /tmp/$REPO_NAME/.github/workflows/job-seeker-ro-spider.yml && echo "EXISTS" || echo "MISSING"
# Check that "Sync with remote" comes before "Install dependencies"
grep -q "Sync with remote" /tmp/$REPO_NAME/.github/workflows/job-seeker-ro-spider.yml && echo "HAS_SYNC" || echo "NO_SYNC"
# Check no EPAM references in workflow labels
grep -q "EPAM SYSTEMS\|EPAM Careers" /tmp/$REPO_NAME/.github/workflows/*.yml && echo "HAS_EPAM_LABEL" || echo "CLEAN"
```

**Fix if MISSING:** Ensure `.github/workflows/job-seeker-ro-spider.yml` exists with correct ordering. Create it if missing (copy structure from EPAM template, adjusting for new company).

### [V9] GitHub repo settings

**Check:**
```bash
# Topics
TOPICS=$(gh api repos/sebiboga/$REPO_NAME -q '.topics | join(",")')
echo "$TOPICS" | grep -q "job-seeker-ro-spider" && echo "TOPIC_OK" || echo "TOPIC_MISSING"

# Homepage URL
HOMEPAGE=$(gh api repos/sebiboga/$REPO_NAME -q '.homepage')
echo "$HOMEPAGE" | grep -q "$REPO_NAME" && echo "HOMEPAGE_OK" || echo "HOMEPAGE_MISSING"

# GitHub Pages
gh api repos/sebiboga/$REPO_NAME/pages &>/dev/null && echo "PAGES_OK" || echo "PAGES_MISSING"

# SOLR_AUTH secret
gh secret list --repo sebiboga/$REPO_NAME | grep -q SOLR_AUTH && echo "SECRET_OK" || echo "SECRET_MISSING"
```

**Fix if any missing:**
```bash
gh repo edit sebiboga/$REPO_NAME --add-topic job-seeker-ro-spider --add-topic peviitor-ro
gh repo edit sebiboga/$REPO_NAME --homepage "https://sebiboga.github.io/$REPO_NAME/"
gh api -X POST repos/sebiboga/$REPO_NAME/pages -f source[branch]=main -f source[path]=/docs
gh secret set SOLR_AUTH --repo sebiboga/$REPO_NAME --body "$SOLR_AUTH"
```

### [V10] "Generated from" badge

**Check:**
```bash
BADGE=$(gh api repos/sebiboga/$REPO_NAME -q '.template_repository.full_name')
echo "$BADGE" | grep -q "epam-systems" && echo "BADGE_OK" || echo "BADGE_MISSING"
```

**Fix if BADGE_MISSING:** This requires repo deletion and recreation with `--template` flag. Follow Step 2.1 of ALGORITHM.md.

**Important:** Before deleting, save any customized files. After recreation, clone fresh and apply saved customizations.

**But be VERY careful:** Only delete and recreate if no CI has run and no data has been produced. If the repo has any committed customizations, preserve them.

---

## Step 4 — Commit and push any fixes

```bash
cd /tmp/$REPO_NAME
git add -A
if ! git diff --staged --quiet; then
  git commit -m "fix: resume derivation — complete missing steps for <BRAND> scraper

Resumed from sebiboga/AI-Factory-job-seeker-ro-spider via resume workflow."
  git push
fi
```

---

## Step 5 — Trigger CI and verify

```bash
gh workflow run job-seeker-ro-spider.yml --repo sebiboga/$REPO_NAME

for i in $(seq 1 30); do
  STATUS=$(gh run list --repo sebiboga/$REPO_NAME --workflow job-seeker-ro-spider.yml --limit 1 --json conclusion,status -q '.[0]')
  echo "CI Status: $STATUS"
  echo "$STATUS" | grep -q "conclusion.*success" && break
  echo "$STATUS" | grep -q "conclusion.*failure" && { echo "CI FAILED"; break; }
  sleep 15
done
```

If CI fails, read the error, fix, commit, and re-trigger. Maximum 3 fix attempts.

---

## Step 6 — Update EPAM template's Derived Scrapers list

```bash
git clone https://github.com/sebiboga/epam-systems-international-srl-nodejs-scraper.git /tmp/epam-template
cd /tmp/epam-template
```

Check if the repo is already in the Derived Scrapers table. If not, add a row:
```markdown
| [<REPO_NAME>](https://github.com/sebiboga/<REPO_NAME>) | <Legal Name> | <CIF> | <Method> | ✅ Live |
```

Commit and push:
```bash
git add -A
git commit -m "docs: add <BRAND> to Derived Scrapers list"
git push
```

---

## Step 7 — Update the scrapers index

Navigate back to the AI Factory repo:
```bash
cd /home/runner/work/AI-Factory-job-seeker-ro-spider/AI-Factory-job-seeker-ro-spider
```

### 7.1 Check if repo already exists in `docs/scrapers/scrapers.json`

```bash
jq --arg repo "$REPO_NAME" '.scrapers[] | select(.repo == $repo) | .repo' docs/scrapers/scrapers.json
```

If it exists, update the entry. If not, append a new entry.

### 7.2 Add or update entry in `scrapers.json`

```bash
jq --arg repo "$REPO_NAME" \
   --arg url "https://github.com/sebiboga/$REPO_NAME" \
   --arg desc "<LEGAL_NAME> (CIF: <CIF>)" \
   --arg cif "<CIF>" \
   --arg legal "<LEGAL_NAME>" \
   --arg brand "<BRAND>" \
   --arg career "<CAREER_URL>" \
   --arg website "<WEBSITE>" \
   --arg method "<SCRAPING_METHOD>" \
   --arg pages "https://sebiboga.github.io/$REPO_NAME/" \
   '.scrapers = ([.scrapers[] | select(.repo != $repo)] + [{
     "repo": $repo, "url": $url, "description": $desc,
     "cif": $cif, "legalName": $legal, "brand": $brand,
     "careerUrl": $career, "website": $website,
     "scrapingMethod": $method, "isTemplate": false,
     "hasWorkflow": true, "pagesUrl": $pages,
     "derivedFrom": "epam-systems-international-srl-nodejs-scraper",
     "updatedAt": "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"
   }])' docs/scrapers/scrapers.json > docs/scrapers/scrapers.json.tmp \
  && mv docs/scrapers/scrapers.json.tmp docs/scrapers/scrapers.json

jq '.meta.lastUpdated = "'$(date -u +%Y-%m-%dT%H:%M:%SZ)'"' docs/scrapers/scrapers.json \
  > docs/scrapers/scrapers.json.tmp && mv docs/scrapers/scrapers.json.tmp docs/scrapers/scrapers.json
```

### 7.3 Update `docs/scrapers/SCRAPERS.md`

Add or update the markdown table row for this scraper.

### 7.4 Commit and push

```bash
git add -A
git commit -m "feat: add <BRAND> scraper (CIF <CIF>)"
git pull origin main --rebase -X theirs && git push
```

---

## Error handling

| Situation | Action |
|---|---|
| Repo does not exist on GitHub | Error — the repo must exist (created by the main workflow) |
| `config/company.json` missing AND no user-provided company/CIF | Try to infer from repo name. If ambiguous, try ANAF search by extracted name. |
| Career URL not found | Default to ANOFM API scraping |
| CI fails on first run | Read the error, fix the issue, commit fix, re-trigger |
| CI still red after 3 fix attempts | Error — manual intervention needed |
| SOLR_AUTH already set | Skip — `gh secret set` is idempotent |
| Pages already enabled | Skip — will get 422, which is fine |
| "Generated from" badge missing | Delete and recreate with `--template` flag (save any customizations first) |
| Repo has no changes to commit | Skip commit step — nothing was missing |

---

## Verification checklist

After completion, confirm:

- [ ] Derived repo exists and is public
- [ ] "Generated from" badge visible
- [ ] `config/company.json` has correct CIF, brand, legalName, URLs
- [ ] Root `company.json` deleted
- [ ] `docs/jobs.md` deleted
- [ ] `AI-DERIVATION-GUIDE.md` deleted
- [ ] All tests pass: `npm test`
- [ ] Topics set: `job-seeker-ro-spider`, `peviitor-ro`
- [ ] Homepage URL set to GitHub Pages URL
- [ ] GitHub Pages enabled on `main` `/docs`
- [ ] `SOLR_AUTH` secret set on derived repo
- [ ] CI workflow triggered and green
- [ ] EPAM template's Derived Scrapers table updated
- [ ] Scrapers index updated (`docs/scrapers/scrapers.json`)
- [ ] No EPAM brand strings remain in derived repo
