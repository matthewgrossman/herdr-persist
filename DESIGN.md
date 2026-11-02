# herdr-persist: design

Status: draft, nothing built yet. Targets herdr-app 0.1.11 and herdr-core (`hc`) 0.1.16.

## The problem

Some programs should outlive the pane you happen to run them in, and there should be
exactly one of them for some scope. The motivating case is an editor: one Neovim per
project, which you pull into whichever pane you're working in, instead of starting a
new `nvim` in every pane and forgetting which one has your buffers. The same idea fits
a log tail per repo, a test watcher per tab, or a global notes scratchpad.

tmux users build this out of `join-pane` / `break-pane` or `display-popup`; i3 and sway
call it a scratchpad; Neovim plugins like toggleterm do it inside the editor. herdr-app
has the pieces (see below) but no mod that puts them together.

## Why herdr-app makes this easy

- **herdr-core owns every terminal, not the pane.** `hc run --bg --no-parent --name X -- prog`
  starts a terminal that belongs to no tab. It keeps running when panes close and when
  the app restarts.
- **A name is a lookup.** Names are unique among live terminals (1 to 32 of `a-z 0-9 - _`),
  so "is the editor for this scope running?" is `hc get <name>`.
- **The app can put any terminal anywhere:**
  - `terminal.show {terminal, pane}` shows it in a pane in place of that pane's own
    terminal (`terminal.show {pane}` with no terminal goes back).
  - `terminal.keep {terminal, place: "right" | "down" | "tab"}` gives a paneless
    terminal a real pane. `terminal.park {terminal}` takes the pane away and leaves it running.
  - `terminal.peek {terminal}` floats it over the tab, with keep-as-tab and close controls.
  - A mod can open a window overlay holding a terminal block
    (`openOverlay({ kind: "window", key, layout: { ... { content: { terminal } } } })`).

The old herdr TUI couldn't do this: panes owned their programs, so a shared editor had
to be a headless `nvim --listen` with every pane attaching `nvim --remote-ui`. With
herdr-app it's an ordinary program in an ordinary terminal.

## Concepts

An **entry** is a named kind of long-lived terminal in the person's config (`editor`,
`logs`, `notes`). Each entry has one **instance** per scope key: the editor for space A,
the editor for space B. A **view** is somewhere an instance is on screen: a pane showing
it in place, its own pane, or a float.

## Configuration

```json
{
  "editor": {
    "argv": ["nvim", "--listen", "{sock}"],
    "per": "space",
    "place": "right",
    "others": "move",
    "cleanup": "kill"
  },
  "logs": { "argv": ["tail", "-f", "log/dev.log"], "per": "repo", "place": "down", "others": "mirror" },
  "notes": { "argv": ["nvim", "~/notes.md"], "per": "global", "place": "peek", "others": "focus" }
}
```

