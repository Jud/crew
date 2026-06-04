# work-loop

**Fast iteration in, hardened code out.**

`work-loop` is a Claude Code plugin that wraps a single, opinionated skill: an
**iterative development loop**. You build one unit at a time — the agent decides
what a unit is — and review each one the moment it's committed (correctness
audit + quality pass), then review the accumulated chunk a different way. It
imposes no upfront planning; the rigor is applied continuously as you build, and
a cross-model review gates the work before it can merge.

The bet: skip the heavyweight front-loading, iterate fast, and harden each unit
as it lands — so you get a quality artifact instead of a fast mess.

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

Edits to `skills/work-loop/SKILL.md` take effect immediately in-session;
manifest changes (`plugin.json` / `marketplace.json`) need `/reload-plugins`
— run it too if a skill edit doesn't seem to apply.

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
  reproducible behavior, at the cost of no automatic upstream updates. To move
  to a newer Codex, bump the `sha`. Note that update detection keys on the
  `version` string, not the SHA: if the new Codex commit keeps the same
  `skill-codex` version, existing installs may need a `/plugin` reinstall (or
  `claude plugin prune`) to pick it up.

## License

MIT — see [LICENSE](LICENSE).
