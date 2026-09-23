---
name: linkedin-job-match
description: >-
  Review LinkedIn job postings against Scott's resume and produce a
  match-scored table (artifact) with direct apply links. Use when the user
  points at a LinkedIn Jobs search, job alert, or open jobs tab and wants each
  posting screened for fit — "review these jobs", "which of these match me",
  "build a job match board", screening SRE / DevOps / Platform / Cloud roles.
  Encodes the browser-automation flow, the LinkedIn description rate-limit
  behavior and how to avoid it, and the output table format.
---

# LinkedIn Job Match

Screen a LinkedIn Jobs search against the user's resume and hand back a
reviewable, match-scored table with a direct apply link per role. This is a
recurring task (running until the user secures employment), so favor a repeatable,
throttle-safe rhythm over speed.

## Inputs

- **Resume**: `/home/slikens/work/me/frac/AustereCV/fullresume.tex` (read the
  `.tex` — faster and cleaner than the PDF). Re-read it each run; it changes.
- **Search URL**: the user supplies a LinkedIn `jobs/search-results` URL (often
  from a job alert) or says "my open tab". The user is already logged in.

## Daily run & the dedup ledger (so jobs don't repeat day-to-day)

`/home/slikens/work/me/jobs/seen-jobs.json` records every job_id already
surfaced (keyed by id → {company, title, first_seen}). Each run:

1. **Load** it. Build the set of known job_ids.
2. After searching, **exclude** any result whose job_id is already in the ledger
   — that's how recurring reposts (they run for weeks) stop reappearing.
3. Only score/fetch details for the **new** ids.
4. After building the board, **append** the newly-listed ids (with company,
   title, and today's date as `first_seen`) and bump `updated`. Write the file
   back. This is a project data file the user asked for — writing it is expected.

**Recency window:** the user runs this daily and asks for a recent window (e.g.
"24h up to 72h"). `search_jobs` `date_posted` has no exact match, so use
`past_week` and filter by each posting's age from `get_job_details` ("X hours
ago", "Reposted N days ago"). Read "≤ 72h" inclusively (keep < 24h too — they're
the freshest) unless the user insists on a strict band; tag each row's age.
Most promoted reposts are 4–7 days old and fall out of the window automatically.

**Remote rule:** default to remote-only; include on-site/hybrid only if in Salem,
OR. Watch details for residency constraints (state lists, specific time zones,
US-citizen/clearance) and flag them — several "remote" roles exclude Oregon.

## Step 1 — Load the candidate profile

Read `fullresume.tex` and hold the current skill set. As of last run, the profile is:

> **Scott M. Likens — Principal SRE, 22+ yrs.** AWS (EKS, EC2, RDS, EFS, ALB,
> Lambda, SQS, ECR, DMS, CloudWatch, SSM); Kubernetes (EKS/K3S, Helm, Istio,
> Calico); IaC (Terraform, AWS CDK, CDKTF, CloudFormation); Chef, Ansible;
> CI/CD (GitHub Actions, Jenkins, Atlantis); observability (Prometheus,
> Alertmanager, Grafana, Loki, Pyroscope, Sloth SLOs, ServiceNow ITOM, Sensu);
> security/compliance (SOC 2 Type 2, Falco, GuardDuty, CloudTrail, pgAudit,
> mTLS, RBAC, NetworkPolicy); DBs (PostgreSQL, MySQL, MariaDB, MongoDB,
> DocumentDB); Docker, HAProxy, Apache, Nginx, Redis, RabbitMQ, Elasticsearch,
> Packer, Vault. Languages: **Ruby, Bash, TypeScript, Rust proficient; Go
> comfortable; Python scripting**. Leadership up to 6. Long contract history,
> prefers small companies, remote. **No 4-year degree listed** (see caveats).

Always refresh this from the resume — don't trust the cached copy blindly.

## Preferred method — the LinkedIn MCP server (use this first)

**If the `mcp-server-linkedin` tools are available, use them instead of the
browser.** They hit LinkedIn's API and return clean, structured data with the
**full job description and real job IDs** — completely sidestepping the React
#418 SPA crash that breaks browser scraping (see Step 3). This is the reliable
path (proven 2026-09-14).

- `search_jobs(keywords, location, work_type, date_posted, experience_level,
  job_type, easy_apply, sort_by, max_pages)` → returns `job_ids` + a list
  snapshot (titles, companies, comp, location). `work_type`: on_site/remote/
  hybrid; `date_posted`: past_hour/past_24_hours/past_week/past_month. It does
  **not** take a salary filter — approximate, or filter comp yourself after.
- `get_job_details(job_id)` → returns the **complete "About the job"** text
  (requirements verbatim), comp, and company info. Call it per job you want to
  score. Payloads are large and include trailing "more jobs" noise — read the
  `About the job` section and ignore the rest. Parallel calls (several in one
  message) work.
