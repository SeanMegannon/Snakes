# Structure Context Pack

## 1. Project Organization and Folder Structure (observed)

```
c:\BA-Marlin-Test-Repo\
|-- game_logic.py                       # Shared logic (used only by snake_game.py + tests)
|-- snake_game.py                       # Pygame frontend
|-- snake_game.py.bak                   # Historical snapshot
|-- snake_game.py.bak-mds               # Historical snapshot pre Mine Detonation Storm
|-- snake_game.py.bak-pre-white-food    # Historical snapshot pre white-food change
|-- snake_game_tkinter.py               # Tkinter frontend
|-- snake_game_tkinter.py.bak           # Historical snapshot
|-- snake_game_tkinter.py.bak-mds       # Historical snapshot
|-- snake_game_tkinter.py.bak-pre-white-food
|-- snake_game_console.py               # Console frontend
|-- snake_game_console.py.bak           # Historical snapshot
|-- snake_game_console.py.bak-mds       # Historical snapshot
|-- snake_game.html                     # Browser frontend (HTML+CSS+JS)
|-- snake_game.html.bak-mds             # Historical snapshot
|-- leaderboard.py                      # HighScoreManager class
|-- highscores.json                     # Persistent leaderboard (schema v2)
|-- requirements.txt                    # pygame==2.5.2, pytest
|-- pytest.ini                          # [pytest] testpaths = tests
|-- pytest.bat                          # Wrapper invoking Python314 pytest.exe
|-- run_game.bat                        # py snake_game_tkinter.py
|-- run_pygame_version.bat              # py snake_game.py
|-- run_web_game.bat                    # start snake_game.html
|-- README.md                           # User-facing readme
|-- STRUCTURE.md                        # Project-structure doc (describes intended .cursor/ and .vscode/ — not present)
|-- Auto-PR-Solution.md                 # Post-mortem of GitHub PR-creation failures
|-- test_food_color.py                  # unittest, parses snake_game.py source
|-- test_leaderboard.py                 # unittest, isolated via tempfile
|-- test_snake_game_tkinter_display.py  # unittest with mocks
|-- verify_color.py                     # Standalone color verification script
|-- tests\
|   |-- conftest.py                     # pytest fixtures
|   |-- test_game_logic.py              # pytest unit tests for game_logic
|   `-- test_smoke.py                   # pytest smoke tests
|-- flashing-mines\
|   |-- Build.md
|   |-- flashing-mines-plan.md
|   |-- post-implementation-lessons-learned.md
|   `-- ToDo.md
|-- mine-detonation-storm\
|   |-- Build.md
|   |-- feature.json
|   |-- Plan.md
|   |-- run-log.md
|   |-- ToDo.md
|   `-- user-story.md
|-- .pytest_cache\
|   |-- CACHEDIR.TAG
|   |-- README.md
|   `-- v\cache\{lastfailed, nodeids}
`-- docs\
    `-- .cursorrules
```

Notes (from files):
- `STRUCTURE.md` describes `.cursor/`, `.vscode/`, `.gitignore`, and a root `.cursorrules` — none are present in the repo.
- Three pairs of `.bak*` files are historical snapshots of the four frontends.

## 2. Module Dependencies and Relationships

### Import edges (verified by reading source)

- `snake_game.py` -> `pygame`, `random`, `sys`, `math`, `game_logic`
- `snake_game_tkinter.py` -> `tkinter`, `tkinter.simpledialog`, `tkinter.messagebox`, `random`, `winsound`, `sys`, `leaderboard`
- `snake_game_console.py` -> `random`, `os`, `time`  (does NOT import `game_logic`)
- `snake_game.html` -> none (browser-only)
- `leaderboard.py` -> `json`, `os`, `datetime`
- `game_logic.py` -> `random`
- `tests/test_game_logic.py` -> `pytest`, `game_logic`
- `tests/test_smoke.py` -> `game_logic`
- `tests/conftest.py` -> `pytest`
- `test_food_color.py` -> `unittest`, `os`, `re` (reads `snake_game.py` as text)
- `test_leaderboard.py` -> `unittest`, `tempfile`, `os`, `json`, `leaderboard`
- `test_snake_game_tkinter_display.py` -> `unittest`, `unittest.mock`, `snake_game_tkinter` (mocks Tkinter)
- `verify_color.py` -> `re` (reads `snake_game.py` as text)

