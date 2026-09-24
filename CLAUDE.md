# Job search — project guidance

Working directory for a remote SRE / DevOps / Platform / Cloud job hunt. When
reviewing job listings here (e.g. via the `linkedin-job-match` skill), apply the
rules below.

> **Forking this?** These rules encode one person's preferences (Scott's). Swap in
> your own keywords, blacklist, and flags. Keep anything personal — active
> interviews, recruiter contacts, employers you're avoiding by name — in
> `CLAUDE.local.md`, which is git-ignored and auto-loaded alongside this file.

## Excluded sources (blacklist)

Do **not** include postings from these companies in any job board, table, or
shortlist. They are job-board reposters / aggregators, not the actual employer —
the "company" is the middleman, so the listing can't be applied to directly and
duplicates a real posting elsewhere. LinkedIn offers no way to filter them out,
so we do it here.

**The specific named blocklist — which companies to drop, and why — lives in
`CLAUDE.local.md`** (git-ignored, auto-loaded alongside this file). It covers
recruiters / aggregators / job-board reposters, employers requiring a security
clearance the candidate can't obtain, and employers the candidate is personally
not interested in. Load it each run and drop anything it names. This file keeps
only the generic method (below); the named "who" stays local and personal.

### Aggregator tells (drop these even if the company isn't listed above)

Some reposters use throwaway/rotating company names, so also drop any posting
whose text gives it away as a middleman, and note it in the run summary as a
new blacklist candidate. Signals:

- "listed on behalf of a partner company, who manages all applications and next
  steps" (or similar "on behalf of a partner/client" phrasing).
- "Our partner is looking for…", "our client is seeking…", responses/next steps
  "managed off LinkedIn" by a third party rather than the employer.
- A recruiting/staffing agency as the posting "company" (e.g. names ending in
  Staffing, Recruiting, Consulting, Talent, Search), or a resume-review /
  job-board service.

When a listing from a blacklisted source (or matching a tell above) appears:
- Omit it from the output entirely (don't score it, don't link it).
- In the run summary, note how many rows were dropped and which source(s), so
  the exclusion is visible rather than silent.

To block another source later, add it to the blocklist in `CLAUDE.local.md`
with a one-line reason.

## Active — applications / interviews in progress (do not re-surface)

While an application or interview is active, omit **all** postings from that
company (any job_id, not just the one already seen) — you're already engaged and
don't want to re-apply. Keep the actual company list in `CLAUDE.local.md` (it
changes constantly and often names third-party recruiters). Note the exclusion in
the run summary. Remove a company from the local list once the process concludes.

## Employment-structure red flags (flag, don't necessarily drop)

Strong preference for **direct-employer roles** (clean W2 or direct contract).
Multi-layer staffing arrangements are a real negative and should be **flagged in
the board** so they can be weighed, even when the underlying role is a good fit:

- The "company" is an **IT-staffing vendor fronting a real employer** — diffuse
  accountability, and the process itself is often a signal (e.g. repeatedly moved
  interview times).
- **Layered employer-of-record / offshore payroll**: employer → staffing vendor →
  separate payroll shell in another timezone/country. Low-accountability, and slow
  or unreliable pay is common. (A real lived example is kept in `CLAUDE.local.md`.)
- **Onsite disguised as remote** / **"[Big Co] is not the employer for this role"**
  third-party-payroll postings — flag prominently; the visible brand isn't who pays
  or employs you.

These are caveats like the degree/clearance/timezone flags, not automatic
exclusions.

## Location / remote eligibility (Oregon)

Only surface roles Scott can actually hold from **Salem, Oregon**. This is a
**drop rule**, not a flag (changed 2026-09-24):

- **DROP** any posting that explicitly **excludes Oregon**, enumerates eligible
  states/regions where **Oregon isn't listed**, or restricts to a
  timezone/region that excludes Pacific/Oregon (e.g. "Eastern US only", "locals
  in DC/VA/MD", "must sit in EST/CST").
- **Keep** generic nationwide **"US Remote"** with no state list — Oregon is
  included by default there.
- When the metadata is ambiguous (no state list shown), keep it but verify
  against `get_job_details` before recommending; drop if the body reveals an
  Oregon-excluding state list.
- Note the count of Oregon-ineligible drops in the run summary.

## Daily board format

- **One combined board** per run: include **both full-time and contract** roles in
  a single artifact, grouped by employment type (Full-time section, then Contract).
- Run LinkedIn `search_jobs` with the explicit `job_type` filter for **both**
  `full_time` and `contract` (part-time has only ever returned gig / AI-trainer
  noise — skip it unless asked).
- **Contract section: staffing-agency contracts may be allowed** (TEKsystems,
  Robert Half, Randstad, Pyramid, etc.) when the operator opts in — remote contract
  SRE/DevOps is dominated by agencies, so excluding them empties the section. Still
  drop obvious off-target noise (AI/ML engineer, data engineer, mainframe,
  fullstack, generic "Software Engineer", ServiceNow/Guidewire, game-engine), and
  still flag onsite-disguised-as-remote and layered/third-party payroll.
- The staffing blacklist (in `CLAUDE.local.md`) **still applies to the full-time
  section** — the contract carve-out does not extend to full-time roles.
- Contract roles can be surfaced at title/rate/agency level when volume is high;
  note they're title-level and offer to deep-verify specific ones.

## Application tracking

- **The LinkedIn MCP server has no applied-jobs endpoint.** `get_saved_jobs` reads
  only the "Saved" tab. To get what you actually applied to, use browser automation
  on `https://www.linkedin.com/jobs-tracker/?stage=applied` (paginate; ~10/page)
  and read the job_ids from each card's `/jobs/view/<id>/` link.
- `applications.json` (git-ignored) is the running application log — it also serves
  as a personal work-search record.
- Every applied job_id must also be present in `seen-jobs.json` so daily boards
  never re-surface a role you already applied to.

## Notes

- The candidate profile and full review workflow live in the `linkedin-job-match`
  skill (`skills/linkedin-job-match/SKILL.md`).
- To block another company, add it to the blocklist in `CLAUDE.local.md` with a
  one-line reason. Distinguish middlemen (recruiters/aggregators) from real
  employers you simply aren't interested in — both live in `CLAUDE.local.md`.
