<p align="center">
  <img src="assets/crew-wordmark.png" alt="crew" width="440">
</p>

<p align="center"><strong>Vibe with your crew.</strong></p>

<p align="center">Fast iteration in, hardened code out.</p>

<br>

**crew** is the Claude Code plugin for *vibe-coding that actually ships*. Build
as fast and loose as you want — crew reviews every commit the moment it lands,
and before anything merges, a *second model* re-checks the work. The vibe stays;
the code holds.

## Meet the crew

| | | |
|---|---|---|
| **`/crew:flow`** | the loop | drives build → review for every commit, chunk, and turn |
| **`/crew:auditor`** | the correctness pass | logic errors, edge cases, broken contracts, intent drift |
| **`/crew:editor`** | the written-content pass | strips AI-isms, narration, and conversational residue |
| **`/crew:specialist`** | the outside read | brings in OpenAI Codex — a different model — to re-check the work |

`flow` runs the rest of the crew for you — or call them in yourself: `auditor`
and `editor` on any diff, `specialist` for a cross-model review or fresh eyes on
a stubborn bug.

## What happens when you build

Every commit gets an **audit** (correctness) and a separate **`/simplify`**
(quality) the moment it lands. Every batch of commits gets a deeper pass. And
every turn ends at the merge gate: Claude's own `/code-review`, then the
**specialist** calls in **OpenAI Codex** — a *different* model — to review what
Claude just did, and catch what it missed.

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