### No-cycle observation

The dependency graph is a DAG: frontends depend on `game_logic` and `leaderboard`; tests depend on those modules.

### Dead / orphan code

- `LIGHT_BLUE_SHINE`, `LIGHT_YELLOW`, `WHITE`, `DARK_BLUE` color constants in `snake_game.py` are partly unused (`LIGHT_YELLOW` annotated as unused in source).
- `SUPER_FOOD_COLOR_A` is referenced in `draw_super_food` in `snake_game.py` but not defined at module level (only `SUPER_FOOD_COLOR_B` is defined). This is a latent NameError in the Pygame path.

## 3. Code Organization Patterns

- One file per frontend; no engine layer.
- Constants at top of each file (grid, colors, intervals, durations).
- Class `Snake` and `Food` in Pygame and Console; class `SnakeGame` in Tkinter; module-level state in HTML.
- Game flow: setup -> rules screen -> main loop -> game over -> (Tkinter only) leaderboard prompt + display.
- Storm state always uses the same variable names across versions: `storm_active`, `storm_queue`, `storm_phase`, `storm_current_mine` (and JS camelCase equivalents).
- Tkinter uses `hasattr(self, '_xxx_after_id')` guards before `root.after_cancel(...)` for timer hygiene.

## 4. Entry Points and Main Flows

| Entry | Command | Effect |
|---|---|---|
| `python snake_game.py` | direct | `main()` invoked under `__name__ == '__main__'` |
| `py snake_game_tkinter.py` | direct | `main()` creates `tk.Tk()`, instantiates `SnakeGame`, `root.mainloop()` |
| `python snake_game_console.py` | direct | `main()` invoked; relies on terminal stdin |
| `snake_game.html` | open in browser | `init()` called at end of `<script>` |
| `run_game.bat` | Windows batch | `py snake_game_tkinter.py` + `pause` |
| `run_pygame_version.bat` | Windows batch | `py snake_game.py` + `pause` |
| `run_web_game.bat` | Windows batch | `start snake_game.html` |
| `pytest.bat` | Windows batch | Invokes `C:\Users\GrahamSaunders\AppData\Local\Programs\Python\Python314\Scripts\pytest.exe` |

### Initialisation order (per frontend)

- Pygame: `pygame.init()` at import, then `main()` constructs `screen`, `clock`, `Snake`, `Food`, then enters loop. `pygame.quit()` is also called after `main()` at module bottom (line 572).
- Tkinter: `SnakeGame.__init__` builds canvas, score label, inv label, `HighScoreManager()`, calls `reset_game()`, binds keys, starts `game_loop()`.
- Console: prints rules screen, awaits Enter, then enters `while not game_over` loop.
- HTML: `init()` resets state, optionally draws rules screen (single keydown listener to dismiss), then `setInterval(update, 150)`.

## 5. Configuration and Environment Setup

### Runtime config (from `requirements.txt`)

```
pygame==2.5.2
pytest
```

### Pytest config (`pytest.ini`)

```
[pytest]
testpaths = tests
```

### Environment dependencies discovered in source

- Pygame `INVINCIBILITY_MUSIC_PATH` and Tkinter `INVINCIBILITY_MUSIC_PATH` (both hard-coded to a Windows user-specific path).
- HTML expects `music/chiptune_triumphant.wav` to be served alongside `snake_game.html`.
- `pytest.bat` hard-codes Python 3.14 install path.
- `winsound` is Windows-only — Tkinter frontend will fail to import on non-Windows.

### Configuration files NOT present (described elsewhere)

- `.cursor/`, `.vscode/`, `.gitignore`, root-level `.cursorrules` are described in `STRUCTURE.md` but are not present in the repo's file listing.
- `.aafm-config.md` is referenced by `Auto-PR-Solution.md` but is not present.

## 6. Build and Deployment Structure

- No build system. Python frontends are run directly with the interpreter. HTML frontend is loaded directly into a browser.
- No CI configuration present (no `.github/workflows/`, no `Makefile`, no `Dockerfile`).
- Distribution mechanism (from `README.md`): copy files locally; install pygame via `pip install -r requirements.txt` if running the Pygame version.
- Persistent state lives only in `highscores.json` next to the Tkinter executable's working directory (`HighScoreManager(filepath='highscores.json')`).