- **Login:** the first call may return "Session expired" and open a login
  window. Ask the user to sign in there (never enter credentials yourself),
  then retry. Map `job_id` → `linkedin.com/jobs/view/<id>/` for apply links.
- Apply the CLAUDE.md blacklist to the results; skip obvious middlemen/gigs
  (staffing agencies, hiring marketplaces, "AI Trainer" data-labeling gigs) and
  report them separately.

Fall back to the browser flow below only if the MCP is unavailable or failing.

## Step 2 — (browser fallback) Open the search and read the list

Load the Chrome tools in one `ToolSearch` (see MCP note). Then
`navigate` a **new** MCP tab to the user's search URL. One
`get_page_text` returns **both** the full result list (titles, companies,
locations, salaries, posted-times) *and* the currently-selected job's full
"About the job". Capture the list now — it's the backbone of the table.

LinkedIn's new Jobs UI has obfuscated class names and job cards are **JS
buttons, not `<a>` links** — you cannot scrape job IDs from `href`s. Use the
`find` tool ("job posting cards in the left results list") to get clickable
`ref_` handles for each card.

## Step 3 — Fetch each posting (why descriptions often won't load)

**Root cause (diagnosed 2026-09-14, revised from the earlier "rate limit"
theory):** the job-detail body frequently fails to render because **LinkedIn's
React app throws a hydration crash — "Minified React error #418"** — visible in
the console at page load (`static.licdn.com/aero-v1/...`). #418 means the
server-rendered HTML didn't match the client render; React unwinds that subtree,
so the detail pane **never mounts its body** (stuck on skeleton loaders) and the
SPA degrades enough that **job-card clicks stop changing `currentJobId`**. So it
presents like a throttle but it's a **client-side app crash**.

**Diagnostics to confirm it (do this instead of guessing):**
- `read_console_messages` with pattern `418|error|hydrat|exception` — if #418 is
  present, it's the hydration crash, not a server gate.
- After a card click, check `location.href` for `currentJobId` — if it didn't
  change, the SPA's click handlers are dead (crash confirmed).
- `read_network_requests` (call it *before* the action that triggers the fetch;
  tracking starts when first called).

**A fresh tab does NOT fix it** when #418 is deterministic — a clean mount hits
the same crash. Re-navigating / new tabs / longer delays don't help once the app
is crashing. **Leading suspect:** hydration mismatches are classically caused by
the DOM being modified before React hydrates — a browser-automation extension
driving the page is a plausible trigger. May be intermittent (a few jobs loaded
early in the very first session before it started crashing).

**When #418 is firing, stop trying to fetch bodies.** The list metadata still
loads reliably — build a role-based (inferred) board from it (Step 4/5) and tell
the user descriptions are unavailable because LinkedIn's SPA is crash-looping.
Verified requirements may be obtainable in a **later session** (or by the user
reading postings in their own non-automated browser).

**Historical note (earlier theory, may still contribute):** rapid, back-to-back
card clicks and chaining multiple detail loads in one `browser_batch` also
correlated with gated bodies. Pacing is still wise, but it does not fix #418.

Do this instead:

- **Process ONE job per `browser_batch`**, paced: `click ref` → `wait 2–3s` →
  extract. Never chain multiple fresh detail loads in a single batch (they time
  out *and* trip the throttle).
- **Capture the description and the job ID together**, on the same visit — don't
  do a separate fast "harvest all IDs" pass. The ID comes from the URL after the
  click: `location.href.match(/currentJobId=(\d+)/)`.
- If a pane is stuck on skeleton, clicking a **neighbor card, waiting, then
  re-clicking the target** sometimes forces a fresh load. If it's globally
  gated, only a **cool-down of several minutes** clears it — don't hammer.

