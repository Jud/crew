<p align="center">
  <img src="assets/crew-wordmark.png" alt="crew" width="440">
</p>

<p align="center"><strong>The new way to build.</strong></p>

<p align="center">Fast iteration in, hardened code out.</p>

<br>

**crew** is a Claude Code plugin. You don't write the code — you say what to
build, and the crew builds it: reviewing every commit as it lands and bringing
in a second model to harden it, so what ships actually holds.

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

## How it works

Every commit the crew makes gets an **audit** (correctness) and a separate
**`/simplify`** (quality) the moment it lands. Each chunk and turn then closes
at the cross-model gate: Claude's own `/code-review`, then the **specialist**
brings in **OpenAI Codex** — a *different* model — to catch what Claude missed.

That second model is the point: crew never lets one model grade its own
homework. What ships has been audited, simplified, edited, and cross-examined.

## Install

```
/plugin marketplace add Jud/crew
/plugin install crew@crew
```

Then, on any commit-producing work:

```
/crew:flow
```

**One prerequisite.** The specialist runs the real OpenAI Codex CLI. The bridge
to it installs with crew, but the CLI is yours to provide — the `codex` command
on your `PATH`, plus OpenAI auth. Without it, the cross-model pass skips and
everything else still runs.

## License

MIT

---

<p align="center"><em>vibe with your crew</em></p>
