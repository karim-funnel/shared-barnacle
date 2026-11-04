---
name: pr-review-orchestrator
description: Fan out a team of lens-specific review subagents over one PR, post each round as labeled GitHub reviews, then verify everything — including the fixes. Use when the user asks to "review PR N", "run the review drill on this PR", or names a PR number/branch to review.
---

# PR Review Orchestrator

Fan out a team of lens-specific review subagents over one PR, post each round as labeled reviews on GitHub,
then verify everything — including the fixes. Built from a 4-PR, ~45-agent campaign; every rule below was
learned the hard way. Works for any stack (web app, API, mobile backend): swap the lens specifics in
`lenses.md`, keep the process.

## When to use

"review PR N", "run the review drill on this PR", "do the subagent thing". User names a PR number or branch.

## Step 0 — Target and scope (always, before any agent)

```
git fetch origin
gh pr view <N> --json headRefName,headRefOid,title,state
git diff <base>...<head> --name-status        # the TRUE delta
git log --oneline <base>...<head> | head
```

- **Stacked-PR trap**: if the branch contains another open PR's commits, review the *delta beyond that PR's
  tip*, not vs main — and say so in every review body ("reviewers should read the X..Y delta").
- Find prior in-repo review reports (e.g. `docs/dev/reviews/`) — agents must read them first and confirm,
  not re-report.
