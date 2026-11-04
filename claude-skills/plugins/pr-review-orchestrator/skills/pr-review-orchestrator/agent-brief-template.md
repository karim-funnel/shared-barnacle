# Agent brief template

One brief per lens group. Fill every bracket; keep the fixed blocks verbatim — they are the campaign's
institutional knowledge. Forks (inheriting your context) need less recap; fresh agents need it all.

---

You are reviewing PR <N> of <repo> ("<title>", branch <branch> @ <sha> — the working tree IS this branch,
<clean?>; read directly) through ONE lens group: **<group>** (<lens 1> + <lens 2>). Scope:
`git diff <base>...<head> -- <this agent's triaged file subset>` — files outside the subset are reference
only: read them, don't review them.
<For forks: You inherit this conversation's context — the known findings and repo layout are in it.>

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
Anything destructive (DBs, queues, buckets) only against a resource you create yourself (e.g.
<scratch-db-name>), dropped afterward — sibling agents may be using the shared test resources concurrently.
Leave no servers or background processes; verify your cleanup. Scope any cleanup globs narrowly — an agent
once deleted sibling reviewers' payload files with an overbroad `/tmp/<pr>*` glob.
</>

**Deliverable** (final message): a JSON array ranked most-severe-first, each
`{"file": ..., "line": <1-indexed at <sha>>, "summary": ..., "failure_scenario": <concrete: who does what,
what breaks/leaks/costs>}`, max <6-8> — verified findings only, quality over quantity; if a check comes back
clean, say so in prose instead of padding. Plus prose: what's sound (be specific — green lights are findings
too), and <lens-specific extras: mutation ledger / run-log / verdict table>.

---

Brief-writing rules (for you, the orchestrator):

- The brief is the agent's ONLY quality lever if it's fresh. Spend the tokens: name files, name the exact
  commands, name the spec sources.
- Tell the agent what's already known, or it will rediscover it and you'll deduplicate forever.
- Ask for capped output — uncapped agents produce 25 findings of which 4 matter.
- Ask for the prose half (what's sound) — without it, every round reads as pure bad news and the PR author
  stops reading.
- Group only lenses that share reading material, and cap at two or three per agent (see lenses.md).
  "General correctness + security + tests" in one brief produces shallow all three.
