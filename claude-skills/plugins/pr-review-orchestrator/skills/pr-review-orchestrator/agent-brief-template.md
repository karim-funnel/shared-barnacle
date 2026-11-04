# Agent brief template

One brief per lens group. Fill every bracket; keep the fixed blocks verbatim — they are the campaign's
institutional knowledge. Agents are fresh (no inherited context), so the brief must be fully self-contained.
You cannot be asked questions mid-run: tell the agent which tools it has and that it must not ask the user
anything.

---

You are reviewing PR <N> of <repo> ("<title>", branch <branch> @ <sha>) through ONE lens group:
**<group>** (<lens 1> + <lens 2>). Create a worktree at the reviewed SHA
(`git -C <repo-path> worktree add --detach /tmp/<pr>-<group> <sha>`) and read there; never read the main
checkout, which may be on another branch or dirty. Scope:
`git diff <base>...<head> -- <this agent's triaged file subset>` — files outside the subset are reference
only: read them, don't review them.

**Ambient context**: repo path `<abs-path>`; base branch `<base>`; service map (which directories are which
service); per-service test command; how CI runs the suite; what CI already proved green at <sha>.

**MUST DO FIRST**: read <prior review reports, e.g. docs/dev/reviews/phase-NN.md> — a prior pass exists; do
NOT re-report its items (confirm the load-bearing ones in one line each).

**Known findings — do not re-report** (recap list: ids, anchors, one-liners; mark GT = ground truth
[empirically confirmed / mutation-proven] which you must not re-run).

**What to examine** (6–10 concrete, stack-specific checks. Not "check security" but "can a revoked user hold
a stream for 600s — trace the re-auth path". Name the files. Include at least one thing to verify against an
external source: the spec, the issue, the legacy code, the platform docs.)

<Safety block — adjust to stack>
Read-only: no edits to the real tree, no commits, no GitHub posting. Mutations/probes only in a throwaway
worktree (`git -C <repo> worktree add --detach /tmp/<name>-mut <sha>` … `git worktree remove` after).
Anything destructive (DBs, queues, buckets, emulator namespaces) only against a resource you create yourself
(e.g. <scratch-db-name>, or none if the repo uses only in-process mocks), dropped afterward — sibling agents
may be using the shared test resources concurrently. Never stop or reset shared emulators/containers; use
port overrides, since siblings hold the default ports. Paid/real-model API calls only if allowed here:
<allowed? cap N>.
Leave no servers or background processes; verify your cleanup. Scope any cleanup globs narrowly — an agent
once deleted sibling reviewers' payload files with an overbroad `/tmp/<pr>*` glob.
</Safety block>

**Deliverable** (final message): a JSON array ranked most-severe-first, each
`{"file": ..., "line": <1-indexed line at SHA>, "summary": ..., "failure_scenario": <concrete: who does what,
what breaks/leaks/costs>}`, max <cap, 6–8> — verified findings only, quality over quantity; if a check comes back
clean, say so in prose instead of padding. Plus prose: what's sound (be specific — green lights are findings
too), and <lens-specific extras: mutation ledger / run-log / verdict table>.

---

Brief-writing rules (for you, the orchestrator):

- The brief is the agent's ONLY quality lever if it's fresh. Spend the tokens: name files, name the exact
  commands, name the spec sources.
- Tell the agent what's already known, or it will rediscover it and you'll deduplicate forever.
- Ask for capped output (6–8) — uncapped agents produce 25 findings of which 4 matter.
- Ask for the prose half (what's sound) — without it, every round reads as pure bad news and the PR author
  stops reading.
- Grouping rules live in lenses.md §Grouping.
