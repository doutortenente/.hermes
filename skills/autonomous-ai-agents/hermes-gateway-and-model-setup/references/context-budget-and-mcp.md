# Context Budget & MCP Wiring (Hermes, non-interactive)

Companion to SKILL.md. Covers cutting the per-message prompt cost and
registering MCP servers without a TTY. All verified against a live install.

---

## 1. Measure first — `hermes prompt-size` is the instrument

Never estimate the prompt budget; print it. This is the only honest way to
report a change to the user.

```bash
hermes prompt-size | head -20     # totals + per-toolset schema sizes
hermes prompt-size | tail -25     # per-skill index cost
```

Read it as three separate budgets:

| Block | Typical share | Lever |
|---|---|---|
| Tool schemas | Usually the LARGEST — was 65.4 KB of ~92 KB on one install | `hermes tools disable <toolset>` |
| Skills index | grows ~100 B per installed skill | prune skills / `external_dirs` scope |
| memory + user profile | small, but volatile-tier | keep memory entries tight |

Quote the measured before/after delta to the user. "65.4 KB -> 33.3 KB per
message" lands; "much smaller" is unverifiable and gets called out.

---

## 2. `tool_search` defers MCP/plugin tools — NEVER core tools

Hermes ships progressive disclosure under `tools.tool_search`: deferrable tools
are replaced in the model-facing array by three bridge tools
(`tool_search` / `tool_describe` / `tool_call`) and their schemas are fetched on
demand.

**The hard limit, stated in `hermes_cli/config_defaults.py` (~line 2634):**

> Core Hermes tools (terminal, read_file, write_file, patch, search_files,
> todo, memory, browser_*, etc.) are NEVER deferred.

So when a user asks for "load everything on demand," the honest answer is:

- **MCP + plugin tools** -> genuinely deferrable, tune `tool_search`.
- **Core toolsets** -> the ONLY lever is disabling the toolset outright.

Do not imply core tools became lazy. They did not.

Aggressive deferral settings:

```yaml
tools:
  tool_search:
    enabled: 'on'            # always defer; 'auto' only engages conditionally
    threshold_pct: 2         # listing budget as % of context (default 5)
    listing_max_tokens: 1200 # cap on embedded catalog (default 4000)
    listing: auto
```

## 3. Choosing which core toolsets to disable

Disable on evidence about how THIS user works, not by generic ranking. Confirm
the intended use before cutting anything the user might rely on.

```bash
hermes tools list                  # see enabled/disabled per platform
hermes tools disable <toolset>
hermes prompt-size | grep -E "Tool schemas|System prompt total"
```

Signals that a toolset is pure overhead on a given box:

- `hermes doctor` reports `system dependency not met` for it — it cannot run at
  all, so its schema is pure toll (e.g. a browser toolset with no browser deps).
- The delivery surface cannot use it (a text-only terminal session gets nothing
  from text-to-speech).
- The user's domain never touches it (image/video generation for a clinical or
  backend workflow).

Re-enable is one command, so a wrong cut is cheap — but say which ones you cut
and why, so the user can veto.

---

## 4. Registering MCP servers without a TTY

`hermes mcp add` prompts interactively (auth question, then a secret prompt)
and is a dead end in a non-TTY environment. Write the `mcp_servers` block
directly, then verify through the CLI.

**Look for an existing MCP config on the box before inventing one.** A user who
already runs another agent usually has a working `.mcp.json` (Claude Code
convention: project root or `~/.claude.json`). Reuse those exact endpoints and
env vars — same schema shape, already known-good.

```yaml
mcp_servers:
  obsidian:                                   # stdio transport
    command: npx
    args: ['-y', 'obsidian-mcp-server']
    env:
      OBSIDIAN_API_KEY: '...'
      OBSIDIAN_BASE_URL: 'http://127.0.0.1:27123'
      OBSIDIAN_VAULT_PATH: '/home/<user>/vaults/<vault>'
    enabled: true
  supabase:                                   # http transport
    type: http
    url: 'https://mcp.supabase.com/mcp?project_ref=...&read_only=true'
    auth: oauth                               # REQUIRED for OAuth — see below
    enabled: true
```

Remember the config.yaml write race (reference §1 of the delegation doc): after
a direct YAML write, verify immediately with `hermes mcp list`.

### The `auth: oauth` prerequisite

`hermes mcp login <name>` refuses with:

```
✗ Server '<name>' is not configured for OAuth (auth=None)
```

unless the server entry carries `auth: oauth`. The login subcommand reads that
key to decide whether an OAuth flow is even applicable — adding the key first
is what unblocks the flow. `hermes mcp login` takes no `--no-browser` flag; run
it under `background=true, pty=true` and read the URL out of the process log.

### Verify per server, and expect partial success

```bash
hermes mcp list                 # registration
hermes mcp test <name>          # actual connection + tool enumeration
```

`hermes mcp test` failing with "Server returned an error response" on an HTTP
server usually means auth, not a broken endpoint. Report per-server status
honestly: one MCP connected and one blocked on the user's OAuth is a normal,
reportable end state — not a completed setup.

### Deferred MCPs cost nothing while idle

After registering servers, re-run `hermes prompt-size`. With `tool_search`
active the tool-schema total should be **unchanged**. That unchanged number is
the proof the deferral works — show it instead of asserting it.

---

## 5. Verification checklist

| Change | Command that proves it |
|---|---|
| Toolset disabled | `hermes prompt-size \| grep "Tool schemas"` before/after |
| tool_search tuned | `hermes config get tools.tool_search` |
| MCP registered | `hermes mcp list` |
| MCP actually reachable | `hermes mcp test <name>` (enumerates real tools) |
| Deferral working | `hermes prompt-size` unchanged after adding MCPs |
