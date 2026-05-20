# Architecture Context Pack

## 1. System Architecture Overview

The Snake Game is a multi-implementation codebase that delivers the same gameplay across four independent runtime environments. Each implementation is a self-contained, single-file program.

### Implementations (Findings)

| Version | Entry File | UI Framework | Loop Mechanism |
|---|---|---|---|
| Pygame | `snake_game.py` | `pygame` | While-true loop, `clock.tick(60)` |
| Tkinter | `snake_game_tkinter.py` | `tkinter` | Recursive `root.after(GAME_SPEED=150ms)` |
| Console | `snake_game_console.py` | stdout + ANSI escapes | While-not-game-over loop, blocking `input()` |
| HTML/JS | `snake_game.html` | HTML5 Canvas + vanilla JS | `setInterval(update, GAME_SPEED=150)` |

### Design Patterns Observed

- Standalone-per-file pattern. Each frontend duplicates game-state logic instead of sharing a runtime engine. Only the Pygame version imports the shared module `game_logic.py`; Tkinter, Console, and HTML duplicate the logic inline.
- Class-encapsulated state in Pygame (`Snake`, `Food` classes in `snake_game.py`) and Tkinter (`SnakeGame` class in `snake_game_tkinter.py`).
- Procedural state-as-locals in Console (`snake_game_console.py main()`) and HTML/JS (module-level `let` variables in `<script>`).
- Immediate-mode rendering: each frame redraws the entire screen. Pygame uses `pygame.display.flip()`; Tkinter uses `canvas.delete('all')` then `create_rectangle/...`; Console uses `os.system('cls' or 'clear')` then prints; Canvas clears with `ctx.fillRect`.
- Module-level UPPERCASE constants for all tunables (grid size, colors, intervals, durations).
- Boolean flag-based state machine for storm phases (`storm_active`, `storm_phase in {'warning','explosion'}`).
- Schema-versioned JSON envelope with legacy auto-migration in `leaderboard.py` (`SCHEMA_VERSION = 2`).

## 2. Key Components and Their Relationships

```
+---------------------------------------------------+
|              Frontend Implementations             |
|                                                   |
|  snake_game.py    snake_game_tkinter.py           |
|  (Pygame)         (Tkinter)                       |
|       |                |                          |
|       v                v                          |
|  game_logic.py    HighScoreManager <-- leaderboard.py
|  (shared logic)        |                          |
|                        v                          |
|  snake_game_console.py (procedural)               |
|  snake_game.html (browser, no Python imports)     |
|                                                   |
+--------------------+------------------------------+
                     |
                     v
              highscores.json
              (v2 envelope persisted on disk)
```

### Components (from code)

- `game_logic.py` — Shared pure-logic module. Exports grid constants, direction vectors, `SNAKE_COLORS`, and functions: `check_wall_collision`, `check_self_collision`, `get_next_head`, `calculate_new_score`, `is_valid_food_position`, `is_valid_mine_position`, `spawn_mine`. Imported only by `snake_game.py` and tests.
- `leaderboard.py` — `HighScoreManager` class. Methods: `load_scores`, `save_scores`, `validate_name`, `is_high_score`, `add_score`, `get_leaderboard`, `migrate_legacy_entries`. Used only by `snake_game_tkinter.py`.
- `snake_game.py` — Pygame frontend; defines `Snake`, `Food` classes plus storm/super-food/invincibility state managed as locals in `main()`.
- `snake_game_tkinter.py` — Tkinter frontend; defines `SnakeGame` class and module-level helpers `format_leaderboard_message`, `_format_row`, `_show_leaderboard_window`.
- `snake_game_console.py` — Console frontend; defines `Snake`, `Food` classes plus its own `spawn_mine` and `draw_game` functions (does not import `game_logic.py`).
- `snake_game.html` — Browser frontend; entire game inline in `<script>` block with `init()`, `update()`, `draw()`, `endGame()`, plus storm/invincibility helpers.

### Cross-component coupling

