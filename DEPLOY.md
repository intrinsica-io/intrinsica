# Deploying Intrinsica

Intrinsica runs as **one Cloud Run service**, `intrinsica`, in `europe-west1`. A single container image holds the React build and the FastAPI backend, which serves the page and `/api` from one origin. Every push to `main` is tested, built and deployed by Cloud Build, and every pull request into `main` runs the tests.

## How it works

```
 visitor ──HTTPS──▶ intrinsica.io  (Cloud Run domain mapping, Google-managed cert)
                         │
                         ▼
          Cloud Run service "intrinsica"  (europe-west1, min 1 / max 3)
          ┌──────────────────────────────────────────────┐
          │ one container (root Dockerfile)              │
          │  FastAPI / uvicorn                           │
          │   /api/landing/*, /api/events, /api/health   │
          │   everything else → built React app (dist/)  │
          └───────────┬──────────────────────┬───────────┘
                      │ ADC (runtime SA)     │ HTTPS
                      ▼                      ▼
        Google Sheet 1e4U4…  (Events tab)   Yahoo Finance (yfinance)

 GitHub intrinsica-io/intrinsica
   PR → main   ──▶ Cloud Build trigger "intrinsica-pr"     : tests only
   push main   ──▶ Cloud Build trigger "intrinsica-deploy" : tests → image → Artifact Registry → Cloud Run → smoke check
```

## Configuration

| Var | Value | Notes |
|---|---|---|
| `INTRINSICA_PUBLIC_MODE` | `1` | §2.4 |
| `INTRINSICA_STATIC_DIR` | `/app/static` | Baked into the image (Dockerfile `ENV`), not set by the deploy |
| `CANONICAL_HOST` | `intrinsica.io` | §2.3 |
| `CORS_ORIGINS` | `https://intrinsica.io` | Same-origin already. This blocks other sites' pages |
| `INTRINSICA_EVENTS_SHEET_ID` | `1e4U4roainSuDJPHwsxkVrZezlA2InDPHV1zaQHuCzqY` | R6 |
| `LANDING_SLOW_TTL` | `604800` | 7 days (R8) |
| `LANDING_FAST_TTL` | `14400` | 4 hours (R8) |
| `LANDING_CACHE_MAX_ENTRIES` | `256` | New. Replaces the hard-coded 64 |
| `LANDING_RATE_LIMIT` / `LANDING_RATE_WINDOW_SECONDS` | `20` / `60` | New (§5) |
| `LANDING_SEED_WAIT_SECONDS` | `90` | Startup waits (bounded) for the AAPL/MSFT/NVDA pre-warm, because request-based billing throttles CPU between requests. Unset/0 = fire-and-forget (local dev) |

- The deploy uses `--set-env-vars`, so **the repo is the single source of truth**. An env var edited by hand in the console is overwritten by the next deploy; change config through a PR instead.
- `GOOGLE_SHEETS_ID`, `GOOGLE_SHEETS_CREDS_JSON` and `GOOGLE_SHEETS_CREDS_PATH` are **never set** in production.

To change a value, edit `cloudbuild.yaml` in a PR.

## One-time setup

> Run commands in **Google Cloud Shell** (console → the `>_` icon, top right). It is already authenticated and has `gcloud`, so nothing is needed on Windows. Anything done in a browser UI is marked **[Console]**, **[Spaceship]**, **[Sheets]** or **[GitHub]**.
> Steps 1–5 can be done at any time. Step 6 needs the implementation PR (the pipeline files) merged first.

### Step 1: Create the project and enable APIs
1. **[Console]** Billing → confirm you have an active **billing account**. If not, create one (pay-as-you-go; a card is required).
2. In Cloud Shell:
   ```bash
   export PROJECT_ID=intrinsica-prod      # must be globally unique; add a suffix if taken
   export REGION=europe-west1
   gcloud projects create $PROJECT_ID --name="Intrinsica"
   gcloud billing accounts list           # copy the ACCOUNT_ID
   gcloud billing projects link $PROJECT_ID --billing-account=XXXXXX-XXXXXX-XXXXXX
   gcloud config set project $PROJECT_ID
   gcloud services enable run.googleapis.com cloudbuild.googleapis.com \
     artifactregistry.googleapis.com sheets.googleapis.com iam.googleapis.com \
     secretmanager.googleapis.com
   ```
   Secret Manager is only used internally by the Cloud Build GitHub connection to store its token. The app has no secrets.
3. **Check:** `gcloud services list --enabled` shows all six.

> Cloud Shell sessions forget variables. At the start of each later session, run `export PROJECT_ID=… REGION=europe-west1` and `gcloud config set project $PROJECT_ID` again.

### Step 2: Budget alert
**[Console]** Billing → Budgets & alerts → **Create budget**:
- Scope: project `Intrinsica`.
- Amount: 25 (in your billing currency).
- Thresholds: 50%, 90% and 100%, with email to billing admins.

A budget **alerts** you; it does not stop spending. The real cap is max instances = 3.

