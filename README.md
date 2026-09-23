# Job Search Kit

A repeatable, AI-assisted workflow for screening remote SRE / DevOps / Platform /
Cloud job postings on LinkedIn against your résumé, and producing a match-scored
"board" you can act on. Built and run with [Claude Code](https://claude.com/claude-code)
plus a LinkedIn MCP server.

It grew out of one person's real job hunt. The **method** is here to share; the
**personal data** (application funnel, dedup ledger, private notes) is git-ignored
and stays on the operator's machine. Fork it, improve it, and — if you're feeling
generous — send improvements back.

## What it does

- Searches LinkedIn Jobs (via MCP) for a couple of keyword sets, remote, posted in
  the last 24h, filtered to **full-time** and **contract** employment types.
- **Deduplicates** against a local ledger so the same reposts don't reappear
  day after day.
- **Scores each role** against your résumé — strong / partial / weak — reading the
  *body* of the posting, not just the title.
- **Flags** the things that quietly disqualify a "remote" role: hard degree
  requirements, security-clearance / US-citizen gates, on-site-disguised-as-remote,
  layered/offshore payroll, and staffing middlemen.
- Publishes a single themed **HTML board** grouping full-time and contract roles.
- Tracks what you've **applied to** (LinkedIn's applied list isn't exposed by the
  MCP — it's read from the Job Tracker page via browser automation) so applied
  roles never get re-surfaced.

## Prerequisites

1. **Claude Code** — https://claude.com/claude-code
2. **uv** (Python package runner used to launch the MCP server) —
   install per https://docs.astral.sh/uv/getting-started/installation/
3. **The LinkedIn MCP server.** Add it to Claude Code with:

   ```
   claude mcp add --transport stdio mcp-server-linkedin --env UV_HTTP_TIMEOUT=300 -- uvx mcp-server-linkedin@latest
   ```

   On first use it will prompt you to sign in to LinkedIn in a browser window —
   Claude never handles your credentials.

4. *(Optional)* The **Claude in Chrome** extension, used only to read your
   LinkedIn "Applied" job-tracker list (the MCP can't see it).

## Layout

```
CLAUDE.md                          # project rules: blacklist, board format, flags
CLAUDE.local.md                    # YOUR private overrides (git-ignored)
skills/linkedin-job-match/SKILL.md # the full workflow the agent follows
templates/board-template.html      # the HTML board design
examples/seen-jobs.example.json    # the dedup-ledger format (empty skeleton)

# Created and maintained locally, git-ignored:
seen-jobs.json                     # your live dedup ledger
applications.json                  # your application log / work-search record
```

## Using it

From this directory, in Claude Code, just ask for the daily board — e.g.
*"run today's job board."* The agent reads `CLAUDE.md` + `CLAUDE.local.md`, runs
the searches, dedupes against `seen-jobs.json`, scores the survivors, and publishes
the board. To adapt it to you: edit `CLAUDE.md` (blacklist, keywords, flags), keep
your personal exclusions/active interviews in `CLAUDE.local.md`, and point the
skill at your own résumé.

## Privacy

`applications.json`, `seen-jobs.json`, and `CLAUDE.local.md` are git-ignored on
purpose — they contain your application history and private notes. Keep third-party
personal data (recruiter names, contacts) in `CLAUDE.local.md`, never in the tracked
files.

## License

[MIT](LICENSE) © 2026 Scott M. Likens.