- Only `snake_game.py` depends on `game_logic.py`. The other three frontends are decoupled.
- Only `snake_game_tkinter.py` depends on `leaderboard.py` and persists to `highscores.json`. The other three frontends have no high-score persistence.
- Tests under `tests/` only test `game_logic.py`. Root tests `test_food_color.py`, `test_leaderboard.py`, `test_snake_game_tkinter_display.py` test individual modules.

## 3. Data Flow and System Boundaries

### Process boundary

- Each frontend is a single OS process. Pygame, Tkinter, and Console launch via `python` / `py`. HTML/JS runs in the browser process.
- The Pygame version attempts to load a music file from an absolute path `C:\Users\GrahamSaunders\Downloads\snake_game_v1.1.0\music\chiptune_triumphant.wav` (`INVINCIBILITY_MUSIC_PATH` in `snake_game.py` and `snake_game_tkinter.py`). The HTML version uses a relative path `music/chiptune_triumphant.wav`.
- No network or IPC boundary exists in the gameplay code. The Auto-PR-Solution describes an external MCP server (`aafm-server.js`) that posts to `api.github.com` for PR creation — this is tooling, not part of the running game.

### Data flow (per frame)

1. Input event (keypress / `input()` line) updates `direction` / `next_direction`.
2. `move_snake` / `Snake.move` / `update` computes `new_head` / `next_head`.
3. Mine / bonus / super-food / wall / self / storm-blast collisions are evaluated.
4. Score and snake length are mutated according to collision outcome.
5. `draw_*` functions render the frame.
6. Frame is presented (`pygame.display.flip()`, `canvas.create_*` already on-screen, `print()`, or Canvas `fillRect`).
7. Loop ticks: 60 Hz (Pygame), 150 ms (Tkinter, HTML), `time.sleep(0.1)` + blocking input (Console).

### Storage boundary

- `highscores.json` is the only persistent data store. Read by `HighScoreManager.load_scores` on demand (open-read-close); written by `HighScoreManager.save_scores` on score insertion (`open(..., 'w')`, `json.dump`). No locking is performed.

## 4. Technology Stack and Dependencies

### Runtime stack (from `requirements.txt`, source imports)

- Python: 3.7+ stated in `STRUCTURE.md` line 91 (`Python 3.7+`). `pytest.bat` uses Python 3.14 (`Python314\Scripts\pytest.exe`).
- `pygame==2.5.2` — declared in `requirements.txt`; imported by `snake_game.py`.
- `pytest` — declared in `requirements.txt`.
- `tkinter` — standard library; imported by `snake_game_tkinter.py`.
- `winsound` — Windows-only stdlib; imported by `snake_game_tkinter.py`.
- `json`, `os`, `random`, `sys`, `time`, `math`, `datetime` — standard library.
- HTML version: no dependencies; vanilla JS, HTML5 Canvas, CSS.

### Testing stack

- `pytest` configured via `pytest.ini` (`testpaths = tests`).
- `unittest` used by root-level tests (`test_food_color.py`, `test_leaderboard.py`, `test_snake_game_tkinter_display.py`).
- Fixtures in `tests/conftest.py`: `default_snake_body`, `empty_mines`.

### Platform notes

- `winsound` and the `C:\...` music path tie the Tkinter and Pygame versions to Windows.
- `os.system('cls' if os.name == 'nt' else 'clear')` in `snake_game_console.py` is the only cross-platform branch in the console code.
- `pytest.bat` hard-codes a Windows-specific Python install path.

## 5. Scalability and Performance Considerations

### Frame loop characteristics

- Pygame target frame rate: 60 fps via `clock.tick(60)`; movement is gated by `animation_frames = 5` in `Snake`, yielding ~12 snake-cell moves per second.
- Tkinter and HTML use 150 ms per logical step (`GAME_SPEED = 150`).
- Console step is bounded by blocking `input()` plus `time.sleep(0.1)`; effectively turn-based.

### Computational hotspots

- `draw_gradient_background` (`snake_game.py`) iterates `range(GRID_PIXEL_HEIGHT)` = 600 lines per frame and calls `pygame.draw.line` on each — measurable cost.
- `draw_food_particles` (`snake_game.py`) iterates a global list `food_particles` per frame; each food eaten appends 10 particles with 30-frame TTL.
- Mine spawn `is_valid_mine_position` checks Manhattan distance against every snake segment; bounded by `for attempts in range(1000)` per mine.
- HTML `update()` uses `findIndex`/`some` over `mines`, `bonusFoods`, and `snake` arrays — O(n) per check.

