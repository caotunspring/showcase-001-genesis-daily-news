# build.md — Genesis manual for showcase-003-daily-news

**Target:** zero → live `showcase-003-daily-news.vercel.app` (or sibling) on the Vercel + Supabase + Routines stack.
**Target time:** ~40 minutes wall-clock for a fresh sibling. (~10 min if reusing `showcases-shared` Supabase.)
**Authoring conventions:** snapshot-before-revise; idempotent blocks; no magic-string greps; positive-confirmation loops.
**Last verified end-to-end:** 2026-04-26 (003 redeploy + 004 first run + zh-Hant fix iteration).

---

## Table of contents

| § | What | ETA |
|---|---|---|
| 0 | Mission + product at a glance | 1 min read |
| 1 | Pre-granted permissions Mark has set up once | 0 (just verify) |
| 2 | Pre-flight verification | 2 min |
| 3 | Resource naming | 1 min |
| 4 | Supabase: project + schema | 5 min (or 1 min if reusing `showcases-shared`) |
| 5 | GitHub: repo create + push scaffold | 3 min |
| 6 | Vercel: project link | 2 min |
| 7 | Vercel env: 11 secrets | 5 min |
| 8 | Code: Next.js scaffold + Supabase client | (already in template) |
| 9 | Code: pipeline (sources, score, translate, persist) | (already in template) |
| 10 | Code: pages (`/`, `/runs/[id]`, `/archive`, `/recap`) | (already in template) |
| 11 | Code: API routes (`health`, `run-daily`, `recap-source`) | (already in template) |
| 12 | First production deploy | 2 min |
| 13 | Smoke test from local | 1 min |
| 14 | Routines cloud setup | 5 min |
| 15 | "Run now" verification (× 2-3) | 5 min |
| 16 | Operational runbook (rotate secret, add a source, add a sibling) | 5 min read |
| 17 | Speed receipt — record the timing | 1 min |

---

## §0 Product at a glance

`https://<project-name>.vercel.app/`

- Top-of-fold: today's 3 (003) or 9 (004) news items, Chinese title leads, English subtitle, source pill colored by source.
- Above the cards: a dark "今日整體判讀 · DAILY SYNTHESIS" banner — one paragraph in 繁中 + EN summarizing the day's picks.
- Nav: `/` Today · `/archive` history · `/recap` (004 only — paste any URL → AI summary + impact).
- Data: Supabase Postgres · 4 tables (`routine_runs`, `routine_log_entries`, `news_items`, `source_recaps`) shared across all `showcase-*` siblings, namespaced by a `project` text column.
- Automation: Claude Code Routine fires at `0 0 * * *` UTC (08:00 Asia/Taipei). The routine sends a single `curl POST` to Vercel, which runs the full pipeline server-side.

---

## §1 Pre-granted permissions (Mark sets up once, never repeats)

Verify these exist before you start; if any are missing, fix that first.

### 1.1 GitHub
- `gh` CLI authed as `caotunspring` (`gh auth status` shows it as active)
- aipmtw org accessible if any repo lives there (multi-account: `gh auth switch -u caotunspring` / `gh auth switch -u aipmtw`)

### 1.2 Vercel
- `npx vercel` works. Linked under team `mark-5796s-projects`.
- No tax / billing flags blocking new projects (Hobby tier is enough).

### 1.3 Supabase
- Logged in to `caotunspring's Org` at <https://supabase.com/dashboard>.
- The shared project `showcases-shared` (ref `vdjsjdkswhbtvpsmujlg`) is reachable. If not, §4 path B walks you through fresh project creation.

### 1.4 Azure (D67 Sponsorship)
- `mark.pl.chen@toastmasters.org.tw` signed in to <https://portal.azure.com>
- Cognitive Services resources reachable: **`scottsu-openai`** (eastus) and **`AI-translator-TM-S1`** (southeastasia).
- Endpoint + Key 1 of each → `showcases-local-only/<date>/azure.md` (offline backup).