**Dead ends (don't retry these):**
- Standalone `linkedin.com/jobs/view/<id>/` pages are **Premium-walled** when
  hit directly — `get_page_text` returns an upsell, not the description.
- `href`/query-string reads via `javascript_tool` get **blocked** ("Cookie/query
  string data") — extract only bare numeric IDs, never full hrefs.
- Chaining 3 `__exWait` polls in one batch → **batch timeout**.

**Reading a description — keep the probe LIGHT.** Do **not** run a loop that
scans the whole DOM (`querySelectorAll('...div,p...')`) repeatedly — on
LinkedIn's huge results DOM that blocks the main thread and the CDP eval times
out at 45s ("renderer frozen"). That freeze is self-inflicted, not LinkedIn.
Preferred order:

1. **`get_page_text`** (extension-native, no injected JS) — returns the whole
   list *plus* the selected job's "About the job". Token-heavy but safe and
   reliable. This is the default extractor now.
2. If you want just the body, a **single-pass** (non-looping) probe is fine:
   ```js
   (function(){var h=[...document.querySelectorAll('h2,h3,strong,span')]
     .find(e=>e.childNodes.length===1&&e.textContent.trim()==='About the job');
     if(!h)return 'NO_BODY';var c=h;while(c&&c.parentElement&&c.innerText.length<600)c=c.parentElement;
     return JSON.stringify({id:(location.href.match(/currentJobId=(\d+)/)||[])[1],text:c.innerText.slice(0,2000)});})()
   ```
   Run it **once after a wait**, not in a poll loop. `NO_BODY` (or empty) means
   the detail isn't in the DOM yet — gated or still loading; wait longer or move on.

Per job: `[{computer left_click ref}, {computer wait 4-6}, {get_page_text}]`.

**The gate can persist for a long time (learned 2026-09-14).** Once the
detail-fetch gate trips, it can stay active on the same **account + browser
session for 30+ minutes** — a fresh navigation and generous per-job delays do
**not** clear it. When gated, the **list metadata always loads fine; only the
"About the job" body is withheld** (absent from the DOM, not just unrendered).
That's the tell it's a server-side gate, not client overload. Don't burn the
session waiting it out — the realistic clear is a **later session**.

**Harvesting IDs is safe when already gated.** Card clicks still update
`currentJobId` even when bodies won't load, and you need the IDs for the apply
links. Paced ~2s apart, light URL-only reads
(`(location.href.match(/currentJobId=(\d+)/)||[])[1]`), 3 refs per
`browser_batch`, this works fine and is the way to build links for a
role-based (inferred) board.

If descriptions are gated, **tell the user** and offer: (a) resume in a **later
session** (not just minutes later) to get verified text, or (b) build the board
now from title/company/level/comp and mark rows **inferred**. Reuse verified
rows captured in prior runs where the same posting recurs.

## Step 4 — Score each role

**Score the body, not the title (learned 2026-09-18).** LinkedIn titles are
recruiter-optimized and often oversell. Read the "About the role" /
responsibilities and score *those*. If the body opens by naming a different or
less-specialized role than the title — e.g. title "Infrastructure Engineer" but
the body says "we're seeking a Software Engineer" and lists generalist SWE work,
or an "SRE" title whose body is really backend product dev — **trust the body,
down-score to `partial`, and add a prominent flag** ("titled X, but the role is
really Y"). A Tailscale "Infrastructure Engineer" that was actually a generic SWE
role slipped through by title once; don't repeat it.


For each job, write **brief requirements** and a **match verdict** —
`strong` / `partial` / `weak` — with a one-line rationale, judged against the
profile from Step 1. Guidance:

- **Strong**: platform/SRE/DevOps/cloud/K8s/IaC/observability roles — the core.
- **Partial**: role leans heavy **software engineering** (expert Go, distributed-
  systems dev) since his Go is "comfortable"; or demands a language he lacks
  (e.g. **Haskell**); or comp/culture is a poor fit for a Principal (e.g. a
  flat-$100K Crossover/Trilogy listing).
- **Weak**: different discipline — e.g. **offensive security / pentest**
  ("exploitation specialist"); his security is *defensive* (SOC 2, Falco).
- **Caveat flags** (surface, don't down-score): "requires a 4-year degree" (not
  on his resume); government/defense roles (GDIT etc.) usually need **US
  citizenship + clearance**; large-enterprise roles run against his stated
  preference for small companies.

Mark each row's requirements as **verified** (read from the posting) or
**inferred** (from role/title/company).

## Step 5 — Deliver the table (artifact)

Build a filterable HTML **artifact** (load `artifact-design` first). Columns:
**#** · **Role & Company** (title links to `linkedin.com/jobs/view/<id>/`, with
location/comp tags) · **Brief requirements** (+ verified/inferred chip) · **Match**
(colored pill + rationale + any caveat flag). Add stat tiles (total / strong /
partial / weak) and filter chips. Engineering-vernacular treatment fits (IBM Plex
Sans + Mono, slate neutrals, teal accent; green/amber/rose semantic match colors).
A prior version lives in this session's artifacts — reuse its structure.

Then summarize in chat: the strong shortlist, the notable caveats (degree,
clearance), and how many rows are verified vs inferred.

## Cleanup

Close the MCP tab you opened (`tabs_close_mcp`) — it's a separate tab the tool
created, not the user's own. Leave the user's original tab alone.

## MCP note

If Chrome tools are deferred, load them in one `ToolSearch`:
`select:mcp__claude-in-chrome__tabs_context_mcp,mcp__claude-in-chrome__navigate,mcp__claude-in-chrome__computer,mcp__claude-in-chrome__get_page_text,mcp__claude-in-chrome__find,mcp__claude-in-chrome__javascript_tool,mcp__claude-in-chrome__browser_batch,mcp__claude-in-chrome__tabs_close_mcp`
