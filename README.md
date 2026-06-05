<p align="center">
  <img src="assets/crew-wordmark.png" alt="crew" width="440">
</p>

<p align="center"><strong>Fast iteration in, hardened code out.</strong></p>

<br>

**crew** is a Claude Code plugin for people who'd rather build than plan. You
write code one commit at a time — no upfront design doc, no checklist to forget —
and crew reviews every commit the moment it lands. Then, before anything merges,
a *second model* re-checks the work.

## Meet the crew

| | | |
|---|---|---|
| **`/crew:flow`** | the loop | drives build → review for every commit, chunk, and turn |
| **`/crew:auditor`** | the correctness pass | logic errors, edge cases, broken contracts, intent drift |
| **`/crew:editor`** | the written-content pass | strips AI-isms, narration, and conversational residue |

`flow` runs the other two for you — or call them yourself on any diff.

## What happens when you build

Every commit gets an **audit** (correctness) and a separate **`/simplify`**
(quality) the moment it lands. Every batch of commits gets a deeper pass. And
every turn ends at the merge gate: Claude's own `/code-review`, then **OpenAI
Codex** — a *different* model — reviewing what Claude just did, to catch what it
missed.

That last step is the point. Most review tools ask one model to grade its own
homework. crew brings a second one, with different blind spots. You move fast
and loose; what comes out has been audited, simplified, edited, and
cross-examined.

## Install

```
/plugin marketplace add Jud/crew
/plugin install crew@crew
```

Then, on any commit-producing work:

```
/crew:flow
```

**One prerequisite.** crew's cross-model gate runs the real OpenAI Codex CLI.
The bridge to it (the `skill-codex` plugin) installs with crew, but the CLI
itself is yours to provide: the `codex` command on your `PATH`, plus OpenAI
auth. Without it, the Codex passes skip — everything else still runs.

## License

MIT
