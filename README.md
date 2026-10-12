# Backstage TechDocs: How It Works and How to Run It

This guide explains how our Backstage app shows the documentation of GitHub repos from the `optimum-data-analytics-demo` organization.

---

## 1. What this setup does

Backstage reads our GitHub organization, finds repos that have a `catalog-info.yaml` file, and shows their documentation (the `docs/` folder) in the **Docs** tab.

```
GitHub repos  ->  Catalog (finds catalog-info.yaml)  ->  TechDocs (builds docs/ with MkDocs)  ->  Browser
```

---

## 2. Files each repo needs

| File | Purpose |
|---|---|
| `catalog-info.yaml` | Registers the repo in the Backstage catalog |
| `mkdocs.yml` | Tells MkDocs how to build the docs |
| `docs/*.md` | The real documentation content |

**catalog-info.yaml** (the `name` must be unique for every repo):

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: sonic
  annotations:
    backstage.io/techdocs-ref: dir:.
spec:
  type: service
  lifecycle: production
  owner: group:default/guests
```

**mkdocs.yml**:

```yaml
site_name: sonic
plugins:
  - techdocs-core
```

Note: if you add a `nav:` section, only the pages listed there appear in the menu. Remove `nav:` to show all pages automatically.

---

## 3. Changes made in `app-config.yaml`

### 3.1 GitHub integration (lets Backstage call the GitHub API)

```yaml
integrations:
  github:
    - host: github.com
      token: ${GITHUB_TOKEN}
```

`${GITHUB_TOKEN}` is read from the shell environment, not directly from the `.env` file (see section 5).

### 3.2 GitHub provider (finds repos automatically)

```yaml
catalog:
  providers:
    github:
      orgDiscovery:
        organization: 'optimum-data-analytics-demo'
        catalogPath: '/catalog-info.yaml'
        filters:
          repository: 'sonic'      # use '.*' for all repos
        schedule:
          frequency: { minutes: 30 }
          timeout: { minutes: 3 }
```

| Setting | Meaning |
|---|---|
| `organization` | The GitHub organization to scan |
| `catalogPath` | The file to look for in each repo |
| `filters.repository` | Which repos to include (regex). `'.*'` means all repos, `'^(a\|b)$'` means only `a` and `b` |
| `frequency` | How often the scan repeats. Backstage updates on this schedule, not on every commit |
| `timeout` | Maximum time one scan may take. Increase it when there are many repos |

### 3.3 TechDocs builder

```yaml
techdocs:
  builder: 'local'
  generator:
    runIn: 'local'
  publisher:
    type: 'local'
```

With `runIn: 'local'`, MkDocs must be installed on the machine (see section 5).

### 3.4 Backend plugin (`packages/backend/src/index.ts`)

```ts
backend.add(import('@backstage/plugin-catalog-backend-module-github'));
```

Make sure `backend.start();` appears only **once** at the bottom of the file.

### 3.5 Things to remember

- Remove duplicate entity sources. If `sonic` is registered both from a local file (`examples/sonic.yaml`) and from GitHub, only one version is used.
- The database is `:memory:`, so the catalog is empty after every restart. Wait a few minutes for the first scan.

---

## 4. GitHub token: check that the API call works

Create a token at GitHub -> Settings -> Developer settings -> Personal access tokens.
A fine-grained token needs read access to **Contents** and **Commit statuses** for the repos. A classic token needs the `repo` and `read:org` scopes. If the organization uses SAML SSO, authorize the token for it.

Run these checks in the same terminal where you will run `yarn dev`.

**Step 1: the token is set**

```bash
[ -n "$GITHUB_TOKEN" ] && echo "set (${#GITHUB_TOKEN} chars)" || echo "EMPTY"
```

**Step 2: the token is valid** (expected result: `200`)

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/user
```

**Step 3: the token can see the organization repos** (expected result: `200`)

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/orgs/optimum-data-analytics-demo/repos
```

**Step 4: the token can read the catalog file** (expected result: `200`)

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/optimum-data-analytics-demo/sonic/contents/catalog-info.yaml
```

**Step 5: check the remaining API quota**

```bash
curl -s -H "Authorization: Bearer $GITHUB_TOKEN" https://api.github.com/rate_limit
```

Look at `core.remaining`. A personal token has about 5,000 requests per hour.

| Result | Meaning |
|---|---|
| `200` | Working |
| `401` Bad credentials | Token is empty, expired, or has extra characters |
| `403` | No permission, or SSO is not authorized |
| `404` | Wrong repo name, or the token cannot access that repo |

---

## 5. How to run

**One-time setup (WSL):**

```bash
sudo apt install python3-venv python3-pip
python3 -m venv ~/.venvs/techdocs
source ~/.venvs/techdocs/bin/activate
pip install mkdocs-techdocs-core
```

Create a `.env` file in the project root:

```
GITHUB_TOKEN=ghp_your_token_here
```

Add `.env` to `.gitignore` so the token is never committed:

```bash
grep -qx '.env' .gitignore || echo '.env' >> .gitignore
```

**Every time you start:**

```bash
cd ~/projects/backstage_app/backstage
source ~/.venvs/techdocs/bin/activate
set -a; source .env; set +a
yarn start
```

Then open **http://localhost:3000**.

**Stop:** press **Ctrl+C** in the same terminal. If an old backend is still running:

```bash
ss -ltnp | grep -E ':7007|:3000'
fuser -k 7007/tcp
fuser -k 3000/tcp
```

**Health check** (after about 30 seconds):

```bash
curl -s http://localhost:7007/.backstage/health/v1/readiness
```

Expected result: `{"status":"ok"}`.

---

## 6. Common problems

| Problem | Likely cause and fix |
|---|---|
| `401 Bad credentials` | `GITHUB_TOKEN` is empty in this shell. Load `.env` (section 5) |
| `404` or `503` from the backend | Old backend still running, duplicate `backend.start()`, or a plugin failed. Check the `yarn dev` log |
| Only `index.md` is visible | `nav:` in `mkdocs.yml` lists only that page. Remove `nav:` or add the other pages, then push |
| Second repo is missing | Same `metadata.name` as another repo, filter does not match, or token has no access |
| Changes not visible | Updates follow the scan schedule. Use the refresh icon on the entity or wait for the next scan |

---

## 7. Scaling later (many repos)

For many repos (50 or more), plan these changes:

- Use PostgreSQL instead of `:memory:`.
- Use a GitHub App instead of a personal token.
- Build docs in CI (GitHub Actions) and store them in cloud storage (`builder: 'external'`).
- Use webhooks for real-time updates.
- Use a repository template so every new repo includes `catalog-info.yaml`, `mkdocs.yml`, and `docs/index.md`.
