# Glossary Context Pack

All entries are derived from repository contents. Quoted constants are verbatim from source.

## 1. Domain-Specific Terminology

- **Snake** — Ordered list of `(x, y)` cells representing the player avatar. Head is index 0. Defined in all four frontends.
- **Food** — A single `(x, y)` target the snake must collide with to grow. White circle in Pygame and HTML (`FOOD_COLOR = (255,255,255)` / `'#FFFFFF'`); hot pink in Tkinter (`FOOD_COLOR = '#FF69B4'`).
- **Mine** — A static (Console) or flashing (Pygame, Tkinter, HTML) hazard cell. Collision shrinks the snake by 3 and reduces the score by 3 (floored at 0).
- **Bonus food** — Green food produced when a mine detonates during a storm; appended to `bonus_foods`.
- **Super-food** — A `'#FFFF00' / '#000000'` flashing cell spawned exactly once per storm at the chosen `super_food_mine_index` detonation. Consuming it triggers invincibility.
- **Invincibility** — 10-second state during which collisions with walls, self, mines, and explosion blast cells do not end the game.
- **Detonation storm (a.k.a. Mine Detonation Storm)** — Sequence of warning + explosion phases triggered when `len(mines) >= STORM_TRIGGER_COUNT` (=10).
- **Warning phase** — `'warning'`: lasts `WARNING_DURATION` (3.0 s) / `WARNING_DURATION_MS` (3000 ms) / `WARNING_TICKS` (3 ticks).
- **Explosion phase** — `'explosion'`: lasts `EXPLOSION_DURATION` (1.0 s) / `EXPLOSION_DURATION_MS` (1000 ms) / `EXPLOSION_TICKS` (1 tick). Kill-zone is the 3x3 neighborhood around `storm_current_mine`.
- **Storm queue (`storm_queue`)** — Shuffled list of remaining mine positions to detonate.
- **Storm border** — Animated red/black 2-pixel rectangle around the play area during a storm.
- **Pause** — Suspends game logic; rendered as `'PAUSED'` overlay (Pygame/Tkinter/HTML) or `*** PAUSED ***` text (Console).
- **Rules screen** — Pre-game informational overlay titled `'Test1 Rules'` (verbatim in all four versions).
- **Leaderboard** — Persistent top-10 list of `(score, name, date, food_eaten, mine_shrinks, invincibility_count)` entries.

## 2. Technical Terms and Abbreviations

- **AAFM** — Agentic AI Feature Manager. Referenced in `Auto-PR-Solution.md` (`AAFM_GITHUB_TOKEN`, `.aafm-config.md`, `aafm-server.js`).
- **ADR** — Architectural Decision Record. References to `ADR-007`, `ADR-FM-001` through `ADR-FM-004`, `IMP-FM-D1` through `IMP-FM-D4` in feature folders.
- **PAT** — Personal Access Token (GitHub). Documented in `Auto-PR-Solution.md`.
- **MCP** — Model Context Protocol server. `Auto-PR-Solution.md` references `mcp--Agentic_AI_Feature_Manager--create_pr` and `aafm-server.js`.
- **TR / EH / T / G / IMP / AC / SC** — Trailing numeric codes used in feature plans for Technical Requirement, Error Handling, Test, Gap, Implementation Decision, Acceptance Criterion, and Success Criterion respectively.
- **dt** — Frame delta time in seconds (Pygame: `dt = clock.get_time() / 1000.0`).
- **TTL** — Time To Live in frames; used by particles in `snake_game.py` (`ttl: 30`).
- **HUD** — Heads-Up Display. `HUD_INVINCIBILITY_OFFSET = 5 * GRID_SIZE` controls the inv-countdown placement.
- **ANSI escape code** — Used in `snake_game_console.py` for terminal colors (e.g. `'\033[91m'` = bright red).
- **Manhattan distance** — `|x1 - x2| + |y1 - y2|`; used in mine placement with threshold 10.
- **Schema version** — Integer key persisted alongside scores list; current value `SCHEMA_VERSION = 2`.

## 3. Business / Gameplay Logic Terminology

