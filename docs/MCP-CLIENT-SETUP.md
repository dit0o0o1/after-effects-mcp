# Connecting this MCP server to a client (Registration & `.mcp.json`)

This document explains how to "register" the After Effects MCP server with an AI client
(Claude Code, Claude Desktop, Cursor, …), how the different registration methods differ,
and exactly how `.mcp.json` works. It also covers what survives a Claude restart and a PC
reboot.

> TL;DR for this machine: we use the project **`.mcp.json`** (already configured and path-
> corrected). You must start a **fresh Claude Code session** once to load + approve it.
> After that it persists across PC reboots; the only recurring manual step is reopening the
> bridge panel inside After Effects.

---

## 1. What "registration" actually means

The MCP "server" is just a normal program — here, the command:

```
node <repo>/build/index.js
```

It speaks the Model Context Protocol over **stdio** (standard input/output). It does **not**
run on its own and is **not** a network service you start once. Instead, the *client* (e.g.
Claude Code) launches it as a child process when a session starts, talks to it over
stdin/stdout, and shuts it down when the session ends.

"Registering" the server = adding **one config entry** that tells the client *how to launch
it* (the `command` + `args`). That's the whole concept. The differences between methods are
only about **where that entry lives** and therefore **what scope** it applies to.

---

## 2. The registration methods (Claude Code)

| Method | Where the config lives | Scope (when it applies) | Shared / committable? | Needs approval? |
|---|---|---|---|---|
| **Project** (`.mcp.json`) | `<repo>/.mcp.json` | Only when Claude Code's working folder **is this project** | ✅ Yes (in repo) | ✅ Yes, once per machine |
| **Local** | `~/.claude.json` → `projects[<path>].mcpServers` | Only this project, **only you** | ❌ No (private) | ❌ No |
| **User / global** | `~/.claude.json` → top-level `mcpServers` | **Every** project/folder for your user | ❌ No (private) | ❌ No |

