# Risk Context Pack

All risks below are derived from observed code or documented incidents in the repository. No assumptions are added.

## 1. Security Vulnerabilities and Concerns

### S-1 — Secrets / credentials in source-control-adjacent docs
- `Auto-PR-Solution.md` describes `AAFM_GITHUB_TOKEN` env var living in `C:\Users\GrahamSaunders\Downloads\aafm-mcp\nodejs\.env`. The token itself is not in the repo, but the documentation pinpoints its filesystem location and the required PAT scope (`repo` or fine-grained "Pull requests Read and Write").
- Mitigation: keep `.env` out of any future repo commits; rotate token if it was ever pasted into chat history.

### S-2 — Hard-coded absolute Windows paths to user-specific resources
- `INVINCIBILITY_MUSIC_PATH = r"C:\Users\GrahamSaunders\Downloads\snake_game_v1.1.0\music\chiptune_triumphant.wav"` in `snake_game.py:53` and `snake_game_tkinter.py:56`.
- `pytest.bat` hard-codes `C:\Users\GrahamSaunders\AppData\Local\Programs\Python\Python314\Scripts\pytest.exe`.
- These leak a specific Windows username and break portability. Not exploitable directly but should not be committed.

### S-3 — Unvalidated file read
- `HighScoreManager.load_scores` opens `highscores.json` with no path-traversal protection. Caller controls `filepath` via constructor. Trust boundary is the caller — currently only `snake_game_tkinter.py` calls it with the default `'highscores.json'`.

### S-4 — Test files read source code by file path
- `test_food_color.py` opens `snake_game.py` using `os.path.join(os.path.dirname(__file__), 'snake_game.py')` and `verify_color.py` opens `'snake_game.py'` with a relative path. These rely on a specific working directory.

### S-5 — Audio playback exception swallow
- Pygame `_stop_invincibility_music` and Tkinter `winsound.PlaySound(None, winsound.SND_PURGE)` use bare `except Exception: pass`. This makes audio failures silent.

### S-6 — No transactional write of `highscores.json`
- `save_scores` opens the file with `'w'` and `json.dump`s in place. A crash mid-write would truncate the file. The next `load_scores` would then return `[]` via the `JSONDecodeError` branch — silent data loss.

## 2. Performance Bottlenecks and Risks

### P-1 — `draw_gradient_background` redraws 600 lines per frame
- `snake_game.py:510-516` iterates `range(GRID_PIXEL_HEIGHT)` (=600) and issues `pygame.draw.line` per scanline at 60 fps.

### P-2 — Repeated JSON re-reads of `highscores.json`
- `HighScoreManager.is_high_score`, `add_score`, and `get_leaderboard` each call `load_scores`, which re-opens and re-parses the file. Tkinter `handle_game_over` triggers two consecutive reads (one inside `is_high_score`, one inside `display_leaderboard`).

### P-3 — Console screen clear per frame
- `os.system('cls' if os.name == 'nt' else 'clear')` in `draw_game` is expensive and flickers in many terminals.

### P-4 — Mine spawn worst case O(1000 * len(snake_body))
- `is_valid_mine_position` is checked up to 1000 times per missing mine; each check does an O(len(body)) Manhattan scan and an O(grid) path projection.

### P-5 — Particle list growth
- `food_particles` is a global list in `snake_game.py`. Each food spawn appends 10 entries; each lives 30 frames. Sustained eating creates allocation churn.

### P-6 — Per-event game logic in Pygame
- The body of the `for event in pygame.event.get():` loop in `snake_game.py` (lines 271-455) contains `snake.move()`, mine collision checks, storm logic, and food consumption. When multiple events arrive in a single frame, gameplay updates multiple times.

## 3. Technical Debt and Maintenance Issues

### D-1 — Three implementations of the same logic
- `spawn_mine` exists in `game_logic.py`, inlined in `snake_game_tkinter.py._try_spawn_mine`, inlined in `snake_game_console.py.spawn_mine`, and inlined in `snake_game.html.trySpawnMine`. Behavior diverges (console uses modulo wrap).

### D-2 — Console source is broken as-is
- `snake_game_console.py` lines 290-291: `if not paused and storm_active:` is followed by `storm_phase_ticks -= 1` at the wrong indentation. As read, this would raise `IndentationError` at import.

### D-3 — Pygame references undefined constant
- `snake_game.py:551` `draw_super_food` uses `SUPER_FOOD_COLOR_A`, which is not defined at module scope (only `SUPER_FOOD_COLOR_B` is defined on line 49). Triggers `NameError` when super-food appears.

### D-4 — README ↔ source colour inconsistency
- `README.md` line 67: "**Apples/Food:** Yellow (always)". Source: Pygame and HTML use white (`(255,255,255)` / `'#FFFFFF'`), Tkinter uses hot pink (`'#FF69B4'`), Console uses bright white (`'\033[97m'`). Comments in Tkinter `draw()` still say "ALWAYS YELLOW per .cursorrules".

