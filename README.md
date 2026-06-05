# crew

**Fast iteration in, hardened code out.**

`crew` is a Claude Code plugin: an iterative development loop. You build one
unit at a time — the agent decides what counts as a unit — and review each the
moment it's committed. There's no upfront
planning; the rigor lands continuously, as you build.

The crew is three skills:

- **`/crew:flow`** — the loop. Drives build → review for every unit, and closes
  each chunk and the turn with deeper gates.
- **`/crew:auditor`** — a correctness-only review of a diff range. Flow spawns
  it per unit; also runnable on its own.
- **`/crew:editor`** — an editorial pass over a diff's written content (comments
  *and* prose). Flow calls it per unit; also runnable on its own.

## The loop

For every **unit** (one logical commit), right after you commit:

1. **Audit** (`/crew:auditor`) — correctness only: logic errors, edge cases,
   broken contracts, intent mismatch.
2. **/simplify** — a separate quality pass: reuse, simplification, efficiency,
   altitude. (Distinct from the audit — correctness and quality catch different
   things.)
3. **Inline Codex** *(only on technically hard units — math, crypto,
   concurrency, public contracts)* — a second, different model (OpenAI Codex)
   audits the diff.
4. **Editor** (`/crew:editor`) — cleans the unit's written content: strips
   narration, AI-isms, and conversational residue.

Once a **chunk** (several units) forms, it's closed with a `/simplify` → Codex
pass. And every turn ends with the **ceremony**: `/code-review --fix` → Codex
over the whole chunk. That ceremony is the merge gate — Codex runs *after* the
Claude review specifically to catch what the review itself broke.

## Install

```
/plugin marketplace add Jud/crew
/plugin install crew@crew
```

Then invoke the loop on commit-producing work:

```
/crew:flow [optional unit/chunk label]
```

`/crew:auditor` and `/crew:editor` are available the same way, for an ad-hoc
review of a diff range.

## Prerequisites

| Dependency | How it's provided | You still need |
| --- | --- | --- |
| **`skill-codex`** plugin | Bundled with crew | The **Codex CLI** on your `PATH` **and** OpenAI authentication. The plugin is just the bridge — without the CLI + auth, the Codex passes cannot run. |
| **`/simplify`**, **`/code-review`** | Built into recent Claude Code | A reasonably current Claude Code version |

> **Heads up:** the single most common failure mode is the Codex passes
> silently no-op'ing because the `codex` CLI isn't installed or isn't
> authenticated. If you see the loop skipping every Codex step, check that
> first.

## Local development

Load the plugin from a local checkout — both forms give you the real `/crew:*`
namespace:

```
# persistent — add this repo as a local marketplace, then install
/plugin marketplace add /path/to/crew
/plugin install crew@crew

# or session-scoped — load the plugin directory directly
claude --plugin-dir ./plugins/crew
/reload-plugins        # after editing the manifests
```

Edits to a skill's `SKILL.md` take effect immediately in-session; manifest
changes (`plugin.json` / `marketplace.json`) need `/reload-plugins` — run it too
if a skill edit doesn't seem to apply.

> **Note:** the `--plugin-dir` form has no marketplace context, so it won't
> resolve the `skill-codex` dependency — install it separately for the Codex
> passes to run. (The marketplace-add form above resolves it the normal way.)

## Updating

- **crew itself** is versioned in `plugins/crew/.claude-plugin/plugin.json`.
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
