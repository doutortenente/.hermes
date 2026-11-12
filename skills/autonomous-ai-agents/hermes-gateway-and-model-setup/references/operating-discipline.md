# Operating Discipline for Hermes Config Work

Cross-cutting rules for how to RUN a Hermes configuration pass, independent of
which subsystem is being configured. Read this before starting; the subsystem
detail lives in SKILL.md and the other references.

---

## 1. Run the whole chain; don't hand steps back

Config work is a sequence of small verifiable steps. Execute it end to end and
verify each step yourself from logs, `hermes status`, `hermes prompt-size`, or
a real dispatch. Do **not** stop between steps to ask the user to confirm an
intermediate result, and do not hand them a command to paste into their own
terminal.

This is the single most common complaint against this workflow. Phrasings that
mean you already over-asked: *"just do it"*, *"execute without interruption"*,
*"don't ask me to test"*, *"skip the manual confirmation"*.

Only two things legitimately pause the run:

1. **A secret or authorization only the user holds** — a bot token, an OAuth
   approval. Ask once, hand over the URL verbatim, then continue.
2. **A destructive or irreversible choice** — deleting, overwriting, acting on
   their behalf, or anything with real spend.

Everything else — restarting the gateway, editing config, disabling a toolset,
probing models, dispatching a test subagent — just do it, then report the
measured result.

When a step IS blocked on the user, do not idle on it. Park it, finish the rest
of the pass, and list the blocked item at the end. A partially-blocked pass that
completed nine of ten steps is a good outcome; a pass that stalled on step one
waiting for a reply is not.

## 2. Verify with the CLI, not with the file you just wrote

Every subsystem in this skill failed at least once in a way that looked like
success from the config file alone. Reading back your own YAML write proves
nothing — a running gateway can revert it moments later (see §1 of the
delegation reference).

The CLI is the source of truth:

| Change | Command that proves it |
|---|---|
| Model/provider | `hermes status \| grep -A2 "Model:"` |
| Delegation | a real `delegate_task` returning a checkable fact |
| Fallback chain | `hermes fallback list` + per-model tool-calling probe |
| Toolset/context cost | `hermes prompt-size` before/after |
| MCP | `hermes mcp list` then `hermes mcp test <name>` |
| Gateway | `journalctl --user -u hermes-gateway -n 20` + a user-sent message |

Prefer a probe that returns a **checkable fact** (a hostname, a byte count, an
enumerated tool list) over one that returns "OK".

## 3. Report measured numbers, and separate what is not working

- Lead with the outcome, not the process.
- Use measured values: `65.4 KB -> 33.3 KB per message`, `83 -> 113 skills`.
  Never "much smaller" or "significantly faster" — unverifiable adjectives get
  called out, and rightly.
- Give unfinished/blocked/unverified items their **own section**. Do not bury a
  blocker inside a success summary.
- State plainly what you did NOT verify. "Config is correct and the targets are
  live, but I have not seen the failover actually fire" is a legitimate and
  useful thing to say; implying it was tested is not.
- Never present a config write as a working feature.

## 4. Test what the user will actually experience

A test that exercises a different code path than the user's real usage proves
less than it appears to.

Concrete case: calling the Telegram Bot API `sendMessage` proves the token is
valid. It does **not** exercise the gateway's receive path — the thing that
actually breaks. Only a message the user sends from their client tests that.

Before reporting a component as working, ask which path the test drove and
whether it is the same one the user will hit.

## 5. Prefer pointing at the user's existing config over creating a parallel one

Users who already run another agent usually have working config on the box —
an `.mcp.json`, a skills repo, a credentials file. Read it and reuse those
exact values instead of authoring a second copy.

This respects "single home" conventions users often maintain deliberately (and
document in their own repo rules), and avoids a duplicate that silently drifts.
`skills.external_dirs` and reused `mcp_servers` entries are both pointer-style
integrations — edits at the source are visible immediately, with nothing to
sync.

Corollary: anything imported this way is **externally owned**. Never edit it
from Hermes. If it needs a change, say so and let the user make it in their
repo.

## 6. Respect the user's own repo rules

If the user's project carries an `AGENTS.md` / `CLAUDE.md` / rules directory,
those constraints bind your actions there. Commonly: direct commits to `main`
forbidden (branch -> PR -> merge), secrets confined to one vault file, scripts
confined to one directory.

When you notice a dirty working tree or a deleted hook while working in such a
repo, report it — do not quietly fix or commit it.
