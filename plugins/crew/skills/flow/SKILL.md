---
name: flow
description: |
  The default way to do any work that writes or changes code or durable
  artifacts — features, refactors, fixes, design docs, investigations: build it
  through this loop rather than editing freely. Skip it only for pure
  conversation, read-only exploration, or a one-line typo — and if you're
  unsure whether work qualifies, it does; invoke it. Pass an optional
  unit/chunk label via $ARGUMENTS.
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
    Editor                   # clean the unit's written content (comments + prose)
    Mid-turn close           # required once a chunk forms: /simplify → codex
    └─ loop back to the next unit (a close does NOT end the turn)

End-of-turn ceremony         # ALWAYS, when the turn ends:
                             #   /code-review --fix → codex
Report
```

The per-unit steps (Audit → /simplify → Inline codex → Editor) run
completely for each unit before you fetch the next; never batch units and review
them later.

## Audit + Simplify (every unit, right after commit)

Two MANDATORY passes per unit, run as DISTINCT steps in order — never fold
them into one review. **Audit** hunts correctness; **/simplify** hunts quality
(reuse, simplification, efficiency, altitude). They catch different things, so
running one is not a substitute for the other.

1. Confirm the tree holds ONE unit; stage explicitly (no `git add -A`)
   and commit with a per-unit label. The unit is `HEAD~1..HEAD` right after
   this commit; call the implementation commit's parent `<unit-base>` (it
   stays fixed as audit/simplify/inline-codex fix commits land on top), so
   review steps that run later scope to `<unit-base>..HEAD` — the whole unit
   so far.

2. **Audit (correctness).** Skip if trivial (≤20 lines, doc/comment/
   whitespace only) — note it. Otherwise invoke the **`/crew:auditor`
   skill** via the Skill tool — never a hand-rolled prompt — and let the
   forked subagent run to completion; never fake the run:

   ```
   Skill({skill: "crew:auditor", args: "<unit-range> <intent>"})
   ```

   `<unit-range>` is `HEAD~1..HEAD`; `<intent>` is a brief plain-text
   description of what this unit changes and why — free prose, not flow's
   chunk-label `$ARGUMENTS`. The auditor fetches its own diff and returns
   correctness findings only. Apply findings (or one-line skip reasons).
   Commit `Address <unit> audit findings` (no empty commits).

3. **Simplify (quality).** Skip if trivial (same threshold). Otherwise
   invoke the **literal `/simplify <unit-base>..HEAD` skill** via the Skill
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
fixes. Call **`/crew:specialist`** — the crew's cross-model gate; it brings in
Codex (never the raw `codex` CLI) with the locked defaults:

```
Skill({skill: "crew:specialist", args: "<unit-base>..HEAD"})
```

`<unit-base>..HEAD` is the whole unit so far (see step 1). Apply findings, commit
`Address <unit> inline-codex review` — labelled `review`, not `findings`, so it
can't be mistaken for a chunk-close `codex findings` commit (see Mid-turn close).

## Editor (every unit — mandatory, never folded into a close)

Right after the Audit and Simplify (and any Inline codex commit), on the same
unit. **Run it on every unit. The end-of-turn `/code-review` is not a
substitute:** that pass and `/simplify` clean *code* (reuse, simplification,
efficiency, altitude); the Editor cleans **written content** — code comments
**and** authored prose (docs, READMEs, docstrings, manifest/skill descriptions,
user-facing strings) — hunting the residue they never look for: AI-isms,
narration, conversational and requirements residue, filler. Invoke the
**`/crew:editor` skill** via the Skill tool:

   ```
   Skill({skill: "crew:editor", args: "<unit-base>..HEAD <intent>"})
   ```

`<unit-base>..HEAD` (see step 1) gives the editor the unit's full written
content, not just the latest fix; `<intent>` is as in the Audit step. The forked
editor fetches its own diff and returns cut/rewrite findings. Apply
deletions/rewrites (or one-line skips). Commit `Address <unit> editor pass` —
skip the commit, never the pass, when the writing is clean.

## Mid-turn close (required once a chunk forms)

**Recognizing when a chunk has formed (see Definitions) is your
judgement; closing it once it has is not.** The moment accumulated units
become a chunk and you intend to keep going, you MUST close it before
starting the next unit — don't let chunks pile up unreviewed. This is the
lighter close — `/simplify` → codex, without the `/code-review` pass — and
does **not** end the turn; loop back afterward. (The turn's *final* chunk is closed by the **End-of-turn
ceremony** instead.)

Chunk range = `<last chunk-close commit>..HEAD` — the most recent
`Address <chunk> codex findings` from a close. A per-unit `Address <unit>
inline-codex review` is NOT a close and does not anchor the range
(`origin/main` if no prior close on this branch).
1. Invoke `/simplify <chunk-range>` → apply (or one-line skips) → commit
   `Address <chunk> simplify findings`.
2. Run `/crew:specialist <chunk-range>` (the cross-model gate) → commit
   `Address <chunk> codex findings`. Loop back; don't end the turn.

## End-of-turn ceremony (ALWAYS — when the turn ends)

The broader, applied pass that closes the turn's **final** chunk:
`/code-review` also catches correctness bugs, and `--fix` applies findings
to the tree. Runs once at turn end over the chunk range =
`<last chunk-close commit>..HEAD` — the most recent `Address <chunk> codex
findings` from a close, not a per-unit `inline-codex review` commit
(`origin/main` if none).

1. Invoke `/code-review --fix <chunk-range>`, then **follow the loaded skill
   to completion: spawn its reviewer agents and let them produce the
   findings.** Commit `Address <chunk> code-review findings`.
2. Run the cross-model gate via `/crew:specialist`:
   ```
   Skill({skill: "crew:specialist", args: "<chunk-range>"})
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

