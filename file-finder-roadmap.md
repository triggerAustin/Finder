# File Finder — Project Roadmap

## Purpose

A tool to find files, folders, packages, or projects already on the system that you know exist somewhere but can't remember the location of — and beyond that, to remember *how* you interact with them, so re-opening something takes as few keystrokes as possible instead of re-navigating (`cd`, `cd`, `nvim`...) every time. Triggered by a global keybind, it opens a small popup, checks your own command history first, falls back to a system-wide search for anything not yet in that history, and lets you jump straight to opening it the same way you did last time.

The project is split into phases so a usable daily tool ships quickly, without blocking the separate goal of eventually hand-building the search engine itself.

---

## Design choices made so far

| Decision | Choice | Why |
|---|---|---|
| Display/input layer | **rofi**, script mode | Built for "popup, type, filter, select"; script mode lets a backend script control what's shown |
| Search backend (Phase 1) | **`plocate`** | Index-based, fast root-wide queries; `find` rejected as too slow for live use (walks the live filesystem each call) |
| Search scope | **Root (`/`)**, unscoped | Matches the primary use case; project-local scoping deferred |
| What's actually tracked | **Commands run against paths**, not just paths | The real goal is remembering *how* a file/folder is normally opened, not just where it is |
| History reset windows | **Calendar week / calendar month**, not install-date or per-path clocks | Predictable, shared reset points; avoids needing a stored per-path window-start field |
| Global metadata (tracking start date) | Stored in a separate `history.meta` file | Keeps the history file itself as pure per-row data |
| Opening unfamiliar files (images, video, etc.) | Defer to `xdg-open` / the system's MIME default | Solved problem at the OS level — no need to build a type-to-app mapping |

---

## Architecture: syscall vs. build

**Syscall — existing tools, just invoked or configured:**
- **rofi** — display, input capture, custom keybinding support (for the history-filter toggle)
- **`plocate`** — system-wide path search, index-based
- **`xdg-open` / `xdg-mime`** — resolves the correct app for non-text files, using the system's own file-association config
- **Keybind** — WM/DE configuration, launches rofi
- **Bash `DEBUG` trap** — built into Bash itself; used as the single dispatch point for the command-tracking hook

**Build — logic that has to be written:**
- Query-vs-selection branch (rofi's `ROFI_RETV` states)
- History file read/write logic (per-row lookups, frequency updates, calendar-based resets)
- History + `plocate` merge logic (set-difference to avoid duplicate paths)
- The history-filter toggle (custom rofi keybinding branch)
- The "which command to use" dialogue for paths with no history yet
- The command-tracking hook itself (which commands to watch for, how to extract a path from each one's arguments)
- Terminal/file-manager/editor invocation once a command is chosen or replayed

---

## Project structure

```
file-finder/
├── bin/
│   └── file-finder         # the glue script — rofi calls this via -modi
├── hooks/
│   └── cd-hook.sh          # Bash DEBUG trap: tracks cd + opener commands, sourced from .bashrc
└── data/
    ├── history              # per-command history rows
    └── history.meta         # tracking start date
```

`~/.config/rofi/rofi-mode.rasi` (or equivalent) also needs to exist for rofi to register the custom mode — it lives in rofi's own config directory, not in this project folder, since that's the only place rofi looks for it.

---

## History file — row shape

One row per **(command, path)** pair — not per path alone, since the same path can have multiple different commands tied to it (e.g. sometimes opened with `vim`, sometimes `nano`).

| Field | Meaning |
|---|---|
| cmd | the full command that was run (e.g. `nvim`) |
| path | the target file/folder |
| last_visited | most recent time this exact command was run against this path |
| freq_7d | count so far in the current calendar week |
| last_week | frozen count from the most recently completed calendar week |
| freq_30d | count so far in the current calendar month |
| last_month | frozen count from the most recently completed calendar month |

**Recording a visit:**
1. Compare current ISO week / calendar month against the week/month implied by this row's `last_visited`
2. If the week has changed: `last_week = freq_7d`, then `freq_7d = 0`
3. If the month has changed: `last_month = freq_30d`, then `freq_30d = 0`
4. Increment `freq_7d` and `freq_30d`
5. Update `last_visited`

`history.meta` holds one value: the date the history file was first created, used only for the empty-state message.

---

## Query flow

**On rofi launch (`ROFI_RETV = 0`, no query yet):**
- If history is empty → display an instructional message (tracking start date, how to search, how the toggle works)
- Otherwise → display history rows only (no `plocate` call yet), sorted by recency by default

**On query submitted (`ROFI_RETV = 2`, custom entry):**
1. Search history for rows whose `path` matches the query
2. Search `plocate` for the same query
3. Merge: history rows first (each tagged with its `cmd`, `last_visited`, `freq` columns, and a `most_visited` marker on whichever row has the highest freq when a path has multiple commands tied to it), then `plocate` results — excluding any path already present in the history matches (set-difference, to avoid duplicates)
4. Display combined list, formatted with columns: `cmd | path | last_visited | freq`

**Toggle (bound custom key, distinct `ROFI_RETV` state):**
- Re-sorts/filters **only the history-sourced rows currently displayed** by 7-day or 30-day frequency
- `plocate`-sourced rows in the current view are left untouched, persisting as-is
- If the visible history rows have no activity in the selected window, that portion goes empty while any `plocate` rows remain
- On a fresh launch with no query yet, this simply re-sorts the full history list, since that's all that's displayed at that point

**On row selection (`ROFI_RETV = 1`):**
- Row has a stored `cmd` → run it directly against the path, update that row's history fields
- Row came from `plocate` with no history yet → show a dialogue for how to open it:
  - Text-like file → choose among configured editors (vim/nano/nvim)
  - Folder → `cd` into it (open terminal there) or open in file manager
  - Anything else (image, video, etc.) → `xdg-open`, deferring to the system's own default app
  - Whichever is chosen becomes the first history row for that path

---

## Phase 2 — Hand-built search engine (separate, later project)

Goal: replace `plocate` behind the same query-in/paths-out contract with a self-built indexer and matcher, using `find` only for raw traversal, writing everything else by hand in Bash.

Build order:
1. Static indexer — `find` walks the disk once, writes paths to a flat log
2. Static substring search against that log
3. Fuzzy scoring, replacing substring matching
4. Live-typing loop with raw keypress capture (a hand-built TUI)
5. Incremental re-indexing (only re-walk changed directories)

This phase has no shipping pressure — Phase 1 already provides a working daily tool, so this is purely for the algorithmic/systems learning.

---

## Phase 3 — The swap

Once Phase 2's search step returns paths in the same shape `plocate` does, point the merge step at it instead. Nothing else — history logic, the toggle, the dialogue, the hook — needs to change if the contract held. This is also where the hand-built engine gets benchmarked directly against `plocate`, using the same rofi front end for both.

---

## Reading list

- Rofi script mode: invocation model, `ROFI_RETV` states, custom keybindings (`man rofi-script`)
- `plocate` command-line usage and exit codes
- Bash `DEBUG` trap and `$BASH_COMMAND` — for the single-dispatcher command-tracking hook
- `xdg-open` / `xdg-mime` — querying and invoking the system's default app per file type
- `date` command for ISO week (`%V`) and month (`%Y-%m`) extraction, for calendar-based resets
- Basic flat-file read/modify/write patterns in Bash (for per-row updates without a database)
