# Preferences — After Effects MCP behaviors & mechanics

A reference for the runtime mechanics and behavioral "rules" of this MCP — the knobs and
defaults that determine how it behaves, like an app's preferences. For *connecting it to a
client*, see [MCP-CLIENT-SETUP.md](./MCP-CLIENT-SETUP.md).

## Architecture: a file bridge

There is no direct connection between the MCP server and After Effects. They communicate
through three files in `~/Documents/ae-mcp-bridge/`:

| File | Written by | Read by | Purpose |
|------|-----------|---------|---------|
| `ae_command.json` | MCP server | AE panel | the command to run + args |
| `ae_mcp_result.json` | AE panel | MCP server / tools | the result of the last command |
| `ae_bridge_log.txt` | AE panel | humans / tools | rolling panel log (see Logging) |

The `mcp-bridge-auto.jsx` panel, while open with **Auto-run** on, **polls `ae_command.json`
every 2 seconds**, runs one `pending` command at a time, writes `ae_mcp_result.json`, and marks
the command `completed` (or `error`). Commands are processed **one at a time, in order**.

## Behaviors / rules ("preferences")

| Behavior | Rule |
|----------|------|
| **Auto-open on create** | `createComposition` opens the new comp in the Composition viewer automatically (`openInViewer()`). |
| **Comp targeting** | `openComposition` (and similar) can target a comp by **`name`**, AE **`id`**, or 1-based **`index`** (position among comps). If several are given, the first match wins in project order. |
| **ID semantics** | `item.id` is assigned and owned by After Effects, stored in the project. It is **unique within a project** and **persists across save → reopen of the same `.aep`**. It is **per-project**: a different project has independent ids (a fresh project's first comp is also `id: 1`); copy/import yields a new id; an **unsaved** project's items (and ids) are lost on AE restart. Most stable handle — unaffected by reordering, unlike `index`; not ambiguous, unlike `name`. |
| **Undo grouping** | Each command runs inside one AE undo group named `MCP: <command>` → it shows as e.g. *"Undo MCP: createComposition"* and **one Ctrl+Z reverts that whole command**. See [Undo behavior](#undo-behavior) for multi-command requests. |
| **Stateless project context** | Every command reads the **currently open** `app.project` live; nothing about the project is cached. Switching projects / opening blank just works. Best practice: re-query `getProjectInfo`/`listCompositions` at the start of work and after any project switch rather than assuming earlier state. |
| **Result metadata** | Every result file gets `_commandExecuted` (the command name) and `_responseTimestamp` (ISO, UTC) appended, for freshness/identity checks. |
| **Logging** | `logToPanel()` writes to the panel UI **and** appends to `ae_bridge_log.txt`, which auto-rotates (overwrites) once it passes ~512 KB. |
| **AE 2025+ UI** | On AE 2025+ the panel runs as a **floating palette only** (dockable panels unsupported). Opening it from the `Window` menu also leaves a separate **blank docked panel** — harmless; the floating window is the functional one. |
| **Result staleness** | If `ae_mcp_result.json` hasn't been updated within ~30 s, the server flags the result as possibly stale (usually means the panel isn't running / Auto-run is off). |

## Undo behavior

- **Per-command granularity.** Each MCP command is wrapped in a single AE undo group, so one
  Ctrl+Z reverts that whole command — including commands that change many things at once (e.g.
  `batchSetLayerProperties`). The group is labeled `MCP: <command>` in AE's Edit menu / History.
- **A multi-step request = multiple undo steps.** A request fulfilled as several commands (e.g.
  create comp → add title → add subtitle) produces **one labeled undo step per command**, undone
  individually in reverse order with repeated Ctrl+Z.
- **Shared, chronological stack.** MCP actions live on AE's normal global undo stack, interleaved
  with your manual edits in the order they occurred.
- **Not implemented (yet):** collapsing an entire multi-command request into a *single* undo step
  (a "transaction"). Possible future addition; would require handling AE's nested-undo-group
  semantics.

## Available commands

Executed by the panel (the `command` field in `ae_command.json`):

`getProjectInfo`, `listCompositions`, `getLayerInfo`, `createComposition`, `openComposition`,
`createTextLayer`, `createShapeLayer`, `createSolidLayer`, `setLayerProperties`,
`batchSetLayerProperties`, `setCompositionProperties`, `setLayerKeyframe`,
`setLayerExpression`, `applyEffect`, `applyEffectTemplate`, `createCamera`, `duplicateLayer`,
`deleteLayer`, `setLayerMask`, `bridgeTestEffects`.

Exposed to the client either as **dedicated MCP tools** (`create-composition`,
`setLayerKeyframe`, `setLayerExpression`, `apply-effect`, `apply-effect-template`,
`run-bridge-test`, plus `get-results` / `get-help`) or via the generic **`run-script`** tool,
whose allow-list lives in `src/index.ts` (`allowedScripts`). Adding a command means: implement
it in the panel's `switch`, and (for `run-script` access) add its name to `allowedScripts`.

> Note: changes to `src/index.ts` (the server) take effect after the **client** restarts;
> changes to the panel `.jsx` take effect after **After Effects** restarts (or the panel is
> reloaded). See [MCP-CLIENT-SETUP.md](./MCP-CLIENT-SETUP.md).

## Requirements to run

1. AE preference **Allow Scripts to Write Files and Access Network** enabled (the panel reads
   and writes the bridge files).
2. The **MCP Bridge Auto** panel open with **Auto-run commands** checked.

## Key file locations (this machine)

- Bridge dir: `C:\Users\dit0o\Documents\ae-mcp-bridge\`
- Server entry: `…\after-effects-mcp\build\index.js`
- Panel source: `…\after-effects-mcp\src\scripts\mcp-bridge-auto.jsx` → built to `build\scripts\` → installed to `…\Adobe After Effects 2026\Support Files\Scripts\ScriptUI Panels\`