### Step 3: Service accounts
```bash
gcloud iam service-accounts create intrinsica-run   --display-name="Intrinsica runtime"
gcloud iam service-accounts create intrinsica-build --display-name="Intrinsica build/deploy"
gcloud iam service-accounts create intrinsica-pr    --display-name="Intrinsica PR checks (tests only)"

export RUN_SA=intrinsica-run@$PROJECT_ID.iam.gserviceaccount.com
export BUILD_SA=intrinsica-build@$PROJECT_ID.iam.gserviceaccount.com
export PR_SA=intrinsica-pr@$PROJECT_ID.iam.gserviceaccount.com

for ROLE in roles/run.admin roles/artifactregistry.writer roles/logging.logWriter; do
  gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member=serviceAccount:$BUILD_SA --role=$ROLE --condition=None
done

gcloud iam service-accounts add-iam-policy-binding $RUN_SA \
  --member=serviceAccount:$BUILD_SA --role=roles/iam.serviceAccountUser

# PR builds only run tests, so they get a separate identity that can do nothing but write
# logs: a pull request can change cloudbuild-pr.yaml, so a PR build must never hold the
# rights to deploy to production.
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member=serviceAccount:$PR_SA --role=roles/logging.logWriter --condition=None
```
**Check:** `gcloud projects get-iam-policy $PROJECT_ID --flatten=bindings --filter="bindings.members:$BUILD_SA" --format="value(bindings.role)"` lists the three roles, and the same command with `$PR_SA` lists only `roles/logging.logWriter`.

### Step 4: Artifact Registry
```bash
gcloud artifacts repositories create intrinsica \
  --repository-format=docker --location=$REGION --description="Intrinsica images"

cat > cleanup.json <<'EOF'
[
  {"name": "keep-recent-10", "action": {"type": "Keep"},
   "mostRecentVersions": {"keepCount": 10}},
  {"name": "delete-older-30d", "action": {"type": "Delete"},
   "condition": {"tagState": "any", "olderThan": "30d"}}
]
EOF
gcloud artifacts repositories set-cleanup-policies intrinsica \
  --location=$REGION --policy=cleanup.json --no-dry-run
```

### Step 5: Connect the Google Sheet
1. **[Sheets]** Open `https://docs.google.com/spreadsheets/d/1e4U4roainSuDJPHwsxkVrZezlA2InDPHV1zaQHuCzqY/edit`.
2. **Share** → add `intrinsica-run@<PROJECT_ID>.iam.gserviceaccount.com` → role **Editor** → untick "Notify people" → **Share**. Google may warn that the address is outside your organisation; that is expected.
3. In the same dialog, confirm **General access = Restricted**.
4. Do **not** create the `Events` tab yourself. The app creates it with the right headers on its first write. If you want to pre-create it, name it exactly `Events` and put `Timestamp | Event | VisitorId | Props` in row 1.

### Step 6: Connect GitHub and create the triggers
*Prerequisite: this repo's `main` contains `cloudbuild.yaml` and `cloudbuild-pr.yaml`.*

1. **[Console]** Cloud Build → **Repositories** → **2nd gen** tab → **Create host connection**:
   - Provider GitHub, region `europe-west1`, name `github-intrinsica`.
   - **Connect** → authorise → **Install in a new account**, choose the **`intrinsica-io`** org → **Only select repositories** → `intrinsica` → Install.
   - If prompted to grant the Cloud Build service agent the Secret Manager Admin role, accept.
2. On the connection → **Link repository** → `intrinsica-io/intrinsica` → Link.
3. **[Console]** Cloud Build → **Triggers** (region `europe-west1`) → **Create trigger**. Create two:

| Field | `intrinsica-deploy` | `intrinsica-pr` |
|---|---|---|
| Event | Push to a branch | Pull request |
| Repository | `intrinsica-io/intrinsica` (2nd gen) | same |
| Branch | `^main$` | Base branch `^main$` |
| Comment control | — | Required except for owners and collaborators |
| Configuration | Cloud Build config file, `/cloudbuild.yaml` | `/cloudbuild-pr.yaml` |
| Substitution | `_SMOKE_URL` = *(leave empty for now)* | — |
| Service account | `intrinsica-build@…` | `intrinsica-pr@…` (logs only, not the deploy account) |

4. **[GitHub]** (optional, recommended) Repo → Settings → Branches → the rule for `main` → *Require status checks* → select the `intrinsica-pr` check.

### Step 7: First deploy
1. **[Console]** Triggers → `intrinsica-deploy` → **Run** → branch `main`.
2. Watch Cloud Build → History. The first build takes about 5–8 minutes; later ones are faster thanks to layer cache.
3. When it is green:
   ```bash
   gcloud run services describe intrinsica --region=$REGION --format="value(status.url)"
   ```
   Open that `https://intrinsica-…run.app` URL. The page loads and the AAPL, MSFT and NVDA tiles render.
4. **Check the sheet:** type a ticker on the page, wait about 30 s, then reload the page or click once more. The queued events are written on the next request after that 30 s (Cloud Run only gives the app CPU while it handles a request), and the `Events` tab appears with the rows.

