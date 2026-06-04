---
name: work-loop
description: |
  An iterative development loop: build one agent-sized unit at a time,
  reviewing each as it lands (audit + /simplify, both mandatory, run
  separately); accumulate units into chunks that get a second, different
  review (/simplify → codex); close the turn with a /code-review --fix → codex
  gate. Optional inline codex on hard units. Pass an optional unit/chunk label
  via $ARGUMENTS.
allowed-tools: Bash Read Edit Write Agent Skill TaskCreate TaskUpdate TaskList
---

## Definitions

- **Unit** — one logical commit. What counts as a unit is the agent's
  discretion.
- **Chunk** — two or more units, also the agent's discretion: generally
  the point where the accumulated units of work are *just* large enough to
  benefit from deduplication and/or a fresh pair of eyes. How a chunk is
  closed is defined under **Mid-turn close** and **End-of-turn ceremony**.

## Input

`$ARGUMENTS` — an optional short label for commit messages and the report;
may name a single unit or a chunk theme.

## Loop shape

A turn runs **many units and may close several chunks**. Two rules, and
the direction between them is the point:

1. **The turn ends on a chunk close.** Its final act is the end-of-turn
   ceremony (`/code-review --fix` → codex) over everything since the last
   close. Always, no exceptions.
2. **A chunk close is not a stop signal.** The instant one closes, pick up
   the next unit and keep going — use the 1M context window to drive the
   whole agenda in one turn. The turn ends only when the agenda is
   exhausted, a real stopping point is reached, or context warrants it —
   never merely because a chunk closed. *Turn-end ⟹ chunk close; never
   the reverse.*

```
while agenda not exhausted:
    implement → commit       # one logical change per commit
    Audit                    # correctness pass — diff still fresh
    /simplify                # quality pass — the real skill
    │                        # ↑ run BOTH, as distinct passes; never combine
    Inline codex             # only if the unit is hard (discretion)
    Comment cleanup
    Mid-turn close           # required once a chunk forms: /simplify → codex
    └─ loop back to the next unit (a close does NOT end the turn)

End-of-turn ceremony         # ALWAYS, when the turn ends:
                             #   /code-review --fix → codex
Report
```

The per-unit steps (Audit → /simplify → Inline codex → Comment cleanup) run
completely for each unit before you fetch the next; never batch units and review
them later.

## Audit + Simplify (every unit, right after commit)

Two MANDATORY passes per unit, run as DISTINCT steps in order — never fold
them into one review. **Audit** hunts correctness; **/simplify** hunts quality
(reuse, simplification, efficiency, altitude). They catch different things, so
running one is not a substitute for the other.

1. Confirm the tree holds ONE unit; stage explicitly (no `git add -A`)
   and commit with a per-unit label. Unit range = `HEAD~1..HEAD`.

2. **Audit (correctness).** Skip if trivial (≤20 lines, doc/comment/
   whitespace only) — note it. Otherwise launch ONE `general-purpose`
   Agent (Opus) with this prompt verbatim — actually spawn the agent,
   never fake the run:

   > You are a code auditor. Review the supplied diff for **correctness
   > only**. Do NOT review style, reuse, or efficiency. Check: logic
   > errors, off-by-ones, missing edge cases; type drift, broken
   > contracts, invariants violated; dead code or unreachable branches;
   > bugs the diff *almost* fixes; mismatch between stated intent
   > (`$ARGUMENTS`) and what the code does. Output a short bulleted list.
   > Each finding: file:line, what's wrong, what to do. No narrative. If
   > correct as-is, say so in one line.

   Apply findings (or one-line skip reasons). Commit `Address <unit>
   audit findings` (no empty commits).

3. **Simplify (quality).** Skip if trivial (same threshold). Otherwise
   invoke the **literal `/simplify <unit-range>` skill** via the Skill
   tool — NOT a hand-rolled prompt — and **follow the loaded skill to
   completion: spawn its agents and let them produce the findings.** Same
   bar as the end-of-turn `/code-review`: actually run it; never summarize,
   hand-wave, or fabricate a result. Apply findings (or one-line skip
   reasons). Commit `Address <unit> simplify findings`.

## Inline codex (only if the unit is technically hard — discretion)

Run NOW if the unit touches math, cryptography, numerics, concurrency, or
a public contract (soundness / relation logic, wire formats, public APIs,
error envelopes). Routine units skip it; the ceremony codex covers them.

Scope to the whole unit so far — implementation **plus** the audit/simplify
fixes — via the **`skill-codex:codex` skill, never the raw `codex` CLI:**

```
Skill({skill: "skill-codex:codex",
       args: "Review the committed diff <unit-base>..HEAD for correctness, security, and contract bugs. gpt-5.5, xhigh, read-only — don't prompt for these. Findings only; no edits."})
```

`<unit-base>` = the parent of your implementation commit (`HEAD~1`, walking
back past any audit/simplify fix commits that followed). Apply findings, commit
`Address <unit> codex findings`.

