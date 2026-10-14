# Pravah Todo for Omarchy

A full front-end for the Pravah CLI living in the Omarchy bar: every v2 CLI
capability in a native panel — no terminal needed for daily planning.

## What it does

**Views** (tabs, with live counts):

- **Today** — tasks scheduled today, with overdue pinned at the top in red
  and a collapsible "Completed" section underneath.
- **Inbox** — unscheduled tasks for quick triage.
- **Upcoming** — the next 14 days grouped by date, like `pravah upcoming`.
- **Goals** — goals with linked-task progress bars, add / edit / remove.

**Actions** (all writes go through the CLI's dry-run → apply safety flow):

- Quick capture with tokens: `Draft spec !p1 @work ~30m 9:30` sets priority,
  tags, estimate, and time inline. `⋯` opens the full form (description,
  date picker, time, priority, tags, estimate).
- Complete a task from its checkbox; click again to reopen.
- Row menu (`⋯`): change date (mini calendar), move to Inbox, remove
  (with a confirm dialog; removal stays recoverable through Undo).
- Edit any task field, including notes, via ✎.
- Undo the last write from the toast while the operation is still recoverable.
- Search plus priority and tag filters across all task tabs.
- Health dot in the header: `pravah doctor` / `auth status` run once at
  startup; a read-only credential disables actions with a clear message.

The bar icon keeps a live badge of today's remaining tasks and turns red
when anything is overdue. The last good list is kept visible when a refresh
fails.

## Requirements

- Omarchy Quattro
- Pravah CLI v2.4.0 or later, available on the shell's `PATH`. The `watch`
  command used by the default transport shipped in v2.4.0.
- An authenticated Pravah credential with `tasks:read`; task changes also need
  `tasks:write`.

Check local readiness with:

```bash
pravah doctor --json
pravah auth status --json
```

## Install

Install and enable the plugin from its public repository:

```sh
omarchy plugin add https://github.com/Snehit70/pravah-omarchy-plugin.git --enable
```

Validate the installed plugin with Omarchy:

```sh
omarchy plugin validate "$HOME/.config/omarchy/plugins/raja.pravah-todo"
```

Check that `pravah` is available and authenticated:

```sh
pravah doctor --json
pravah auth status --json
```

The plugin needs an authenticated credential. Read-only credentials can view
lists but cannot change tasks or goals.

## Run the watch service

The default `watch` transport reads a local snapshot maintained by
`pravah watch`. Install the optional user service bundled with this repository:

```sh
install -Dm644 pravah-watch.service ~/.config/systemd/user/pravah-watch.service
systemctl --user daemon-reload
systemctl --user enable --now pravah-watch.service
```

The unit expects the CLI at `~/.local/bin/pravah`. If `command -v pravah`
shows a different path, update `ExecStart` in the copied unit before enabling
it. To run without the service, set `transport` to `cli` in the plugin settings;
that polls the HTTP API instead.

## Keep the snapshot fresh

The widget defaults to the `watch` transport: `pravah watch` holds a Convex
websocket and publishes a snapshot file, which the widget reads. That replaces
the old `tasks list --all` poll, which re-scanned every task and task-image row
on a timer whether or not anything had changed. Cached query reads are not
charged database bandwidth, so an idle bar costs almost nothing.

Watch mode needs that daemon running. Without it the widget reports an error
rather than quietly falling back to HTTP polling, since stale data behind a
working-looking panel is worse than a visible failure.

Check the service status:

```bash
systemctl --user status pravah-watch.service
```

Set `transport` back to `cli` in the plugin settings to poll over HTTP instead.
That path needs no daemon, at the cost of the periodic full read.

Writes always go over HTTP in both transports.

## Controls

- Left click opens or closes the panel; right click refreshes immediately.
- `Today | Inbox | Upcoming | Goals` tabs across the top.
- Type in the quick-add field and press Enter to capture; tokens
  `!p1`, `@tag`, `~30m`, and `9:30` are parsed out of the title.
- Checkbox completes / reopens; ✎ edits; ⋯ opens the row menu.
- Escape closes the panel (or the topmost overlay first).

## Keybindings and scripting

The widget registers an IPC target, so the panel can be summoned without
touching the bar icon. Bind it in Hyprland, e.g.:

```ini
bindd = SUPER ALT T, exec, omarchy-shell raja.pravah-todo toggle
```

Available methods: `open`, `close`, `toggle`, `refresh`.

## Remove

Stop and remove the optional watch service if you installed it, then remove
the plugin through Omarchy:

```sh
systemctl --user disable --now pravah-watch.service
rm -f ~/.config/systemd/user/pravah-watch.service
systemctl --user daemon-reload
omarchy plugin remove raja.pravah-todo
```

If you did not install the service, run only the final `omarchy plugin remove`
command.

## Settings

Configurable through the widget's settings (shell.json entry or the shell's
widget settings UI):

- `transport` — read transport, default `watch`. Set to `cli` to poll the
  HTTP API instead of reading the snapshot maintained by `pravah watch`.
- `pollIntervalSec` — refresh cadence, default 300 (min 10). Right click still refreshes immediately. Ignored in `watch` transport.
- `defaultTab` — which tab opens on click, default `today`.
- `showCompleted` — show the completed section on Today, default `On`.

### Watch transport

With `transport: watch`, the widget reads
`$XDG_RUNTIME_DIR/pravah/snapshot.json` through a `FileView` and repaints when
the file changes. Reads cost no network and no process spawn. The path is
resolved once at startup via `pravah watch --path`, so the XDG fallback rules
stay in the CLI rather than being duplicated in QML. Writes still go over HTTP.

The widget does not fall back to polling when the snapshot is missing or stops
updating. It shows the failure instead, because silently reverting to polling
would hide a dead `pravah watch` behind data that looks current. The daemon
heartbeats the snapshot every minute while idle, and the widget rechecks
freshness on every file change, on a 30s timer, and on manual refresh (right
click, Refresh button, or the `refresh` IPC method).

Run `pravah watch` yourself, or under a supervisor:

```sh
pravah watch
```

Note that the subscription only carries today's completions, so goal progress in
watch transport counts completions within the current day.

## Development checks

The data layer has a QML contract suite that runs against a fake `pravah`
stub (v2 envelope only, no network, no live writes):

Run the suite from the main Pravah repository with `bun run todo-plugin:test`.

It needs `quickshell` and a Wayland/X11 display. Machines without the
Omarchy shell kit skip with exit 0.

The suite covers envelope parsing, task/goal/operation normalization,
today/overdue/upcoming/inbox horizons, quick-add tokens, argv builders,
filters, health, and the dry-run → apply write pipeline with a shared
idempotency key.

## Notes

- The widget only ever runs the v2 CLI contract (`tasks list/add/edit/
  complete/reopen/schedule/unschedule/remove`, `goals …`,
  `operations list/undo`, `doctor`, `auth status`). Every write previews
  with `--dry-run` first and applies with the same argv plus a shared
  `--idempotency-key`, so the dry-run/apply pair for one submission is
  safe. Each new submission mints a fresh key, so a manual retry after
  an ambiguous result is not deduplicated.
- `auth login` / `auth logout` are intentionally not in the panel —
  credentials should never flow through a GUI surface.