- **Game over** — Triggered by wall collision, self-collision, mine collision when snake length reaches 0, or being caught in a Phase-2 blast cell, in all cases bypassed by `invincibility_active = True`.
- **High score** — A score qualifies for the leaderboard if `len(scores) < 10 or score > scores[-1]['score']`.
- **Stats counters** — `food_eaten`, `mine_shrinks`, `invincibility_count`. Tracked by Tkinter only and stored on each leaderboard entry. Other frontends do not collect stats.
- **Color cycle** — `color_index = (color_index + 1) % len(SNAKE_COLORS)` is incremented on each food/bonus consumption.
- **Spawn cap formula** — `expected_count = 1 + score // 5`.
- **Spawn retry budget** — `for attempts in range(1000)`.

## 4. API and Interface Definitions

### `HighScoreManager` (public surface)

- `HighScoreManager(filepath='highscores.json')` — constructor.
- `load_scores() -> list[dict]`.
- `save_scores(scores: list[dict]) -> bool`.
- `validate_name(name: str) -> bool`.
- `is_high_score(score: int) -> bool`.
- `add_score(score, name, food_eaten=0, mine_shrinks=0, invincibility_count=0) -> list[dict]`; raises `ValueError` on invalid name.
- `get_leaderboard() -> list[dict]`.
- `migrate_legacy_entries(entries: list[dict]) -> list[dict]`.

### `game_logic` (public surface)

- `check_wall_collision(head, grid_width=GRID_WIDTH, grid_height=GRID_HEIGHT) -> bool`.
- `check_self_collision(head, body) -> bool`.
- `get_next_head(head, direction) -> tuple[int,int]`.
- `calculate_new_score(current_score) -> int`.
- `is_valid_food_position(pos, snake_body) -> bool`.
- `is_valid_mine_position(candidate, snake_body, snake_direction, food_pos, mines) -> bool`.
- `spawn_mine(mines, snake_body, snake_direction, food_pos, score) -> None` (mutates `mines`).

### Module-level helpers in `snake_game_tkinter.py`

- `format_leaderboard_message(scores: list[dict]) -> str` — pure formatter; tolerates legacy entries lacking stats keys.
- `_format_row(rank: int, entry: dict) -> str`.
- `_show_leaderboard_window(parent_root, message: str) -> None`.

### Keyboard bindings (verbatim)

- Pygame: `K_UP`, `K_DOWN`, `K_LEFT`, `K_RIGHT`, `K_SPACE`, `K_ESCAPE`; mouse for rules dismiss.
- Tkinter: `<Up>`, `<Down>`, `<Left>`, `<Right>`, `<space>`, `<Escape>`.
- Console: `w`, `a`, `s`, `d`, `p`, `q`, plus Enter for rules dismiss.
- HTML: `ArrowUp`, `ArrowDown`, `ArrowLeft`, `ArrowRight`, `' '` (Space), `'p'`/`'P'`, `'Escape'`.

## 5. Configuration Options and Parameters

### Grid

| Name | Pygame / shared | Tkinter | Console | HTML |
|---|---|---|---|---|
| `GRID_SIZE` | 20 | 20 | n/a (text) | 20 |
| `WINDOW_WIDTH` | 600 | 600 | n/a | 600 (canvas) |
| `WINDOW_HEIGHT` | 630 | 600 | n/a | 600 |
| `GRID_PIXEL_HEIGHT` | 600 | n/a | n/a | n/a |
| `SCORE_BAR_HEIGHT` | 30 | n/a | n/a | n/a |
| `GRID_WIDTH` | 30 | 30 | 20 | 30 |
| `GRID_HEIGHT` | 30 | 30 | 20 | 30 |

### Timing

| Name | Pygame | Tkinter | Console | HTML |
|---|---|---|---|---|
| Game speed | `clock.tick(60)` | `GAME_SPEED = 150 ms` | `time.sleep(0.1)` | `GAME_SPEED = 150` |
| `MINE_FLASH_INTERVAL` | 0.2 s | 200 ms | n/a | 200 ms |
| `STORM_BORDER_FLASH_INTERVAL` | 0.2 s | 200 ms | n/a | 200 ms |
| `WARNING_FLASH_INTERVAL` | 0.2 s | 200 ms | n/a | 200 ms |
| Warning duration | `WARNING_DURATION = 3.0 s` | `WARNING_DURATION_MS = 3000` | `WARNING_TICKS = 3` | `WARNING_DURATION_MS = 3000` |
| Explosion duration | `EXPLOSION_DURATION = 1.0 s` | `EXPLOSION_DURATION_MS = 1000` | `EXPLOSION_TICKS = 1` | `EXPLOSION_DURATION_MS = 1000` |
| `STORM_TRIGGER_COUNT` | 10 | 10 | 10 | 10 |
| `SUPER_FOOD_FLASH_INTERVAL` | 0.25 s | 250 ms | n/a | 250 ms |
| Invincibility duration | `INVINCIBILITY_DURATION = 10.0 s` | `INVINCIBILITY_DURATION_MS = 10000` | `INVINCIBILITY_TICKS = 100` | `INVINCIBILITY_DURATION_MS = 10000` |

