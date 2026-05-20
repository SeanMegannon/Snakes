# Decisions Context Pack

This file records architectural decisions strictly evidenced by repo content (source code or feature-folder documents). Decisions are sourced from `flashing-mines/`, `mine-detonation-storm/`, `Auto-PR-Solution.md`, and direct code observations.

## 1. Architectural Decision Records (ADRs)

### ADR-001 — Standalone per-frontend implementation

- **Status:** Accepted (in effect).
- **Context:** Four UI environments are supported (Pygame, Tkinter, Console, HTML).
- **Decision:** Each frontend is a single self-contained file. Only `snake_game.py` imports the shared `game_logic.py`; the other three duplicate logic.
- **Source:** Verified by import scans; cited as `ADR-007` in `flashing-mines/flashing-mines-plan.md` TR-1, and re-affirmed in `mine-detonation-storm/Plan.md` 2.1 (`Standalone per-file | ADR-007`).
- **Consequence:** Same algorithm (e.g. `spawn_mine`) is implemented in three places; the console version diverges (modulo wrapping).

### ADR-002 — Mine state as plain list

- **Source:** `mine-detonation-storm/Plan.md` 2.1 references `Plain list for mine state | ADR-FM-002`.
- **Decision:** Mines are stored as `list[(x,y)]` (Python) / `Array<{x,y}>` (JS). No dedicated `Mine` class.
- **Consequence:** Iteration and membership tests are O(n); spawn validation cost grows linearly with mine count.

### ADR-003 — Per-platform timing strategy

- **Source:** `mine-detonation-storm/Plan.md` 2.1 references `Accumulator timing (Pygame) | ADR-FM-001`, `root.after() scheduling (Tkinter) | ADR-FM-001`, `setInterval/setTimeout (HTML) | ADR-FM-001`, `Tick-based timing (Console) | ADR-FM-001, ADR-FM-003`.
- **Decision:** Each platform uses its native timing primitive. No shared scheduler.
- **Consequence:** Console cannot flash (`G-1` in `flashing-mines-plan.md`); accepted via user story AC-6 — render static `M` instead.

### ADR-004 — Pygame collision against `next_head`

- **Source:** `mine-detonation-storm/Plan.md` 2.1 references `Pygame collision uses next_head | ADR-FM-004 / IMP-FM-D1`; `flashing-mines-plan.md` G-2.
- **Decision:** Because Pygame uses 5-frame interpolation, mine/wall/self collisions must compare against `snake.next_head`, not `snake.body[0]`.
- **Consequence:** Other frontends compute `new_head` and check it directly before insertion.

### ADR-005 — HTML collision-before-unshift order

- **Source:** `mine-detonation-storm/Plan.md` 2.1 references `HTML collision before unshift | IMP-FM-D4`.
- **Decision:** In `update()`, the explosion blast check and mine collision are evaluated before `snake.unshift(head)`, consistent with the wall/self collision evaluation order.

### ADR-006 — Console mine path projection with modulo + visited set

- **Source:** `flashing-mines-plan.md` G-3, `mine-detonation-storm/Plan.md` 2.1 (`Console path projection visited guard | IMP-FM-D2 / C1`).
- **Decision:** Because the console wraps at edges, mine-spawn path projection iterates `(px + dx) % GRID_WIDTH, (py + dy) % GRID_HEIGHT` with a `visited` set to terminate.
- **Consequence:** Path covers a finite cycle rather than running off-grid.

### ADR-007 — Schema-versioned JSON envelope with auto-migration

- **Source:** `leaderboard.py` (`SCHEMA_VERSION = 2`; `_unwrap_scores`; `load_scores` migrates a bare list and persists v2 on first load).
- **Decision:** Persisted leaderboard is a dict `{schema_version: 2, scores: [...]}`. Legacy bare-list files are migrated transparently.
- **Consequence:** `test_leaderboard.py` exercises both code paths; `highscores.json` shows a mix of legacy-shape (`food_eaten` first) and v2-shape (`score` first) entries that successfully load.

### ADR-008 — Manual-only testing

- **Source:** `flashing-mines-plan.md` (G-4: "Testing is manual only, consistent with rules.mdc Rule 10."), `mine-detonation-storm/Plan.md` test-strategy section.
- **Decision:** New gameplay features are validated by manual test procedures recorded in feature folders. Automated tests exist only for `game_logic.py` and `leaderboard.py`.
- **Consequence:** The four frontends have no end-to-end automated coverage.

### ADR-009 — Always-on visual rules screen

- **Source:** Each of Pygame, Tkinter, Console, HTML has a rules screen displayed before play (`draw_rules_screen`, `_draw_rules`, `show_rules_screen`, `drawRulesScreen`).
- **Decision:** Players must dismiss the rules screen (any key/click, or Enter in console) before play begins.

### ADR-010 — Hot-pink food in Tkinter contrasts other versions

