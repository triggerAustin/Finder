# File Finder — Project Roadmap

## Purpose

A tool to find files, folders, packages, or projects already on the system that you know exist somewhere but can't remember the location of. Triggered by a global keybind, it opens a small popup, filters results as you type, and lets you jump straight to the result's location in a terminal. The system-wide, "did I install this at some point" use case is the priority — project-local scoping is a later concern.

The project is deliberately split into phases so a usable daily tool ships quickly, without blocking the separate goal of eventually hand-building the search engine itself.

---

## Design choices made so far

| Decision | Choice | Why |
|---|---|---|
| Display/input layer | **rofi**, script mode | Built exactly for "popup, type, filter, select" — script mode lets a backend script control what's shown instead of relying on rofi's own static-list filtering |
| Search backend (Phase 1) | **`plocate`** | Index-based, so root-wide queries return fast on every keystroke; `find` was rejected for this because it walks the live filesystem each call, too slow for live filtering |
| Search scope | **Root (`/`)**, unscoped | Matches the primary use case (system-wide "does this exist anywhere"); project-local scoping deferred until it's actually needed |
| Matching behavior | Substring (via `plocate`), not fuzzy | Accepted tradeoff for Phase 1 — no typo tolerance yet; fuzzy matching is explicitly a Phase 2 concern if it's built at all |
| Long-term backend | Hand-built indexer + matcher (Bash) | Kept as a separate, unhurried project so learning the algorithm doesn't block having a working tool now |

---

## Architecture: syscall vs. build

The core design principle for Phase 1: **outsource anything that's already a solved problem, write only the glue connecting them.** Every piece of the stack falls into one of two categories:

### Syscall — existing tools, just invoked or configured

- **rofi** — display and input capture. Shows the list, reads what's typed, reads what's selected.
- **`plocate`** — the actual search. Takes a query string, returns matching paths from its own prebuilt index, one per line.
- **Keybind** — window manager / desktop environment configuration (e.g. `sxhkd`, a WM config line, or a GNOME custom shortcut) that fires the rofi popup on a key combo. Not application code.

None of these are written — they're called, piped, or configured.

### Build — logic that has to be written

- **Query-vs-selection branch** — rofi invokes the same script both to fetch fresh results *and* to report a selection. The script has to detect which situation it's in (raw typed text vs. one of its own previously printed lines) and branch accordingly.
- **Output formatting** — reshaping `plocate`'s raw output into whatever line format rofi's script mode expects to display.
- **Terminal-open action** — once a selection is detected, constructing and firing the command that opens a terminal at that path (parent directory if a file was selected, the directory itself if a folder was selected).

This is the entire custom codebase for Phase 1: a branch, a formatter, and one action. Everything else is existing software doing what it already does.

---

## Phase 1 — UX Shell (current phase)

```
rofi (syscall)
   │  passes: current query OR selected line
   ▼
glue script (build)
   │
   ├─ branch: is this a query or a selection? (build)
   │
   ├─ if query → call plocate (syscall) → format output (build) → return to rofi
   │
   └─ if selection → build + fire terminal-open command (build)
```

**Contract this phase establishes**, which future phases must respect: the glue script's search step takes a query string in and returns one matching path per line. Nothing above this seam (rofi, the branch, the terminal-open logic) needs to know or care what's actually doing the searching underneath it.

---

## Phase 2 — Hand-built search engine (separate, later project)

Goal: replace `plocate` behind the same seam with a self-built indexer and matcher, using `find` only for raw filesystem traversal (the one unavoidable syscall) and writing everything else — index format, matching logic, live-filtering loop — by hand in Bash.

Build order:
1. Static indexer — `find` walks the disk once, writes paths to a flat log (build, using `find` as the only syscall)
2. Static substring search — plain matching against the log (build)
3. Fuzzy scoring — hand-rolled subsequence/fuzzy matcher, replacing substring matching (build)
4. Live-typing loop — raw keypress capture and redraw, effectively a hand-built TUI (build)
5. Incremental re-indexing — re-walk only changed directories instead of a full rebuild (build)

This phase exists purely for the algorithmic and systems-level learning — it deliberately has no shipping pressure, since Phase 1 already provides a working daily tool.

---

## Phase 3 — The swap

Once Phase 2's search step matches Phase 1's contract (query in, one path per line out), point the glue script at it instead of `plocate`. If the seam held, nothing else in Phase 1 needs to change. This is also the point where the hand-built engine can be benchmarked directly against `plocate`, using the same rofi front end as the test harness for both.