### Colors (verbatim)

- Pygame snake palette `SNAKE_COLORS = [(0,100,255),(0,255,255),(50,255,50),(255,255,0),(255,165,0),(255,50,50),(255,0,255),(255,105,180)]`.
- Tkinter / HTML snake palette: `['#0064FF','#00FFFF','#32FF32','#FFFF00','#FFA500','#FF3232','#FF00FF','#FF69B4']`.
- Console snake palette uses ANSI escapes: `['\033[94m','\033[96m','\033[92m','\033[93m','\033[38;5;208m','\033[91m','\033[95m','\033[38;5;213m']`.
- `MINE_COLOR_A = (255,0,0) / '#FF0000'`; `MINE_COLOR_B = (200,200,200) / '#C8C8C8'`.
- `STORM_BORDER_COLOR_A = '#FF0000'`; `STORM_BORDER_COLOR_B = '#000000'`.
- `EXPLOSION_COLORS = [(255,0,0),(255,136,0),(255,255,0),(255,255,255),(255,68,0)]` / `['#FF0000','#FF8800','#FFFF00','#FFFFFF','#FF4400']`.
- `INVINCIBILITY_COLOR = (255,255,0) / '#FFFF00'`.
- `SUPER_FOOD_COLOR_B = (0,0,0) / '#000000'`. Pygame is missing `SUPER_FOOD_COLOR_A`.

### Console characters

- `MINE_CHAR = 'M'`.
- `CONSOLE_WARNING_CHAR = '!'`.
- `CONSOLE_EXPLOSION_CHAR = '*'`.
- `CONSOLE_SUPER_FOOD_CHAR = '$'`.
- `CONSOLE_STORM_LABEL = '*** DETONATION STORM ***'`.
- `CONSOLE_INVINCIBLE_LABEL = '*** INVINCIBLE ***'`.

### File paths

- `INVINCIBILITY_MUSIC_PATH` (Pygame, Tkinter) = `r"C:\Users\GrahamSaunders\Downloads\snake_game_v1.1.0\music\chiptune_triumphant.wav"`.
- `INVINCIBILITY_MUSIC_PATH` (HTML) = `'music/chiptune_triumphant.wav'`.
- `HighScoreManager(filepath='highscores.json')` default path.

### Leaderboard constants

- `SCHEMA_VERSION = 2`.
- `VALID_NAME_MAX_LENGTH = 20`.
- Truncation: top 10 (`scores[:10]`).

## 6. Error Codes and Status Meanings (from `Auto-PR-Solution.md`)

- **HTTP 401 Unauthorized** — "Bad credentials". Cause: missing `AAFM_GITHUB_TOKEN` in MCP server `.env`.
- **HTTP 422 Unprocessable Entity** — `Validation Failed` on `head` field. Cause: `.aafm-config.md` `github_log_repo` pointed at the wrong repository.
- **HTTP 404 Not Found** — Repository not found. Cause: regex `([^/.]+)` truncated dotted repository names. Fix: `([^/]+)`.
- **`ValueError`** (in-process, `leaderboard.py`) — Raised by `add_score` when `validate_name` returns False: `"Invalid name '{name}': must be 1-20 alphanumeric characters"`.

## 7. Identifier Conventions

- Python constants in `UPPER_SNAKE_CASE` at module top.
- Python attributes in `lower_snake_case`; "private" timer/handle attributes prefixed with `_` (e.g. `_flash_after_id`, `_border_flash_after_id`).
- JavaScript constants in `UPPER_SNAKE_CASE` (e.g. `STORM_TRIGGER_COUNT`); state in `camelCase` (e.g. `stormActive`).
- Boolean state flags named `_active` or `_state` (e.g. `storm_active`, `mine_flash_state`, `border_flash_state`).