## Comment cleanup (every unit, lightweight)

Right after the Audit and Simplify (and any Inline codex commit), on the same unit.
Launch ONE `general-purpose` Agent (Opus) with this prompt verbatim:

   > You are a comment auditor. Review the supplied diff for **comments
   > only**. Delete comments that:
   > - Explain WHAT the code does (well-named identifiers already do).
   > - Narrate the change (`// added for X`, `// removed Y`).
   > - Reference the task or caller (`// used by X`, `// fixes #123`) —
   >   these belong in the commit message, not the code.
   > - Are AI-style "this method does X" docs without non-obvious WHY.
   > - Leak conversational residue — comments addressed to a reader or
   >   referencing the chat/session (`// as you requested`, `// here's the
   >   fix`, `// per our discussion`, `// let me know if…`, `// I changed
   >   this to…`, `// note for review`).
   >
   > Keep comments that explain non-obvious WHY: hidden constraints and
   > subtle invariants; workarounds for specific bugs (cite the bug);
   > surprising behavior; pinned algorithmic source (paper §refs, vendor
   > docs).
   >
   > Output a short bulleted list: file:line, what to delete, why. No
   > narrative. If clean, say so in one line.

Apply deletions (or one-line skips). Commit `Address <unit> comment
cleanup`; skip the commit if comments are clean.

## Mid-turn close (required once a chunk forms)

**Recognizing when a chunk has formed (see Definitions) is your
judgement; closing it once it has is not.** The moment accumulated units
become a chunk and you intend to keep going, you MUST close it before
starting the next unit — don't let chunks pile up unreviewed. This is the
lighter, quality-only pass and does **not** end the turn; loop back
afterward. (The turn's *final* chunk is closed by the **End-of-turn
ceremony** instead.)

Chunk range = `<last "Address … codex findings" commit>..HEAD`
(`origin/main` if none on this branch).
1. Invoke `/simplify <chunk-range>` → apply (or one-line skips) → commit
   `Address <chunk> simplify findings`.
2. Codex over the range via the `skill-codex:codex` call shown in the
   end-of-turn ceremony → commit `Address <chunk> codex findings`. Loop
   back; don't end the turn.

## End-of-turn ceremony (ALWAYS — when the turn ends)

The broader, applied pass that closes the turn's **final** chunk:
`/code-review` also catches correctness bugs, and `--fix` applies findings
to the tree. Runs once at turn end over the chunk range =
`<last "Address … codex findings" commit>..HEAD` (`origin/main` if none).

1. Invoke `/code-review --fix <chunk-range>`, then **follow the loaded skill
   to completion: spawn its reviewer agents and let them produce the
   findings.** Commit `Address <chunk> code-review findings`.
2. Codex over the range, never the raw CLI:
   ```
   Skill({skill: "skill-codex:codex",
          args: "Review the committed diff <chunk-range> for correctness, security, and contract bugs. gpt-5.5, xhigh, read-only — don't prompt. Findings only; no edits."})
   ```
   Apply findings, commit `Address <chunk> codex findings`, then go to
   Report.

Codex is mandatory on every close — skip it only if the Inline codex
already ran on the chunk's sole unit and the review pass was pure cleanup.

## Single-chunk caveat

When the whole turn is a single chunk — you never triggered a mid-turn
close — running a mid-turn close *and* the end-of-turn ceremony over the
same units is duplicative. **Combine them into one close: just run the
end-of-turn ceremony once over all the turn's units.** A mid-turn
`/simplify` close only earns its keep when a turn breaks into *multiple*
chunks. (Likewise, if a mid-turn close already covered everything and no
units followed, the ceremony may land as a genuine no-op — fine; running
it satisfies the rule.)

## Report

Commits produced (title + short hash); deferred findings + reasons;
whether the chunk closed (if not, and the branch has no prior
code-review+codex, flag it — cannot merge); suggested next step.

## Discipline

- **Run skills to completion.** When a step here invokes a skill
  (`/code-review`, `/simplify`, `skill-codex:codex`), that is mandatory, not
  a judgment call — invoke it and run its procedure verbatim with your own
  tool calls, then report what it produced.
- **One unit per commit.** Split mixed diffs before the Audit.
- **Audit and /simplify are both mandatory, run separately.** Never merge
  them into one pass — Audit is correctness, /simplify is quality; each is a
  distinct run with its own findings commit.
- **Codex only via `skill-codex:codex`**, never the raw CLI. It runs AFTER
  the review pass (to catch what that pass broke), scoped to the diff
  range, with the gpt-5.5 / xhigh / read-only default.
- **Honest ROI.** Skip findings with one-line reasons; don't argue.
- **No safety bypass.** No `--no-verify`, no force ops; fix hook failures
  at the source.
- **Speak rule.** With a speak hook, narrate only state transitions:
  entering audit, /simplify, codex, comment cleanup, mid-turn /simplify, the
  end-of-turn ceremony, loop done.
