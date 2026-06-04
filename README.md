# work-loop

**Fast iteration in, hardened code out.**

`work-loop` is a Claude Code plugin that wraps a single, opinionated skill: a
**post-implementation hardening loop**. It doesn't tell the agent *how* to write
code or impose any upfront planning — it relentlessly *verifies and cleans*
what was written, one commit at a time, and gates the work behind a
cross-model review before it can merge.

The bet: skip the heavyweight front-loading, iterate quickly, and let a
disciplined back-end loop catch the slop — so you get a quality artifact at the
end instead of a fast mess.

## What it does

For every **unit** (one logical commit), right after you commit:

1. **Audit** — a correctness-only pass (spawned subagent): logic errors, edge
   cases, broken contracts, intent mismatch.
2. **/simplify** — a separate quality-only pass: reuse, simplification,
   efficiency, altitude. (Run as a *distinct* step — correctness and quality
   catch different things.)
3. **Inline Codex** *(only on technically hard units — math, crypto,
   concurrency, public contracts)* — a second, different model (OpenAI Codex)
   audits the diff.
4. **Comment cleanup** — strips narration and AI-residue comments.

Once a **chunk** (several units) forms, it's closed with a `/simplify` → Codex
pass. And every turn ends with the **ceremony**: `/code-review --fix` → Codex
over the whole chunk. That ceremony is the merge gate — Codex runs *after* the
Claude review specifically to catch what the review itself broke. Cross-model
verification is the thing this loop has that pure single-model review does not.

## Install

```
/plugin marketplace add Jud/work-loop
/plugin install work-loop@work-loop
```

Then invoke it on commit-producing work:

```
/work-loop:work-loop [optional unit/chunk label]
```

Installing `work-loop` **auto-installs its dependency, `skill-codex`** (the
OpenAI Codex bridge), from this same marketplace — you do **not** need to add
any other marketplace.

## Prerequisites

`work-loop` orchestrates a few things it expects to be present:

| Dependency | How it's provided | You still need |
| --- | --- | --- |
| **`skill-codex`** plugin | Auto-installed with work-loop (declared dependency) | The **Codex CLI** on your `PATH` **and** OpenAI authentication. The plugin is just the bridge — without the CLI + auth, the Codex passes cannot run. |
| **`/simplify`**, **`/code-review`** | Built into recent Claude Code | A reasonably current Claude Code version |

> **Heads up:** the single most common failure mode is the Codex passes
> silently no-op'ing because the `codex` CLI isn't installed or isn't
> authenticated. If you see the loop skipping every Codex step, check that
> first.

## Local development

```
claude --plugin-dir ./plugins/work-loop      # load without installing
/reload-plugins                              # after editing the manifests
```

Edits to `skills/work-loop/SKILL.md` take effect immediately in the session;
changes to the manifests (`plugin.json` / `marketplace.json`) need
`/reload-plugins`.

> **Note:** a `--plugin-dir` load has no marketplace context, so the
> `skill-codex` dependency is **not** auto-resolved this way. For the Codex
> passes to run during local dev, install `skill-codex` separately (or add this
> marketplace and install work-loop normally).

## Updating

- **work-loop itself** is versioned in `plugins/work-loop/.claude-plugin/plugin.json`.
  Bump that `version` on each release — pushing commits without bumping it has
  no effect on installed users.
- **The pinned Codex** is locked to an exact commit via the `sha` in
  `.claude-plugin/marketplace.json` (the `skill-codex` entry). This gives
  reproducible behavior, at the cost of no automatic upstream updates — moving
  to a newer Codex is a deliberate "bump the sha" chore.

## License

MIT — see [LICENSE](LICENSE).
