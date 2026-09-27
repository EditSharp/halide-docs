# Remote API

A running app can be driven from another program over a local pipe: a named pipe on Windows, a Unix socket on macOS
and Linux. Messages are [JSON-RPC 2.0](https://www.jsonrpc.org/specification), one per line, UTF-8.

## Starting it

| Flag (after `--`) | What it does |
|---|---|
| `--api` or `--api=NAME` | Serves on a pipe named `editsharp-<pid>` or `NAME`. |
| `--script=FILE.json` | Runs a list of calls, prints each answer, and quits: exit code 0 if all succeeded, 1 otherwise. |
| `--user-data=DIR` | Keeps settings, recent projects, layouts, shortcuts, theme and extension choices in `DIR`, leaving the user's own alone. |
| `--extensions=DIR` | Loads extensions from `DIR` instead of the usual folder. |
| `--trust-extensions` | Enables new and changed extensions without asking. |
| `--dump-commands=FILE.md` | Writes the command catalogue and quits. |

Add Godot's own `--headless` (before the `--`) to run with no windows. Views still exist, so every command works, and
dialogs answer themselves with their Cancel button. With `--api`, a headless app quits once its last client
disconnects.

```
godot --headless --path edit-sharp-gui -- --api=my-pipe --user-data=/tmp/es
```

The running app writes where to connect to `api.json` in its user data folder:
`{"pipe": "...", "pid": 123, "version": "1.0", "socket": "/tmp/CoreFxPipe_..."}`. The `socket` field only appears on
macOS and Linux.

## Methods

| Method | Params | Result |
|---|---|---|
| `api.version` | none | `{"version", "pid"}` |
| `command.list` | `project`? | every command: `id`, `title`, `enabled`, `checked` |
| `command.run` | `id`, `args`?, `project`? | the command's result |
| `state.get` | `path`, `project`? | the state at `path` (below) |
| `events.subscribe` | `names` | the names that exist |
| `events.unsubscribe` | `names` | what's still subscribed |

`project` names an open project by name or file path; without it, the focused project is used.

**State paths**:

| Path | What it holds |
|---|---|
| `app` | the version, open projects and the focused one |
| `project` | name, file, whether it's dirty, the active layout |
| `timeline` | the playhead, duration, zoom, and channels with their clips (id, name, kind, start, end, speed, link, selected) |
| `media` | id, name, path and kind of each media |
| `layout` | the active layout, every layout, the views (open, floating) and the arrangement |
| `playback` | state, playing, position, loop |
| `history` | whether undo and redo are possible, and what they'd do |
| `commands` | the same as `command.list` |

Times are seconds; clips and media are named by id.

**Events** arrive as notifications (no `id`) once subscribed:

| Event | When |
|---|---|
| `project.opened`, `project.closed` | a project opens or closes |
| `history.changed` | an edit, undo or redo |
| `layout.changed` | the layout or arrangement changes |
| `media.changed` | the project's media changes |
| `playback.position` | playback moves; at most ten a second |

**Errors** use JSON-RPC's codes, with `-32000` when a command doesn't exist, can't run, or fails.

## Example session

```
→ {"jsonrpc":"2.0","id":1,"method":"command.run","params":{"id":"project.create","args":{"folder":"/tmp","name":"Demo"}}}
← {"jsonrpc":"2.0","id":1,"result":"/tmp/Demo/Demo.esproj"}
→ {"jsonrpc":"2.0","id":2,"method":"events.subscribe","params":{"names":["history.changed"]}}
← {"jsonrpc":"2.0","id":2,"result":["history.changed"]}
→ {"jsonrpc":"2.0","id":3,"method":"command.run","params":{"id":"timeline.addChannel","args":{"video":false}}}
← {"jsonrpc":"2.0","method":"history.changed","params":{"project":"Demo","action":"Commit","undo":"Add audio channel"}}
← {"jsonrpc":"2.0","id":3,"result":3}
```

## Python client

`edit-sharp-gui/Tools/editsharp_rpc.py` finds a running app, or takes a pipe name:

```python
from editsharp_rpc import Client

with Client.connect("my-pipe") as app:
    app.command("project.create", {"folder": "/tmp", "name": "Demo"})
    print(app.state("timeline")["channels"])
```

From a shell: `python Tools/editsharp_rpc.py state.get '{"path": "app"}'`, or
`--watch history.changed` to print events as they come.