On this PC the `claude` CLI is **not on PATH** (you run Claude Code embedded in the Claude
desktop app, which doesn't ship the standalone CLI). So `claude mcp add -s local|project|user`
is unavailable — registration here is done by editing config files (`.mcp.json` for project
scope).

### Not the same thing: Claude Desktop *chat* config
`claude_desktop_config.json` (Claude Desktop → Settings → Developer → Edit Config) is a
**different product surface** — the plain desktop *chat*, not *Claude Code*. It has its own
separate MCP list. Only configure the server there if you want to drive After Effects from
the chat window. For Claude Code you use the methods in the table above. Putting it in both
places isn't harmful, just a redundant duplicate to maintain.

### Precedence
If the same server name exists in more than one scope, the most specific wins:
**local > project > user**. So a project entry overrides a user entry of the same name when
you're inside that project.

---

## 3. Which method to use, and why (general guidance)

- **Project (`.mcp.json`)** — use when the server *belongs to this repo* and you want it
  version-controlled and shareable. Trade-off: only active when your working folder is this
  project, each machine/teammate must approve it once, and the `args` path is absolute so it
  isn't portable across machines without editing.
- **Local** — use for a *personal, experimental* server you don't want to commit. Same
  "only this project" limitation, but private and no approval prompt.
- **User / global** — use when the tool is *general-purpose and you want it everywhere*,
  regardless of which folder you open. Trade-off: not shared with the repo, and paths are
  still machine-specific.

**What we chose here and why:** **Project `.mcp.json`.** The server ships with this repo,
you primarily work on After Effects from this folder, and it keeps the config next to the
code. If you later want to control After Effects from *any* folder, move/add it to **user
scope** instead (see §6).

---

## 4. How `.mcp.json` works (so Claude can reliably make it work)

### Location & discovery
A file named `.mcp.json` at the **project root**. Claude Code auto-discovers it whenever its
working directory is that project (or inside it).

### Schema (this server — a local stdio server)
```json
{
  "mcpServers": {
    "AfterEffectsMCP": {
      "command": "node",
      "args": ["C:\\Users\\dit0o\\Desktop\\Code\\after-effects-mcp\\build\\index.js"]
    }
  }
}
```
- `mcpServers` — map of `serverName -> config`. The name (`AfterEffectsMCP`) becomes the
  tool prefix: tools appear to the model as `mcp__AfterEffectsMCP__<toolName>`.
- `command` — the executable Claude Code runs (`node`).
- `args` — arguments; here the **absolute** path to the built entry point. On Windows JSON,
  backslashes must be escaped (`\\`). This path must be correct **for this machine** — the
  shipped default pointed at `C:\Users\Daniel\Downloads\...` and had to be fixed.
- `env` *(optional)* — extra environment variables, e.g. `"env": { "FOO": "bar" }`.

> Remote variants also exist (not used here): `{ "type": "sse", "url": "..." }` or
> `{ "type": "http", "url": "...", "headers": { ... } }`. This project is local stdio.

### Lifecycle
1. **Startup only.** Claude Code connects MCP servers when a session **starts**. Editing
   `.mcp.json` (or any MCP config) has **no effect on a running session** — you must start a
   new session.
2. **Approval (trust).** The first time Claude Code sees a server in a project `.mcp.json`,
   it prompts you to approve it (because it runs an arbitrary command). Approve it. The
   approval is recorded in `~/.claude.json` for this project under `enabledMcpjsonServers`
   (e.g. `["AfterEffectsMCP"]`). You can also manage this with the in-session **`/mcp`**
   command.
3. **Launch & transport.** Claude Code spawns `node build/index.js` and communicates over
   stdio. The server's startup lines (`After Effects MCP Server starting…`, etc.) are written
   to **stderr** as logs — they are not part of the protocol and are normal.
4. **Tools become available.** Once connected, the model can call
   `mcp__AfterEffectsMCP__*` tools.

### Checklist for Claude to make it work (do these in order)
1. `build/index.js` exists → if not, run `npm install` (auto-builds) or `npm run build`.
2. `.mcp.json` `args[0]` is the correct **absolute** path on this machine (escaped `\\`).
3. Start a **new** Claude Code session in this folder; **approve** `AfterEffectsMCP` (or
   enable via `/mcp`).
4. Confirm with `/mcp` that the server is connected and tools are listed.
5. The After Effects side must also be live: the **MCP Bridge Auto** panel open in AE with
   **Auto-run commands** checked, and the AE pref *Allow Scripts to Write Files and Access
   Network* enabled. (See the repo README + `docs`/memory for the bridge mechanism.)

---

## 5. Restart & persistence — answering the two key questions

### "Do I still need to restart Claude / start a new session?"
**Yes — once.** MCP servers load only at Claude Code **startup**. The session in which the
`.mcp.json` path was fixed began *before* the fix, so the server isn't loaded. Start a fresh
Claude Code session in this folder and approve `AfterEffectsMCP`. There is no way to hot-load
it into an already-running session.

### "Will it keep working after I reboot my PC and start a new session?"
**Yes.** Here's what persists vs. what you redo:

| Thing | Persists across PC reboot? | Action needed after reboot |
|---|---|---|
| `.mcp.json` (the registration) | ✅ It's a file on disk | None |
| Server **approval** (`enabledMcpjsonServers` in `~/.claude.json`) | ✅ Stored on disk | None (approve only the *first* time) |
| Built server `build/index.js` | ✅ On disk | None (rebuild only if code changes) |
| The MCP server **process** | ▶️ Launched fresh by Claude Code each session | None — Claude Code starts it on demand |
| AE bridge panel **file** (in ScriptUI Panels) | ✅ On disk | None |
| AE pref *Allow Scripts to Write Files…* | ✅ Saved in AE prefs | None |
| AE bridge panel **open + Auto-run on** | ⚠️ AE restores it *if it was open when you quit* (workspace state) | Usually nothing; if it's missing, reopen via `Window > mcp-bridge-auto.jsx` and check *Auto-run commands* |

So after a reboot, AE normally **restores the panel automatically** (it reopens panels that
were open when you last quit). Just confirm **Auto-run** is checked. Only if the panel isn't
restored do you need to reopen it via `Window > mcp-bridge-auto.jsx`.

> Optional: to force the panel open on every launch regardless of workspace state, place a
> small startup script in AE's `Scripts/Startup` folder that opens it. Not required; ask
> Claude to set this up if you want it.

---

## 6. Common tasks

- **Make it work in every folder (user scope):** add the same `AfterEffectsMCP` block to the
  top-level `mcpServers` in `~/.claude.json` instead of (or in addition to) `.mcp.json`.
  ⚠️ Don't hand-edit `~/.claude.json` while a Claude Code session is running — it can be
  overwritten when that session exits. Edit it with Claude Code closed, then start a session.
- **Verify connection:** run `/mcp` in a Claude Code session.
- **After changing server code:** `npm run build`, then start a new session (the server is
  re-launched fresh, so no other action needed).
- **After changing the bridge panel (`.jsx`):** `npm run build` **and** re-copy the panel
  into AE's ScriptUI Panels folder, then restart/reopen the panel in AE.

---

## 7. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `AfterEffectsMCP` not listed in `/mcp` | Session started before config was added, or not approved | New session; approve, or `/mcp` → enable |
| Tools listed but commands hang / time out | AE panel not open, Auto-run off, or write permission off | Open panel, check Auto-run, enable *Allow Scripts to Write Files and Access Network*, restart AE |
| Server fails to launch | Wrong/way `args` path, or `build/index.js` missing | Fix the absolute path in `.mcp.json`; `npm run build` |
| Worked before reboot, not after | Bridge panel simply isn't reopened | `Window > mcp-bridge-auto.jsx`, ensure Auto-run on |