### D-5 — `STRUCTURE.md` references non-existent files
- `STRUCTURE.md` documents `.cursor/`, `.vscode/`, `.gitignore`, and a root `.cursorrules` — none are present in the file listing.

### D-6 — Three sets of `.bak*` files in repo root
- `snake_game.py.bak`, `.bak-mds`, `.bak-pre-white-food`; same for `snake_game_tkinter.py`; partial set for console and html. These are not under any retention policy in the repo.

### D-7 — Tests not portable
- `pytest.bat` is Windows-only and uses an absolute path. `verify_color.py` uses a relative path.
- `test_snake_game_tkinter_display.py` mocks Tkinter to run on CI but is at the repo root, not under `tests/` (and so excluded by `pytest.ini`'s `testpaths = tests`).

### D-8 — Music path is per-user
- See S-2. Outside Graham Saunders' machine, the file is missing and the catch blocks silence it.

### D-9 — `food_particles` global mutable state
- `snake_game.py:483 food_particles = []` at module level. `create_food_particles` and `draw_food_particles` mutate it via `global food_particles`. No reset on game restart.

## 4. Scalability Limitations

- Grid is fixed at 30x30 (Pygame/Tkinter/HTML) or 20x20 (Console). Mine count maxes near grid capacity but is effectively bounded by `STORM_TRIGGER_COUNT = 10`.
- Leaderboard caps at 10 entries (`scores[:10]`).
- No multi-player or networked play; single-process, single-window.
- No batch / headless mode.

## 5. Dependencies and External Risks

| Dependency | Source | Risk |
|---|---|---|
| `pygame==2.5.2` | `requirements.txt` | Pinned old release; Python 3.7-3.13 stated in `README.md`. Python 3.14 (used by `pytest.bat`) may lack wheels. |
| `tkinter` | stdlib | Distribution-dependent on some Linux flavours; not packaged with all CPython builds. |
| `winsound` | Windows stdlib | Tkinter version unimportable on non-Windows. |
| `pytest` | unpinned | No version pin — future breaking changes possible. |
| Browser HTML5 Audio | implicit | Autoplay policy may block `Audio.play()` until user gesture. |
| GitHub REST API (external tooling) | `Auto-PR-Solution.md` | Rate limits, auth failures, repo rename — documented in failure timeline. |
| `aafm-server.js` | `Auto-PR-Solution.md` | Lives outside this repo at `C:\Users\GrahamSaunders\Downloads\aafm-mcp\nodejs\`; not version-controlled here. |

## 6. Recommended Mitigation Strategies (only those documented or clearly indicated by code)

- **For S-1:** the documented fix in `Auto-PR-Solution.md` Section "Complete Solution" — store `AAFM_GITHUB_TOKEN` outside the repo and never echo it.
- **For S-6:** an atomic write pattern (write temp + rename) is not implemented; documenting this gap.
- **For D-1:** the `flashing-mines-plan.md` and `mine-detonation-storm/Plan.md` explicitly chose duplication via ADR-007. Future consolidation would require deviating from that ADR.
- **For D-2:** the file as committed will not import. The minimum required correction is to indent `storm_phase_ticks -= 1` and the following branch logic under the `if not paused and storm_active:` block.
- **For D-3:** define `SUPER_FOOD_COLOR_A = (255, 255, 0)` in `snake_game.py` (consistent with Tkinter `'#FFFF00'` and HTML `'#FFFF00'`).
- **For D-4:** reconcile `README.md`, `.cursorrules`-style comments, and the actual `FOOD_COLOR` constants across the four frontends. Currently the Tkinter inline comment is the only conflicting site (`# ALWAYS YELLOW per .cursorrules`).
- **For D-5:** either restore the documented IDE configuration folders or remove their description from `STRUCTURE.md`.
- **For P-1:** the gradient could be cached on a `Surface` once at startup; not implemented.
- **For P-2:** an in-memory cache of scores invalidated on save would remove redundant reads; not implemented.

## 7. Documented Incidents and Resolutions

From `Auto-PR-Solution.md`:

| Incident | Root cause | Resolution applied |
|---|---|---|
| GitHub 401 | Missing `AAFM_GITHUB_TOKEN` | User added PAT to MCP server `.env` |
| GitHub 422 on `head` | `github_log_repo` pointed at log repo | Updated `.aafm-config.md` to target repo URL |
| GitHub 404 | Regex `[^/.]+` truncated `snake_game_v1` from `snake_game_v1.1.0` | Changed regex to `[^/]+` in `aafm-server.js` |

These are the only post-incident records present in the repository.