- Build the **known-findings recap** (this PR's + carried-over history). Every agent brief gets it.
- Record the PR head SHA. Branches move mid-review; findings must anchor to the reviewed SHA, and you re-diff
  before posting if the head changed.
- **Lens triage (from the same `--name-status` output — no agent needed):** bucket the changed files by
  class — core code, UI, deploy/CI/config, tests, docs. A lens with no files in its class is **dropped**,
  not downweighted. The surviving lenses are grouped per lenses.md, and each group's agent is scoped to its
  own file subset (`git diff <base>...<head> -- <files>`), not the whole delta. This is the token lever:
  every fresh agent otherwise re-reads the full diff and the same hot files.

## Step 1 — Round plan

Roster, grouping, and lens briefs live in [lenses.md](lenses.md). Default shape for a large PR is **four
agents, two waves** (fewer for small PRs — triage decides; a 10-file bugfix may need one):

| Group | One agent covering | Shares |
|---|---|---|
| A | backend/business-logic + architecture/plan (+ forward-compat) | core-code + plan-doc reading |
| B | security + devops/ops (+ frontend/a11y only if triage shows UI files) | endpoints, config, deploy path |
| C | tests + **empirical execution** | the suite + scratch-DB setup |
| D | **Adversarial verification** + **ship-gate** | the verified-findings record A–C produced |
| (E) | Fix-batch verifier + ship-gate again | After the user says "fixed" |

A–C are independent lenses: fire them **concurrently**. D fires only when A–C are fully posted, so its
brief includes their findings (plus a slot for whatever they surfaced).

Rules that made the campaign work:
- Fire A–C concurrently (independent lenses); fire D only when A–C are fully posted, so D's brief includes
  their findings.
- Every agent gets: the exact diff command, the known-findings recap, "read prior reports first, don't
  re-report", and the deliverable format. Template: [agent-brief-template.md](agent-brief-template.md).
- **Use fresh subagents, not forks.** A fork inherits your whole conversation as its starting context — in a
  long review campaign that means every agent re-reads everything (measured: ~60 fork agents in one session
  was a massive token multiplier), and it locks them to the parent's model. Fresh agents cost only the brief
  you write. That makes the brief the entire quality lever: it must be fully self-contained (repo path, the
  exact diff command, the known-findings recap, the deliverable contract, safety rules, and any ambient
  context the fork would have gotten free — see the template). Reserve forks for short sessions where the
  inherited context is small and the shared state is hard to write down.

## Step 2 — Launching agents

Fill the template per lens group. Non-negotiables in every brief:

1. **Scope discipline**: one diff command, scoped to the agent's triaged file subset; "files outside that
   subset are reference only — read them, don't review them".
2. **Deliverable contract**: JSON array `[{file, line, summary, failure_scenario}]` ranked most-severe-first,
   capped (6–8), plus prose (what's sound — reviewers need the green lights too).
3. **Evidence bar**: every finding verified at the PR head with file:line; "quality over quantity; if a check
   comes back clean, say so in prose instead of padding".
4. **Safety**: read-only; mutations only in a throwaway worktree (`git worktree add --detach /tmp/... `,
   removed after); destructive probes only on a **scratch DB/resource they create** (sibling agents share the
   default test DB — collisions produce false failures; this bit us twice); no commits, no GitHub posting
   (the orchestrator posts).
5. **Ground truth marking**: empirically-confirmed and mutation-proven findings are marked GT so the
   adversarial pass doesn't waste effort re-running them.

## Step 3 — On every agent's return (before saying anything to the user)

1. **Verify anchors yourself** at the PR head SHA: `git show <sha>:<file> | awk 'NR>=X && NR<=Y'`. One command,
   batch all the anchors. Agents are good but occasionally anchor one line off; posting a wrong anchor is
   embarrassing and confusing on GitHub.
2. **Hunk check for inline commentability**: an inline comment must land on a line inside the PR's diff hunks
   (`git diff <base>...<head> -- <file> | grep '^@@'`, compare hunk ranges). Findings on unchanged lines go in
   the review **body** with explicit `file:line` references — say why. Files added wholesale by the PR are
   always fully commentable.
3. Spot-check any surprising factual claim (a file size, a "grep found nothing") — one cheap command.
4. Digest to the user: findings one-liner each, what's sound, what's still running.

## Step 4 — Posting

Lens-labeled review per agent (`**Lens: security**`), inline comments where anchorable, body for the rest.
Verdict tables, run-logs and fix reports go as plain PR comments.

```bash
# ALWAYS validate the payload first — a bad quote or a "file" key instead of "path" is a 400/422:
jq . /tmp/review.json > /dev/null && gh api repos/<owner>/<repo>/pulls/<N>/reviews --input /tmp/review.json
# GitHub does not echo comment counts — verify after:
gh api repos/<owner>/<repo>/pulls/<N>/comments --jq '[.[] | select(.pull_request_review_id == <id>) | {path, line}]'
```

Payload shape: `{"body": "...", "event": "COMMENT", "comments": [{"path", "line", "body"}]}`.
Findings comment body: **bold claim**, **failure scenario** (concrete: who does what, what breaks), **Fix**.
Comment bodies end with the attribution line your harness requires.

**Permission model**: posting is outward-facing. Ask once ("post these?") unless the user gave a standing
instruction for the campaign ("do it 5 times, review and post"). Never let an agent post.

## Step 5 — Fix-batch verification and the gate

When the user says "fixed":
1. Diff the fix commits. Map each to an expected fix (commit messages usually name them).
2. Launch a fix-batch verifier: verdict per fix — FIXED / PARTIAL / NOT FIXED / **FIXED-NEW-PROBLEM** —
   *verified as behavior* (run it, probe it), never as commit message. Check the fix code as code too.
3. Launch a ship-gate: interactions between fixes, collateral on previously-good things, one-story doc
   coherence, and the verdict: **MERGE / MERGE-WITH-N / DO-NOT-MERGE**, with findings sorted into
   blocks-merge / before-cutover(-deploy) / ride-to-next-phase. "This PR's merge itself performs the unsafe
   act" (e.g. a deploy step that runs on merge) promotes deploy-gate findings to merge-blocking.

## Hard-won operational lessons

- **Anchor to a SHA, not a branch.** Branches move mid-review; re-diff at the PR head before posting and say
  which SHA findings were verified at.
- **Sibling test-DB collisions**: concurrent agents running the suite against the same DB produce convincing
  false failures. Every agent that runs tests gets its own scratch DB, created and dropped.
- **Mutations beat coverage opinions**: "does any test catch X?" is answered by deleting X and running the
  suite, not by reading tests. Mutation survivors are the best test findings.
- **Empirical execution finds what reading cannot** (three deploy blockers across the campaign, all
  runtime-only). The lens is worth an agent even on small PRs.
- **Evidence precedence: execution > mutation > reading.** When two passes disagree (an execution agent
  "refutes" a mutation-proven finding), check whether it actually tested the same claim — then reconcile on
  the record rather than silently keeping either verdict. One finding survived as "confirmed" this way: the
  cases were covered, the assertions were not.
- **The adversarial pass pays for itself before a fixing session**: it regularly narrows, strengthens, or
  splits findings (and once caught a coverage gap that made a doc fix *institutionalized*). It also produces
  fix-packaging guidance (which findings must land as ONE change).
- **Ground-truth tiering** (execution-confirmed > mutation-proven > read-verified) tells the adversarial pass
  where to spend skepticism.
- **Verify agents' cleanup claims** (`git status`, `pgrep` for stray servers) — one agent left a server; the
  panel showed it "on" for an hour after.
- **Bodies carry the record**: sound-checks, run-logs, verdict tables. Six months later the PR thread is the
  campaign's memory.
- **A stale subagent entry** ("waiting on background work") in the UI is usually a finished agent whose last
  child process was still exiting — check `ListAgents`/`pgrep` before worrying.