- **Source:** `snake_game_tkinter.py:15 FOOD_COLOR = '#FF69B4'  # Hot Pink (RGB 255, 105, 180)` and inline comment in `draw` on line 399 still says "ALWAYS YELLOW per .cursorrules" — but the literal is hot pink.
- **Decision (observed):** Food color is hot pink in Tkinter, white in Pygame (`FOOD_COLOR = (255, 255, 255)`) and HTML (`const FOOD_COLOR = '#FFFFFF'`).
- **Note:** This is an inconsistency between code and inline comments rather than a clearly-recorded architectural decision; the README says "Apples/Food: Yellow (always)", which contradicts current source.

### ADR-011 — Repository URL parsing must allow dots

- **Source:** `Auto-PR-Solution.md` (404 error fix).
- **Decision:** External MCP server regex `/github\.com[/:]([^/]+)\/([^/]+)/` (dot allowed) supersedes the buggy `[^/.]+` that truncated dotted repo names.
- **Decision:** `.aafm-config.md` must point `github_log_repo` at the target repository (not the log repository).
- **Consequence:** Documented in `Auto-PR-Solution.md` "Complete Solution" section.

## 2. Technology Choice Rationale (as documented)

- Python with Tkinter chosen for zero-dependency desktop GUI (`README.md`: "Uses Python's built-in tkinter library", "No extra dependencies needed!").
- Pygame chosen for "Advanced graphics and animations" (`README.md`).
- HTML/Canvas chosen as the easiest deployment (`README.md`: "Just open the file and play!").
- Console chosen as a minimal fallback that requires only Python standard library.
- `winsound` chosen for audio in the Tkinter version (Windows-only); `pygame.mixer.music` in Pygame; HTML `Audio` API in browser. No cross-platform audio abstraction.
- JSON chosen for high scores: human-readable, no external DB dependency.

## 3. Design Pattern Selections

| Pattern | Choice | Source |
|---|---|---|
| State management | Locals in `main()` (Pygame, Console), instance attrs (Tkinter), module-level `let` (HTML) | Source files |
| Timing | Accumulator (Pygame), `root.after` (Tkinter), `setInterval/Timeout` (HTML), ticks (Console) | ADR-003 |
| Rendering | Immediate mode (full redraw per frame) | All frontends |
| Persistence | JSON file with versioned envelope | ADR-007 |
| Mine state | Plain list | ADR-002 |
| Storm state machine | Boolean + `'warning'/'explosion'` string phases | `Plan.md` 2.4 |
| Migration | Idempotent dict merge `{**defaults, **entry}` | `leaderboard.py` |

## 4. Trade-offs and Compromises Made

- **Duplication vs. shared engine:** Chose duplication (ADR-001). Three of four versions reimplement the same algorithms. Trade-off: simpler per-file reasoning at the cost of fan-out maintenance.
- **Static `M` vs. real-time flash in console:** Accepted the limitation (`flashing-mines-plan.md` G-1, user story AC-6) because `input()` blocks the thread.
- **Manual testing only:** Accepted (ADR-008). Result: source-code regressions in the console frontend (indentation/structure issues at lines 290-291) and in the Pygame frontend (`SUPER_FOOD_COLOR_A` undefined; nested mine event logic) were not caught by tests.
- **Re-read of `highscores.json` on every `is_high_score`/`add_score`/`get_leaderboard`:** Trade-off: simpler code at the cost of repeated disk I/O.
- **Silent skip on mine spawn exhaustion:** Trade-off: predictable gameplay continuity at the cost of detectable spawn failures (`spawn_mine` returns without raising).

## 5. Alternative Approaches Considered (where documented)

- `flashing-mines/post-implementation-lessons-learned.md` (read earlier) records considered alternatives:
  - Sharing a flashing scheduler — rejected because each frontend already has its own timing primitive.
  - Implementing console flashing via threading — rejected to keep console blocking `input()` semantics simple.
- `Auto-PR-Solution.md` considered three regex variants for owner/repo extraction and selected `([^/]+)\/([^/]+)` to admit dotted repository names.

## 6. Future Considerations and Technical Debt (documented in source)

- `LIGHT_YELLOW = (255, 255, 200)  # For super food shine effect (unused by regular food)` — annotated unused.
- `SUPER_FOOD_COLOR_A` is referenced in `snake_game.py:551` (`draw_super_food`) but not defined — latent `NameError` when super-food rendering executes.
- `snake_game_console.py` lines 290-291: the inner statement `storm_phase_ticks -= 1` is not properly indented under `if not paused and storm_active:`. This is in the file as read and would prevent the module from importing/running as-is.
- `snake_game.py` lines 271-455: per-event execution of game logic (movement, collision checks) inside `for event in pygame.event.get():` rather than once per frame is an observed structural quirk.
- `pytest.bat` hard-codes a specific user's Python install path (`C:\Users\GrahamSaunders\AppData\Local\Programs\Python\Python314\Scripts\pytest.exe`).
- `STRUCTURE.md` references `.cursor/`, `.vscode/`, `.gitignore`, and a root `.cursorrules` that are not present in the repository.