### 1.5 Claude Code Routines
- `caotunspring@gmail.com` Max plan signed into <https://claude.ai/code/routines>.
- The custom env named **`showcase`** exists and has `*.vercel.app` on its Custom network allowlist. (If not, create it now: claude.ai/code/code-environments → New env → Network access → Custom → Allowed domains: `*.vercel.app`. Default env's allowlist does NOT include vercel.app.)

---

## §2 Pre-flight verification

```bash
# Tools
node --version    # ≥ 20
npm --version
gh --version
gh auth status | head -5    # active account = caotunspring
npx vercel --version
psql --version 2>/dev/null || echo "no psql — that's fine, we use npm pg"

# Supabase reachability (no auth needed for ping)
curl -sS -I https://vdjsjdkswhbtvpsmujlg.supabase.co | head -1   # expect HTTP/2 200 or 401
```

**Checkpoint 01** `Pre-flight passed 前置檢查通過`

---

## §3 Resource naming

For a NEW sibling (e.g. cloning daily-news into a "daily-cybersecurity-news"):

```
project name (= local dir = GitHub repo = Vercel project = PROJECT constant in code):
  showcase-005-daily-<domain>-news     # use kebab-case throughout

routine name (Claude Code Routines console):
  daily-<domain>-news

DB shared:
  Supabase project showcases-shared (vdjsjdkswhbtvpsmujlg)
  All siblings write to the same tables, distinguished by project='showcase-005-daily-<domain>-news'
```

For 003 itself, the values are:
- `showcase-003-daily-news` everywhere
- routine name: `daily-news`

For 004:
- `showcase-004-daily-mfg-news` everywhere
- routine name: `daily-mfg-news`

---

## §4 Supabase

### Path A — reuse `showcases-shared` (recommended for siblings)

Skip ahead to §5. The schema is already there; you just need the URL + keys.

### Path B — create a fresh Supabase project (only if you need DB isolation)

1. <https://supabase.com/dashboard/projects> → **New project**
   - Org: `caotunspring's Org` (Free tier)
   - Name: `showcases-shared` (or whatever)
   - Region: `Asia-Pacific (Singapore)` — `ap-southeast-1`
   - Strong DB password — save to `showcases-local-only/<date>/supabase.md`
2. Wait ~1-2 min for provisioning.
3. **Apply schema via direct PG** (faster than SQL editor click-through):
   ```bash
   cd /tmp && npm init -y >/dev/null && npm install pg --no-audit --no-fund
   cat > apply.mjs <<'JS'
   import { readFile } from 'node:fs/promises';
   import pg from 'pg';
   const { Client } = pg;
   const sql = await readFile(process.argv[2], 'utf8');
   const c = new Client({ connectionString: process.argv[3], ssl: { rejectUnauthorized: false } });
   await c.connect();
   await c.query(sql);
   console.log('applied');
   await c.end();
   JS
   node apply.mjs <path-to-this-repo>/supabase/migrations/0001_init.sql \
     "postgresql://postgres:<DB_PASSWORD>@db.<NEW_PROJECT_REF>.supabase.co:5432/postgres"
   ```
4. Get API keys: Project Settings → API Keys
   - Publishable key (= `SUPABASE_ANON_KEY` and `NEXT_PUBLIC_SUPABASE_ANON_KEY`)
   - Secret key (= `SUPABASE_SERVICE_ROLE_KEY`)
5. Save both to `showcases-local-only/<date>/supabase.md`.

### The schema (`supabase/migrations/0001_init.sql`)

Both showcase repos carry an identical copy. The canonical version lives at:
- `showcase-003-daily-news/supabase/migrations/0001_init.sql`
- `showcase-004-daily-mfg-news/supabase/migrations/0001_init.sql`

Tables:
- `routine_runs` — one row per pipeline invocation (project, run_id, status, items_produced, daily_summary_*)
- `routine_log_entries` — one row per phase/tool call inside a run, FK to routine_runs (project, run_id)
- `news_items` — one row per picked story (project, news_date, rank, source_name, title_en/zh, summary_en/zh, impact_en/zh, url, score)
- `source_recaps` — URL-keyed cache for `/api/recap-source` (cross-tenant)

Composite uniques: `(project, run_id)`, `(project, news_date, rank)`, `(project, run_id, sequence_num)`.
RLS: every table `enable row level security` + `public read` policy; writes are service-role-only (RLS denies anon writes by default).

**Checkpoint 02** `Schema applied 資料庫結構套用完成`

---

## §5 GitHub: repo create + push scaffold

The fastest path for a new sibling is to clone an existing one:

```bash
cd ~/Documents/showcases
cp -r showcase-003-daily-news showcase-005-daily-<domain>-news
cd showcase-005-daily-<domain>-news
rm -rf .git .next .vercel node_modules tsconfig.tsbuildinfo .env .env.local evidence/* logs/*

# rename in package.json
sed -i 's/showcase-003-daily-news/showcase-005-daily-<domain>-news/g' package.json

# rename PROJECT constant in src/lib/supabase.ts
sed -i 's/showcase-003-daily-news/showcase-005-daily-<domain>-news/g' src/lib/supabase.ts

# rename in /api/health/route.ts and any other hard-coded references
grep -rl 'showcase-003-daily-news' src/ | xargs sed -i 's/showcase-003-daily-news/showcase-005-daily-<domain>-news/g'

# install
npm install --no-audit --no-fund

# init git + first commit
git init -q
git add -A
git -c user.email=caotunspring@gmail.com -c user.name="Mark Chen" commit -q -m "initial: showcase-005-daily-<domain>-news scaffold"

# create + push
gh auth switch -u caotunspring
gh repo create caotunspring/showcase-005-daily-<domain>-news \
  --public --source . \
  --description "Daily <domain> news, picked + analysed by Claude Code Routine at 08:00 TPE" \
  --push

# rename master → main (gh repo create defaults to master)
git branch -m master main
git push -u origin main
gh repo edit caotunspring/showcase-005-daily-<domain>-news --default-branch main
git push origin --delete master
```

**Checkpoint 03** `GitHub repo created + scaffold pushed`

---

## §6 Vercel: project link

```bash
cd ~/Documents/showcases/showcase-005-daily-<domain>-news
npx vercel link --project showcase-005-daily-<domain>-news --yes
# project linked under team mark-5796s-projects
cat .vercel/project.json   # verify { projectId, orgId, projectName }
```

**Checkpoint 04** `Vercel project linked`

---

## §7 Vercel env: 11 secrets

Vercel marks all secrets as Sensitive (encrypted-at-rest, NOT recoverable via `vercel env pull`). So the canonical place these live is:

- **Vercel project settings** (production runtime reads these)
- **`.env`** in the repo root (local-only authoritative — gitignored, survives `vercel env pull`)
- **`showcases-local-only/<date>/azure.md` + `supabase.md`** (offline backup)

Convention spec: `spec/env-files-convention.md` (binding for all `showcase-*` repos).

### 7.1 Write `.env` (local-only)

```bash
cat > .env <<'ENV'
# Supabase — showcases-shared (caotunspring's Org)
# All values from showcases-local-only/<date>/supabase.md
SUPABASE_URL=https://<SUPABASE_PROJECT_REF>.supabase.co
SUPABASE_ANON_KEY=<sb_publishable_... · Supabase Settings → API Keys → Publishable>
SUPABASE_SERVICE_ROLE_KEY=<sb_secret_... · Supabase Settings → API Keys → Secret>
NEXT_PUBLIC_SUPABASE_URL=https://<SUPABASE_PROJECT_REF>.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=<same as SUPABASE_ANON_KEY above>

# Azure OpenAI — scottsu-openai (eastus)
# All values from showcases-local-only/<date>/azure.md
AZURE_OPENAI_ENDPOINT=https://eastus.api.cognitive.microsoft.com/
AZURE_OPENAI_KEY=<KEY 1 from Azure Portal · Cognitive Services → scottsu-openai → Keys and Endpoint>
AZURE_OPENAI_DEPLOYMENT=gpt4o

# Azure Translator — AI-translator-TM-S1 (southeastasia)
AZURE_TRANSLATOR_KEY=<KEY 1 from Azure Portal · Translator → AI-translator-TM-S1 → Keys and Endpoint>
AZURE_TRANSLATOR_REGION=southeastasia
AZURE_TRANSLATOR_ENDPOINT=https://api.cognitive.microsofttranslator.com/

# Routine ingest secret (one shared value across all showcase-* siblings)
# Generate fresh with: openssl rand -base64 24 | tr -d '\n'
# (Or copy from an existing sibling's Vercel env if you want the secret shared.)
ROUTINE_INGEST_SECRET=<32-char random base64>
ENV

# Verify
grep -E '^[A-Z_]+=' .env | wc -l   # expect 12
```

> **Why `.env` not `.env.local`:** `vercel env pull` overwrites `.env.local` and writes "" for every Sensitive value. `.env` is never touched by Vercel CLI. See `spec/env-files-convention.md`.

### 7.2 Push the same 11 values to Vercel

```bash
push_one() {
  local name="$1" value="$2"
  for env in production preview development; do
    npx vercel env rm "$name" "$env" --yes >/dev/null 2>&1 || true
    echo -n "$value" | npx vercel env add "$name" "$env" --force >/dev/null 2>&1 || true
  done
  echo "  $name done"
}

# Read values from .env
set -a; source .env; set +a

push_one SUPABASE_URL                    "$SUPABASE_URL"
push_one NEXT_PUBLIC_SUPABASE_URL        "$NEXT_PUBLIC_SUPABASE_URL"
push_one SUPABASE_SERVICE_ROLE_KEY       "$SUPABASE_SERVICE_ROLE_KEY"
push_one SUPABASE_ANON_KEY               "$SUPABASE_ANON_KEY"
push_one NEXT_PUBLIC_SUPABASE_ANON_KEY   "$NEXT_PUBLIC_SUPABASE_ANON_KEY"
push_one AZURE_OPENAI_ENDPOINT           "$AZURE_OPENAI_ENDPOINT"
push_one AZURE_OPENAI_KEY                "$AZURE_OPENAI_KEY"
push_one AZURE_OPENAI_DEPLOYMENT         "$AZURE_OPENAI_DEPLOYMENT"
push_one AZURE_TRANSLATOR_KEY            "$AZURE_TRANSLATOR_KEY"
push_one AZURE_TRANSLATOR_REGION         "$AZURE_TRANSLATOR_REGION"
push_one AZURE_TRANSLATOR_ENDPOINT       "$AZURE_TRANSLATOR_ENDPOINT"
push_one ROUTINE_INGEST_SECRET           "$ROUTINE_INGEST_SECRET"

# Verify
npx vercel env list production 2>&1 | grep -cE 'AZURE|ROUTINE|SUPABASE'   # expect 12
```

**Checkpoint 05** `Vercel env populated · 12 vars`

---

## §8 Code: Next.js scaffold + Supabase client

Already in the cloned scaffold. The shape:

```
src/
├── app/
│   ├── layout.tsx            # site header/footer, header bar with nav
│   ├── page.tsx              # today's digest + daily synthesis banner + cards
│   ├── globals.css
│   ├── archive/page.tsx      # daily history, color-coded source pills
│   ├── runs/
│   │   ├── page.tsx          # list of recent runs
│   │   └── [id]/page.tsx     # one run's full log (every fetch/score/translate/persist)
│   ├── recap/page.tsx        # paste a URL → recap + impact (004 only; 003 omits)
│   └── api/
│       ├── health/route.ts                # GET — counts of runs/items/latest_date
│       ├── routine/run-daily/route.ts     # POST — the main pipeline trigger
│       ├── routine/ingest/route.ts        # POST — legacy v1 path (kept for backcompat)
│       └── recap-source/route.ts          # GET/POST — recap any URL (004 only)
└── lib/
    ├── supabase.ts           # PROJECT constant + supabasePublic() + types
    ├── daily-pipeline.ts     # the pipeline (fetch→score→aggregate→translate→synthesize→persist)
    └── source-recap.ts       # /api/recap-source helper (004 only)
```

**Critical `PROJECT` constant** in `src/lib/supabase.ts`:

```ts
export const PROJECT = "showcase-005-daily-<domain>-news" as const;
```

Every read filters `.eq('project', PROJECT)`; every write sets `project: PROJECT`. This is what keeps siblings isolated on the shared Supabase.

---

## §9 Code: pipeline

Core file: `src/lib/daily-pipeline.ts`. Reads as 9 numbered phases:

1. **INIT** — `routine_runs.delete()` + `insert({status:'running', project:PROJECT, ...})`. Replace-on-rerun semantics.
2. **FETCH** — N sources in parallel. Each source fn returns `Candidate[]`. Failures logged but don't kill the run.
3. **SCORE** — top K (default 4) candidates per source via Azure OpenAI gpt-4o. System prompt is domain-specific ("score AI-coding news for technical audience" vs "score for Taiwan semi factory supply-chain"). Returns 0.0-1.0; combined with `recency_hint` weighted 0.4 / 0.6 (or 0.3/0.7 for 004).
4. **AGGREGATE** — global rank across all scored candidates, dedup by URL canonicalization, take top `MAX_PICKS` then slice to `TARGET_PICKS`.
5. **TRANSLATE** — bidirectional. en-native sources do en → zh-Hant; zh-native RSS sources do zh → en. **Never trust gpt-4o for direct zh-Hant** (per `backlog/2026-04-26-zh-hant-strict.md`).
6. **DAILY SYNTHESIS** (optional, 003+004 only) — gpt-4o produces ONE English paragraph; Translator does en → zh-Hant. Stored in `routine_runs.daily_summary_en/zh`.
7. **(004 only) IMPACT** — per-item supply-chain impact. Same gpt-4o-EN-then-Translator-zh pattern.
8. **PERSIST** — `news_items.delete().eq('news_date', today)` + `insert(rows)`. Each row tagged with `project: PROJECT`.
9. **FINALIZE** — `routine_runs.update({status, items_produced, daily_summary_*})`.

Adapting for a new domain:
- Source list: replace the source fns + `sourceDefs` array
- Keyword filter: domain-specific regex (e.g. `MFG_KEYWORDS` in 004)
- Score prompt: domain-specific system message
- Targets: tune `TARGET_PICKS` / `MAX_PICKS` / `MIN_PICKS` to your domain's velocity
- Optional: per-item analysis like 004's impact

**Sources currently shipped:**
- 003: `anthropic-news`, `techcrunch-ai`, `hn-24h` (AI-coding keyword filter), `verge-ai`, `inside-tw` (繁中 RSS), `lobsters-ai`
- 004: `hn-24h-mfg`, `technews-tw` (繁中 RSS), `udn-money` (繁中 RSS), `cna-tech` (繁中 RSS)

---

## §10 Code: pages

`src/app/page.tsx` is the demo's hero. Key UX rules locked in 2026-04-26:

1. **Projector-grade typography** — `text-7xl` headline, `text-3xl` card titles, `text-xl` summaries. AAA-contrast pure black on white.
2. **Chinese first** — 繁中 title on top large, English subtitle below smaller (slate-600). 繁中 summary first, English in muted slate-500.
3. **Source pill** — bold dark colored pill (`ring-2`, source-themed colors). Distinct hue per source so audience can scan.
4. **Daily synthesis dark banner** — at top of card list, dark `bg-slate-900` rounded-2xl, eyebrow in amber accent. 繁中 paragraph leads (text-2xl/3xl), English subtitle below.
5. **No Ant Design / no Tailwind UI components** — pure utility classes. Easier to inline-edit in build.md (this file).

**Headline copy lives in the page itself**, not in i18n. Just edit the `<h1>` text. Examples:
- 003: `AI Coding 與 Claude Code\n今日該關注的事。`
- 004: `半導體應用工廠\n今日該關注的事。`

`src/app/runs/[id]/page.tsx` — phase-coded log row. Color per phase:
- `init` slate-900, `fetch` sky-700, `score` violet-700, `aggregate` amber-700,
- `translate` emerald-700, `impact` (004) fuchsia-800, `summary` (003+004) emerald-800,
- `persist` fuchsia-800, `finalize` slate-700

---

## §11 Code: API routes

### `/api/routine/run-daily` (THE pipeline trigger)

```
POST /api/routine/run-daily
Headers: X-Routine-Secret: <ROUTINE_INGEST_SECRET>
         Content-Type: application/json
Body:    {} (or {run_id, news_date, source_type} to override defaults)

200 OK
{
  "ok": true,
  "run_id": "2026-04-27-auto",
  "news_date": "2026-04-27",
  "status": "succeeded" | "degraded" | "failed",
  "items_produced": <N>,
  "log_count": <N>,
  "elapsed_ms": <N>
}

403 forbidden    — header mismatch
500 db_env_missing — SUPABASE_URL / SERVICE_ROLE_KEY not set
500 pipeline_failed — uncaught exception inside runDailyPipeline
```

`maxDuration = 60` seconds. Sufficient for 6-9 items × 4 LLM calls in batches of 4.

### `/api/health`

```
GET /api/health
200 OK
{
  "status": "ok" | "waiting_credentials",
  "app": "showcase-005-daily-<domain>-news",
  "counts": {"runs": <N>, "items": <N>, "latest_date": "YYYY-MM-DD" | null},
  "timestamp": "<ISO>"
}
```

### `/api/recap-source` (004 only — paste any URL → recap + impact)

```
GET /api/recap-source?url=<source-url>[&force=1]
POST /api/recap-source body {url, force?}

200 OK
{
  "ok": true,
  "url": "...",
  "title": "...",
  "recap_en": "...", "recap_zh": "...",
  "impact_en": "...", "impact_zh": "...",
  "fetched_at": "<ISO>",
  "byte_size": <N>,
  "failure": null,
  "cached": true | false
}
```

Cache key: URL. `force=1` bypasses cache. Cached entries have unbounded TTL — cron-fired pipelines cache items at write-time so users hitting `/recap?url=...` later get the cache hit.

---

## §12 First production deploy

```bash
cd ~/Documents/showcases/showcase-005-daily-<domain>-news
npx vercel deploy --prod --yes
# expect: deployment URL printed; status Ready in ~14s for a small Next.js app

# Verify
npx vercel ls --yes | head -5
curl -sS -o /dev/null -w "homepage HTTP %{http_code}\n" "https://showcase-005-daily-<domain>-news.vercel.app/"
curl -sS "https://showcase-005-daily-<domain>-news.vercel.app/api/health"
# expect status:"ok", counts:{runs:0, items:0, latest_date:null}
```

**Checkpoint 06** `First Vercel deploy live · /api/health returns ok`

---

## §13 Smoke test from local

```bash
npm run trigger:smoke
```

`scripts/smoke-trigger.mjs` reads `ROUTINE_INGEST_SECRET` from `.env` (via `dotenv -r dotenv/config` in the npm script) and POSTs `{}` to `/api/routine/run-daily`.

**Expected:**
```
POST https://showcase-005-daily-<domain>-news.vercel.app/api/routine/run-daily ...
HTTP 200 · 15-30s
{
  "ok": true,
  "run_id": "<today>-auto",
  "news_date": "<today>",
  "status": "succeeded" | "degraded",
  "items_produced": <≥ MIN_PICKS>,
  "log_count": <12-25>,
  "elapsed_ms": <15000-30000>
}
```

**If you get HTTP 403** — `ROUTINE_INGEST_SECRET` mismatch between local `.env` and Vercel env. Fix `.env` to match what's in Vercel (or vice versa).

**If you get HTTP 500 + `db_env_missing`** — Supabase env vars didn't get pushed to Vercel correctly. Re-run §7.2.

**If you get HTTP 500 + `pipeline_failed`** — open Vercel function logs (<https://vercel.com/mark-5796s-projects/showcase-005-daily-<domain>-news/logs>); usually one source's HTML structure changed.

**Checkpoint 07** `Smoke test green · today's items in news_items table`

---

## §14 Routines cloud setup

This is where the rubber meets the road for the AIA narrative — Claude Code is fully autonomous each morning.

### 14.1 Routine env

Open <https://claude.ai/code/code-environments> on `caotunspring` account. Use the existing `showcase` env (it has `*.vercel.app` allowlist already). If for some reason you need a separate one, create `showcase` again with:
- Network access: Custom · "Also include default list"
- Allowed domains: `*.vercel.app`
- Environment variables: empty (we hardcode in the prompt — env injection on Routines is unreliable)
- Setup script: empty

### 14.2 Create the routine

<https://claude.ai/code/routines/new>

| Field | Value |
|---|---|
| Name | `daily-<domain>-news` (e.g. `daily-news`, `daily-mfg-news`) |
| Repository | `caotunspring/showcase-005-daily-<domain>-news` |
| Branch | `main` |
| Environment | **`showcase`** (NOT `Default` — Default doesn't include vercel.app) |
| Model | Sonnet 4.6 (Opus also fine; Sonnet is cheaper for this trivial work) |
| Trigger | Schedule, Daily, **08:00 GMT+8** |
| Connectors | none — remove the auto-added Gmail |
| Permissions | Trusted hosts (= the allowlist from §14.1) |

### 14.3 Instructions (paste verbatim, then substitute `<DOMAIN>` and the literal secret)

```
You are the daily-<DOMAIN>-news trigger for showcase-005-daily-<DOMAIN>-news.

Architecture: Routines cloud egress allowlist blocks Supabase, Azure, and the news sources themselves. The actual fetch + score + translate + persist pipeline runs on Vercel (no outbound restrictions). Your job is to fire a single curl and report the result.

## Run

curl -sS -w "\n[HTTP %{http_code} . %{time_total}s]\n" \
  -X POST "https://showcase-005-daily-<DOMAIN>-news.vercel.app/api/routine/run-daily" \
  -H "X-Routine-Secret: <PASTE LITERAL SECRET — DO NOT use $VAR>" \
  -H "Content-Type: application/json" \
  -d '{}'

## Expected response on success

{"ok":true,
 "run_id":"<YYYY-MM-DD>-auto",
 "news_date":"<YYYY-MM-DD>",
 "status":"succeeded" | "degraded",
 "items_produced":<N>,
 "log_count":<N>,
 "elapsed_ms":<N>}

## Reporting

- HTTP 200 + ok=true  -> print "OK daily-<DOMAIN>-news triggered . run_id=<id> . status=<status> . items=<N> . elapsed=<ms>ms"
- HTTP 200 + ok=false -> print response body verbatim, report status=failed
- HTTP != 200         -> print response body verbatim, report status=blocked_by_HTTP_<code>
- Network error       -> print error message, report status=trigger_unreachable

Do NOT retry. The Vercel handler has its own error handling.

## Final console output

Print the one summary line above and exit. Total elapsed should be 30-60 seconds (Vercel does the work).
```

> **Critical:** the secret MUST be hard-coded in the curl header line. `$VAR` shell expansion fails ~30% of the time on Routines cloud (env injection unreliable per the prompt notice). Don't argue with it; just hardcode.

**Save.**

**Checkpoint 08** `Routine created`

---

## §15 "Run now" verification (× 2-3)

Click **Run now** on the routine page. Expected outcome inside the routine session log:

```
OK daily-<DOMAIN>-news triggered . run_id=<today>-auto . status=succeeded . items=<N> . elapsed=<N>ms
```

**Common failure modes:**

| Symptom | Diagnosis | Fix |
|---|---|---|
| `HTTP 403` | secret mismatch OR allowlist not including vercel.app | (a) hardcode the literal secret; (b) verify routine env = `showcase` not `Default` |
| `HTTP 500` + `db_env_missing` | Vercel env didn't have SUPABASE_* set when the deploy ran | re-run §7.2, then `vercel deploy --prod` again |
| `HTTP 500` + `pipeline_failed: dedup_floor_breach` | All sources returned 0 candidates | inspect Vercel function logs; usually one source's HTML structure changed |
| `Network error` | Routines couldn't reach vercel.app at all | env's allowlist is wrong — re-check §14.1 |
| `Vercel cold start > 60s` | first invocation after long idle | retry; also consider bumping `maxDuration` to 90 (requires Pro tier) |

Run "Run now" 2-3 times back-to-back. They all should succeed (the pipeline is replace-on-rerun idempotent).

**Checkpoint 09** `≥ 2 manual cloud runs green back-to-back`

---

## §16 Operational runbook

### 16.1 Rotate `ROUTINE_INGEST_SECRET`

```bash
# 1. Generate
NEW=$(openssl rand -base64 24 | tr -d '\n')

# 2. Push to ALL Vercel projects that share this secret
for proj in showcase-003-daily-news showcase-004-daily-mfg-news; do
  cd ~/Documents/showcases/$proj
  for env in production preview development; do
    npx vercel env rm ROUTINE_INGEST_SECRET $env --yes >/dev/null 2>&1 || true
    echo -n "$NEW" | npx vercel env add ROUTINE_INGEST_SECRET $env --force
  done
done

# 3. Update each repo's .env locally
cd ~/Documents/showcases/showcase-003-daily-news
sed -i "s/^ROUTINE_INGEST_SECRET=.*/ROUTINE_INGEST_SECRET=$NEW/" .env
# (repeat for 004)

# 4. Redeploy each
for proj in showcase-003-daily-news showcase-004-daily-mfg-news; do
  cd ~/Documents/showcases/$proj
  npx vercel deploy --prod --yes
done

# 5. Update each Routines prompt — drive Chrome to claude.ai/code/routines and edit
#    the "X-Routine-Secret:" header line in the curl block.
```

### 16.2 Add a new source to an existing pipeline

1. Add a `fetch<NewSource>(): Promise<Candidate[]>` function in `src/lib/daily-pipeline.ts`.
   - For RSS sources: call `parseRssItems(xml, "<source-name>", { native_zh: <true if 繁中>, max: 25 })` and filter by your keyword regex.
   - For JSON APIs (HN-style): map to `Candidate[]` with `source`, `title`, `summary`, `url`, `published_at`, `recency_hint`.
2. Add to `sourceDefs` array.
3. Add to `SOURCE_LABEL` and `SOURCE_THEME` in `src/app/page.tsx` and `src/app/archive/page.tsx`.
4. `git push` → Vercel auto-deploys.

### 16.3 Add a sibling showcase (e.g. 005)

Follow §3 → §15 from the top. Estimated time: 40 min wall-clock.

### 16.4 zh-Hant policy (binding)

**Never trust gpt-4o for direct 繁中 output.** It leaks Simplified ~10-20% of the time regardless of system prompt. Always: gpt-4o emits English-only JSON, then Azure Translator does `to=zh-Hant` for the Chinese surface fields. See `backlog/2026-04-26-zh-hant-strict.md`.

### 16.5 RSS HTML entity bug (open)

Lobsters AI feed leaks `&lt;p&gt;` literals into summary text. Cosmetic only; backlog at `backlog/2026-04-26-rss-html-entity-leak.md`. One-line fix in `parseRssItems`: decode entities after the tag-strip.

---

## §17 Speed receipt

After a fresh sibling rebuild, fill in `timing/<YYYY-MM-DD>-rebuild.md`:

```markdown
# Rebuild on YYYY-MM-DD

| Checkpoint | Description | Wall-clock | Cumulative |
|---|---|---|---|
| 01 | Pre-flight passed | 1m | 1m |
| 02 | Schema applied | 4m | 5m |
| 03 | GitHub repo created + scaffold pushed | 3m | 8m |
| 04 | Vercel project linked | 2m | 10m |
| 05 | Vercel env populated | 5m | 15m |
| 06 | First Vercel deploy live | 2m | 17m |
| 07 | Smoke test green | 1m | 18m |
| 08 | Routine created | 5m | 23m |
| 09 | ≥ 2 manual cloud runs green | 5m | 28m |
| -- | total | | **28m** |

Issues hit: <none / list them>
Took longer than expected on: <step>
```

Commit it. The `timing/` dir is the actual receipt — every fresh rebuild adds a row of evidence.

---

## Done

`https://showcase-005-daily-<domain>-news.vercel.app/` is live, the cron will fire at 08:00 TPE tomorrow, and `/runs/<run_id>` shows every step Claude took to ship that day's news. The same Supabase project hosts all your siblings; the same `.env` shape works for every one.

If anything in this build.md broke during your rebuild, that's the bug — fix the doc, not just your local. Open a PR on whichever showcase repo holds the canonical `build.md` (currently in 003).