### Scaling limits (from code)

- Grid is fixed: `GRID_WIDTH * GRID_HEIGHT = 30 * 30 = 900` cells (`game_logic.py` derives 30x30 from `WINDOW_WIDTH=600 / GRID_SIZE=20`). Console uses fixed `GRID_WIDTH = GRID_HEIGHT = 20`.
- Mine count grows as `1 + floor(score / 5)` (`spawn_mine` in `game_logic.py` and inlined in each frontend) — bounded by grid size.
- Storm trigger: `STORM_TRIGGER_COUNT = 10` mines.
- Leaderboard truncated to top 10 (`scores = scores[:10]` in `add_score`).

### Concurrency

- Single-threaded throughout. Tkinter timers (`root.after`) and HTML timers (`setInterval`, `setTimeout`) are cooperative within the event loop.
- No locks on `highscores.json` writes; a concurrent run could clobber writes (no concurrent run is anticipated by the code).

## 6. Integration Patterns and External Services

### In-product integrations (from source)

- File I/O: `HighScoreManager` reads/writes `highscores.json` with `open` + `json.dump`/`json.load`. Errors are caught (`json.JSONDecodeError`, `IOError`, `OSError`) and degraded gracefully (return `[]`, return `False`).
- Audio: Pygame uses `pygame.mixer.music.load` and `.play()`. Tkinter uses `winsound.PlaySound(..., SND_FILENAME | SND_ASYNC | SND_NODEFAULT)`. HTML uses `new Audio(...).play()`. All wrapped in `try/except` (Python) or `.catch` (JS).
- No HTTP clients, sockets, databases, or message queues are imported by the game code.

### External tooling (from `Auto-PR-Solution.md`)

- `mcp--Agentic_AI_Feature_Manager--create_pr` calls GitHub REST API (`api.github.com/repos/{owner}/{repo}/pulls`) via the external MCP server `aafm-server.js` (located at `C:\Users\GrahamSaunders\Downloads\aafm-mcp\nodejs\`).
- Config file `.aafm-config.md` (not present in repo) carries `github_log_repo` and `github_log_filename`. Authentication via `AAFM_GITHUB_TOKEN` env var.
- Documented failure modes: 401 (missing PAT), 422 (wrong head repo), 404 (regex bug `[^/.]+` stripping dotted repo names).

### Source-derived constraints (not assumptions)

- Pygame `INVINCIBILITY_MUSIC_PATH` and Tkinter `INVINCIBILITY_MUSIC_PATH` are absolute Windows paths. If the file is missing, both fail softly: Pygame prints to stderr; Tkinter's `winsound.PlaySound` is wrapped in `try/except Exception`.
- HTML `INVINCIBILITY_MUSIC_PATH = 'music/chiptune_triumphant.wav'` is relative to the served document; failures are logged via `console.warn`.

## 7. Architectural Conformance Matrix (Observed)

| Concern | Pygame | Tkinter | Console | HTML/JS |
|---|---|---|---|---|
| Imports `game_logic.py` | Yes | No | No | N/A |
| High-score persistence | No | Yes (`highscores.json`) | No | No |
| Audio | `pygame.mixer.music` | `winsound` | None | `Audio` API |
| Flashing mines | Yes (accumulator) | Yes (`after`) | Static `M` | Yes (`setInterval`) |
| Storm border flash | Yes | Yes | No | Yes |
| Game speed | 60 fps / 5-frame anim | 150 ms | input-blocking | 150 ms |
| Wrapping movement | No | No | Yes (`% GRID_WIDTH`) | No |
| Particle effects | Yes (`food_particles`) | No | No | No |
| Rules screen | Yes | Yes | Yes | Yes |
| Pause | Yes (overlay drawn) | Yes (`toggle_pause`) | Yes (`p` key) | Yes (`p`/`P` key) |