Where this file lives is open (mod folder, `shared.json` under a `persist.` key, or a
file in the person's app home). It should be editable by hand and live-reload.

### `per`: one instance per what

The scope key is resolved from the focused pane when the entry is opened.

| `per` | One instance per... | Resolved from |
|---|---|---|
| `global` | machine | constant |
| `space` | herdr-app space | `focused().space` (or the `herdr.space` label of `$HC_TERMINAL_ID` from a shell) |
| `tab` | tab | `focused().tab` (or the `herdr.tab` label) |
| `repo` | git root of the focused folder | `git -C <focused().cwd> rev-parse --show-toplevel` |
| `folder` | the focused folder | `focused().cwd` |

"Per window" doesn't map to anything: herdr-app has one main window. Spaces are the
closest thing to a workspace.

### `place`: how a view appears

| `place` | Mechanism | Notes |
|---|---|---|
| `here` | `terminal.show {terminal, pane}` | Replaces the focused pane's view until toggled back. The pane's own terminal keeps running underneath. |
| `right`, `down` | `terminal.keep {terminal, place}`; hide with `terminal.park` | A real pane: pane.go, zoom, headers and `panes()` all see it. Closing that pane the normal way ends the terminal (Herdr ends a pane's terminal on close), so hide it with the toggle. |
| `peek` | `terminal.peek {terminal}` | Built-in float over the tab. |
| `window` | mod window overlay with a terminal block | Draggable, sized, position remembered. Global across spaces, so a per-space entry must swap or hide its contents on space switch. |

The same entry will sometimes want different places. Keys pass one argument at most,
so either one command per place (`persist.editor-right`, `persist.editor-peek`) or a
command that cycles the open view through places. `place` in config is the default.

### `others`: what happens to existing views

| `others` | Opening it somewhere new... | Good for |
|---|---|---|
| `move` | takes it off screen everywhere else first (show back, park, close the float) | full-screen programs: nvim, htop |
| `focus` | jumps to an existing view in this tab instead, if there is one; otherwise like `move` | "where's my editor?" |
| `mirror` | leaves the other views and adds one | `tail -f`, logs, watching an agent |

"Off screen" never kills the instance. A terminal has one size, set by the view used
last (`the last active viewer sets the size`), so full-screen programs look wrong when
mirrored. The default should follow the program: `move` for programs that use the
alternate screen, `mirror` otherwise.

### `argv` placeholders

- `{sock}`: a per-instance socket path, `$XDG_STATE_HOME/herdr-persist/<name>.sock`, so a
  CLI can talk to the program (`nvim --listen {sock}`, then `nvim --server {sock} --remote`).
- `{key}`: the resolved scope key (repo path, folder, space id).
- `{dir}`: the folder to start in (the focused pane's folder; the repo root for `per: repo`).

### `cleanup`: when the scope ends

Herdr doesn't clean up terminals it didn't start. When a space or tab is closed or a
repo folder disappears, the mod chooses: `kill`, `keep` (default), or `ask`. Detecting
this means watching `spaces()` / tab lists, or checking on each open.

### Remote spaces

A space on another machine (`spaces()[i].machine`) should run its instance there:
`hc run --on <machine>`, and `exec(..., { on })` for any shell-outs. Out of scope for the
first version, which should refuse remote spaces with a clear message.

## Identity

- Terminal name: `p-<entry>-<hash of scope key>`, cut to 32 characters. Stable, so any
  process can find the instance with `hc get <name>`.
- Labels on every instance: `herdr.mod=persist`, `persist.entry=<entry>`,
  `persist.key=<scope key>`. Lets the mod list its own instances (`hc ls`) and never touch
  terminals it didn't start (herdr-float-space does the same).
- Start with `hc run --bg --rm --no-parent --name <name> --label ... --cwd <dir> -- <argv>`.
  `--rm` drops the record once the program exits, so `:q` really ends it and the next
  open starts fresh. Don't use the `herdr.space` / `herdr.tab` labels: those make Herdr
  adopt the terminal into a tab as a pane (`terminal.keep` sets them when giving it a pane).

## Tracking views

`panes()` lists each pane's own terminal but not a terminal shown in it with
`terminal.show`; only `focused().shown` reports that, and only for the focused pane.
So:

- For `right` / `down`, `panes()` is the source of truth: find the pane whose `terminal`
  is the instance.
- For `here`, the mod must remember which pane it showed the instance in. That memory is
  lost when the mod reloads or the app restarts, and misses views made by anything else.
- For floats, the mod knows which overlays it opened.

This is a reason to prefer `right` / `down` as the default place.

## CLI

A small script so shells and other programs can use entries:

```
persist open <entry> [--place P]        # show it (start if needed), print its address
persist hide <entry>                    # take it off screen everywhere
persist address <entry>                 # print the instance's address if running
persist edit [--wait] [+LINE] FILE...   # for the editor entry: open files, $VISUAL with --wait
```

`edit --wait` opens the files in the instance (`nvr --remote-wait`), shows it, waits for
the buffers to close, then hides it, so `VISUAL="persist edit --wait"` works for git.
Outside herdr-app (`$HC_TERMINAL_ID` unset) `edit` falls back to plain `nvim`.

The CLI should route through the mod (`herdr-app do persist.<command>`), so view
tracking stays in one place, and fall back to plain `herdr-app do terminal.*` commands
when the mod isn't loaded (`herdr-app do` exits 1 with "need the mod loaded").

## Hosting and keys

`<mod>.<command>` key actions only reach a mod whose block is in the shown layout
(`<mod>.<command> reaches shown copies only`). Two ways:

1. **A block in the layout.** Needs the person to clone their layout and add a 0px block
   (`{"id": "persist", "type": "block", "size": {"px": 0}, "content": {"mod": "persist"}}`).
   The clone stops getting upstream layout updates. Bad first impression for an
   open-source mod. Related: herdr-app #106, #288.
2. **`mod.open:<mod>` with an overlay `mod.json`**, as herdr-float-space does: the key
   opens the mod as a 1x1 window, it does its work in `init` and closes. Works in any
   layout. But `mod.open` takes no props (herdr-app #164), so each distinct action needs
   its own tiny mod folder. Unknown: the delay this adds per key press.

To verify first: whether a `"persist.toggle:editor"` binding delivers `toggle:editor` to
`nextCommand()`, which would allow one command per action with the entry as its argument.

## Herdr limits found so far

- `keys.json` actions can't run a shell command, can't depend on the focused program, and
  pass at most one argument (#164).
- `hc keys` / `terminal.keys` wait until the person has stopped typing in that terminal
  (herdr-core #5, #9, #52). Typing as the person goes as raw DATA on a framed
  `terminal.attach` instead. Not needed here unless the mod forwards keys.
- `panes()` doesn't report a terminal shown with `terminal.show`.
- The app's idea of a pane's program (`panes()[i].program`) can lag many seconds behind
  the core's (`hc get <t> --json` → `.foreground.name`).
- herdr-float-space reports that a mod can't focus a terminal in its window overlay
  (seen on 0.1.10), so the person has to click it before typing. Check on 0.1.11.
- herdr-app terminals can inherit stale `HERDR_*` variables from the old herdr TUI. Use
  `HC_TERMINAL_ID` and the terminal's labels, never `HERDR_WORKSPACE_ID` / `HERDR_PANE_ID`.
- Edits to symlinked config files apply after a ~10 s poll, or at once if the link is
  recreated (#65).

## First version

- Entries from a config file, with an `editor` entry as the built-in example.
- `per`: `global`, `space`, `tab`, `repo`.
- `place`: `here`, `right`, `peek`.
- `others`: `move`, `focus`.
- CLI: `open`, `hide`, `address`, `edit [--wait]`.
- Local machine only.

Later: `mirror`, `down`, `window`, `folder`, `cleanup`, remote spaces, per-program
defaults for `others`.

## Unknowns to settle by experiment, in this order

1. Does `terminal.show` work on a terminal that belongs to no tab, and what happens to
   the pane when that program exits?
2. Does `terminal.keep {place: "right"}` on such a terminal work, does `terminal.park`
   return it to paneless (and does keep stay off), and can it then be kept again elsewhere?
3. Does `terminal.peek` take the keyboard?
4. Does a `<mod>.<command>:<value>` binding pass the value to `nextCommand()`?
5. How long does `mod.open` of a 1x1 overlay mod take from key press to action?

Test in a scratch space of your own and leave the person's existing spaces, tabs and
panes alone: no keys, `terminal.show`, `pane.go` or focus changes on their panes.

## Prior art

- [herdr-float-space](https://github.com/thuanlm215/herdr-float-space): a floating
  scratch window with its own shell tabs and notes, toggled by `mod.open`. Covers
  `place: window` for plain shells; its labels and ownership checks are worth copying.
- The original prototype, in `~/dotfiles` on the author's machine (not yet committed):
  `config/herdr-app/bin/hvim` (one nvim per space, `terminal.show` in place, `--wait` for
  `$VISUAL`) and the `vimnav` mod's `editor` commands in
  `config/herdr-app/mods/vimnav/main.js`, which track views for `here`.
- tmux `join-pane` / `break-pane` / `display-popup`, i3 and sway scratchpads, Zellij
  floating panes, toggleterm.nvim.

## Publishing

A public repo with `mods/persist/`, a `keys.json` of suggested bindings, a `RECIPE.md`
(the brief to rebuild it, per herdr-app's AGENTS.md), a `preview.png`, and the GitHub
topic `herdr-mod`. It then appears at herdr.dev/mods within the hour. Add the topic only
once it works.