### Step 8: Verify domain ownership
```bash
gcloud domains verify intrinsica.io
```
This opens Google Search Console. Choose **Domain** property → copy the **TXT** record → **[Spaceship]** Domains → `intrinsica.io` → **DNS / Nameservers** → Add record: type **TXT**, host `@`, value `google-site-verification=…`, TTL default → Save. Back in Search Console → **Verify**. It can take a few minutes; retry if it fails at first.

Use the **same Google account** that runs `gcloud` in Cloud Shell. Only a verified owner can create the mapping.

### Step 9: Map the domain
```bash
gcloud beta run domain-mappings create --service=intrinsica --domain=intrinsica.io     --region=$REGION
gcloud beta run domain-mappings create --service=intrinsica --domain=www.intrinsica.io --region=$REGION
gcloud beta run domain-mappings describe --domain=intrinsica.io --region=$REGION
```
The `describe` output lists `resourceRecords`: 4 × `A` and 4 × `AAAA` for `intrinsica.io`, and 1 × `CNAME` → `ghs.googlehosted.com.` for `www`.

### Step 10: DNS records at Spaceship
**[Spaceship]** `intrinsica.io` → DNS records:
1. **Delete** any existing records on host `@` or `www` of type A, AAAA or CNAME, and any Spaceship parking or URL-redirect records. Keep the TXT verification record.
2. Add the **4 A** records: host `@`, each IP from step 9.
3. Add the **4 AAAA** records: host `@`, each IPv6 address from step 9.
4. Add **1 CNAME**: host `www`, value `ghs.googlehosted.com.`.
5. Save, then watch the mapping status:
   ```bash
   gcloud beta run domain-mappings describe --domain=intrinsica.io --region=$REGION \
     --format="value(status.conditions)"
   ```
   Wait until `Ready` and `CertificateProvisioned` are `True`. That takes 15 minutes to 24 hours; propagation can be checked at dnschecker.org.

### Step 11: Lock down and turn on the smoke check
1. Confirm that `https://intrinsica.io` and `https://www.intrinsica.io` work, and that `www` redirects to the apex.
2. Disable the default URL:
   ```bash
   gcloud run services update intrinsica --region=$REGION --no-default-url
   ```
   Re-open `https://intrinsica.io`; it must still work. The old `run.app` URL now returns 404. If the domain stopped serving, revert with `--default-url` and note it.
3. **[Console]** Triggers → `intrinsica-deploy` → Edit → set `_SMOKE_URL` = `https://intrinsica.io` → Save.

### Step 12: Launch checklist
- [ ] `https://intrinsica.io` shows a valid padlock. `http://intrinsica.io` and `https://www.intrinsica.io` both end at `https://intrinsica.io/`.
- [ ] `https://intrinsica.io/api/health` returns `{"status":"ok"}`.
- [ ] `https://intrinsica.io/api/database` and `/api/analysis` return 404, and `/app` shows the landing page.
- [ ] The AAPL, MSFT and NVDA tiles are instant. A new ticker (e.g. AMZN) evaluates live, and a second view of it is instant.
- [ ] New rows in the sheet's `Events` tab.
- [ ] View source shows the `og:title`, `og:image` and `og:url` tags. The X card preview is checked (post a draft or use a card-preview tool).
- [ ] The `run.app` URL no longer serves.
- [ ] Open a trivial PR → `intrinsica-pr` runs green on it → merge → `intrinsica-deploy` runs green → the change is live.
- [ ] Cloud Run → `intrinsica` → Metrics shows 1 instance idle-warm.

## Operations

- **Deploy:** merge a PR to `main`. Nothing else.
- **Change config** (cache TTLs, rate limit, sheet): edit the env vars in `cloudbuild.yaml` in a PR. Console edits are overwritten on the next deploy.
- **Roll back:**
  ```bash
  gcloud run revisions list --service=intrinsica --region=$REGION
  gcloud run services update-traffic intrinsica --region=$REGION --to-revisions=<GOOD_REVISION>=100
  ```
  Then fix forward with a PR. After it deploys, `gcloud run services update-traffic intrinsica --region=$REGION --to-latest` returns traffic to the newest revision.
- **Logs:** Cloud Run → `intrinsica` → Logs. Save this Logs Explorer query as "Yahoo errors":
  ```
  resource.type="cloud_run_revision"
  resource.labels.service_name="intrinsica"
  (textPayload=~"429|Too Many Requests|YFRateLimit|rate limit" OR severity>=ERROR)
  ```
  Frequent hits mean Yahoo is throttling Cloud Run IPs; revisit §7.
- **Cost:** Billing → Reports, filtered to project Intrinsica.

## Local development

Unchanged: `./start.sh` runs the backend on `:8000` (analyst app included; events stay in memory unless `INTRINSICA_EVENTS_SHEET_ID` is set) and the frontend on `:5173`.

To preview production locally:

```bash
cd frontend && VITE_PUBLIC_MODE=1 npm run build
cd ../backend && INTRINSICA_PUBLIC_MODE=1 INTRINSICA_STATIC_DIR=../frontend/dist python -m uvicorn main:app --port 8080
```

Then rebuild `frontend/dist` without the flag (`npm run build`) so a local build is not left in public mode.