## Gotchas

- **A hand-rolled `codex` hangs and floods.** Running `codex exec` directly
  (not via `/crew:specialist`, which goes through `skill-codex`) blocks forever
  on unclosed stdin and dumps its reasoning tokens into your context. Always go
  through the specialist.
- **Inline-codex commits must not look like chunk closes.** A per-unit inline
  codex commits as `Address <unit> inline-codex review` — never `… codex
  findings` — so the chunk/ceremony range sentinel won't anchor on it and skip
  the units before it.
- **The chunk range starts at the last *close*, not the last codex commit.**
  Anchor on the most recent chunk-close `Address <chunk> codex findings`
  (`origin/main` if none), never on a per-unit inline-codex review.

## Discipline

- **Run skills to completion.** When a step here invokes a skill
  (`/code-review`, `/simplify`, `/crew:specialist`, `/crew:auditor`,
  `/crew:editor`), that is mandatory, not a judgment call — invoke it and run
  its procedure verbatim with your own tool calls, then report what it
  produced.
- **One unit per commit.** Split mixed diffs before the Audit.
- **Audit and /simplify are both mandatory, run separately.** Never merge
  them into one pass — Audit is correctness, /simplify is quality; each is a
  distinct run with its own findings commit.
- **The Editor is mandatory on every unit, and `/code-review` is not a
  substitute.** The Editor cleans *written content* (AI-isms, narration,
  conversational and requirements residue); `/code-review` and `/simplify` clean
  *code* (reuse, simplification, efficiency, altitude) — different defect
  classes. Run the Editor over every unit's written content; never fold it into
  a close.
- **Codex only via `/crew:specialist`** (which goes through `skill-codex`),
  never the raw CLI — the specialist owns the gpt-5.5 / xhigh / read-only
  defaults and keeps codex's reasoning tokens out of your context (see Gotchas
  for why a hand-rolled call hangs and floods). It runs AFTER the review pass,
  to catch what that pass broke.
- **Honest ROI.** Skip findings with one-line reasons; don't argue.
- **No safety bypass.** No `--no-verify`, no force ops; fix hook failures
  at the source.
- **Speak rule.** With a speak hook, narrate only state transitions:
  entering audit, /simplify, codex, the Editor, mid-turn /simplify, the
  end-of-turn ceremony, loop done.

## Rationalizations to reject

These excuses don't hold:

- *"One review pass covered correctness and quality."* — No. Audit and
  /simplify catch different defect classes; run both, separately.
- *"The ceremony's `/code-review` already has a cleanup pass, so the per-unit
  Editor is covered."* — No. That pass cleans *code* (reuse, simplification,
  efficiency); the Editor cleans *written content* (AI-isms, narration,
  conversational residue) and runs only when you run it. Editor on every unit.
- *"It's one small chunk, I'll skip the ceremony."* — No. The turn ends on a
  close; run the end-of-turn ceremony once (see Single-chunk caveat).
- *"I'll batch these units and review them later."* — No. Per-unit review runs
  to completion before you fetch the next.
- *"This won't really produce a commit, so the loop doesn't apply."* — No. The
  trigger is doing real code/artifact work, not predicting a commit; only the
  description's skip-list (pure conversation, read-only, one-line typo) is
  exempt. When unsure, invoke.
