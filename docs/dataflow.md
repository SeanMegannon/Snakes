# DataFlow Context Pack

## 1. Data Models and Schemas

### Snake body

- Python (Pygame): `Snake.body` is `list[tuple[int, int]]`. Initial value: `[(GRID_WIDTH // 2, GRID_HEIGHT // 2)]`.
- Python (Tkinter): `SnakeGame.snake` is `list[tuple[int, int]]`. Initial: `[(GRID_WIDTH // 2, GRID_HEIGHT // 2)]`.
- Python (Console): `Snake.body` is `list[tuple[int, int]]`.
- JS (HTML): `snake` is `Array<{x: number, y: number}>`.

### Direction

- Python: `tuple[int, int]` constants `UP=(0,-1)`, `DOWN=(0,1)`, `LEFT=(-1,0)`, `RIGHT=(1,0)`.
- JS: object literals `{ x: 0, y: -1 }`, etc.

### Food

- Python (Pygame): `Food` class with `.position: tuple[int, int]`.
- Python (Tkinter): `self.food: tuple[int, int]`.
- Python (Console): `Food` class with `.position: tuple[int, int]`.
- JS: `food: { x, y }` plain object.

### Mines / bonus foods / super food

- Python: `mines: list[tuple[int, int]]`, `bonus_foods: list[tuple[int, int]]`, `super_food_position: tuple[int, int] | None`.
- JS: `mines: Array<{x, y}>`, `bonusFoods: Array<{x, y}>`, `superFoodPosition: {x, y} | null`.

### Storm state

- `storm_active: bool`
- `storm_queue: list[tuple[int, int]]` (or `Array<{x,y}>`)
- `storm_phase: 'warning' | 'explosion' | None`
- `storm_current_mine: tuple[int, int] | None`
- Pygame timing: `storm_phase_elapsed: float`, `storm_warning_flash_acc: float`, `border_flash_acc: float`, `storm_warning_flash_index: int`.
- Tkinter timing: `_storm_phase_after_id`, `_warning_flash_after_id`, `_border_flash_after_id` (each `root.after` id or None).
- HTML timing: `stormPhaseTimer`, `stormWarningFlashTimer`, `borderFlashTimer` (each timer id or null).
- Console timing: `storm_phase_ticks: int`.

### Invincibility / super-food selection

- `super_food_mine_index: int | None` — index inside `storm_queue` chosen at storm start.
- `super_food_mine_counter: int` — incremented each time a mine transitions warning -> explosion.
- `invincibility_active: bool`.
- Pygame: `invincibility_timer_remaining: float` (seconds).
- Tkinter: `invincibility_timer_remaining: int` (seconds), with `_inv_tick_after_id` and `_inv_end_after_id`.
- Console: `invincibility_ticks_remaining: int` (ticks; HUD displays `// 10` as seconds).
- HTML: `invincibilityTimerRemaining: number` (seconds), with `invincibilityCountdownTimer`, `invincibilityEndTimer`.

### High-score entry (`leaderboard.py`)

JSON object shape (observed in `highscores.json`):

```json
{
  "score": 48,
  "name": "GRA",
  "date": "2026-05-08T10:51:13.524798Z",
  "food_eaten": 51,
  "mine_shrinks": 1,
  "invincibility_count": 0
}
```

### High-score file envelope

```json
{
  "schema_version": 2,
  "scores": [ /* up to 10 entries, sorted desc by score */ ]
}
```

Legacy format (auto-migrated on load): a bare list of entry objects without the `stats` fields. Missing fields are filled with `0` via `_default_stats`.

## 2. Data Transformation Pipelines

### Score lifecycle

```
input event -> direction update
            -> snake.move() -> new head
            -> collision/consumption checks
            -> score += 1 (food/bonus) or score = max(0, score - 3) (mine)
            -> displayed in HUD
            -> on game over (Tkinter only) -> HighScoreManager.add_score(score, name, stats)
            -> persisted to highscores.json
```

### Mine lifecycle

```
food consumed -> spawn_mine() (up to 1000 attempts)
              -> if expected > len(mines), append candidate
              -> rendered with flash alternation (Pygame/Tkinter/HTML)
              -> on snake.next_head == mine: mines.remove(mine), snake shrink, score -= 3
              -> when len(mines) >= 10: startStorm()
                  -> mines moved into storm_queue; mines cleared
                  -> queue drained one-at-a-time (warning -> explosion)
                  -> each detonation appends to bonus_foods
                  -> exactly one detonation also sets super_food_position
```

### High-score migration pipeline (legacy -> v2)

```
open(highscores.json) -> json.load -> if list:
  migrated = [{food_eaten:0, mine_shrinks:0, invincibility_count:0, **entry} for entry in raw]
  save_scores(migrated)  # persists v2 envelope
  return migrated
elif dict with "scores": return raw["scores"]
else: return []
```

## 3. Input/Output Flows and APIs

### Input sources

- Pygame: `pygame.event.get()` -> `KEYDOWN` events (`K_UP`, `K_DOWN`, `K_LEFT`, `K_RIGHT`, `K_SPACE`, `K_ESCAPE`) and `MOUSEBUTTONDOWN` (rules dismissal).
- Tkinter: `root.bind('<Up>', ...)`, etc.; `simpledialog.askstring` for player name.
- Console: `input().lower()` returning `w/a/s/d/p/q`.
- HTML: `document.addEventListener('keydown', ...)` for `ArrowUp/Down/Left/Right`, `Space`, `p`, `Escape`.

### Output sinks

- Pygame: `pygame.display.flip()` (screen), `pygame.mixer.music` (audio), `print(..., file=sys.stderr)` (errors).
- Tkinter: `Canvas.create_*` (screen), `winsound.PlaySound` (audio), `messagebox.showinfo/showerror`, `simpledialog.askstring`, `Toplevel` for leaderboard.
- Console: `print` to stdout with ANSI escapes; `os.system('cls' or 'clear')` per frame.
- HTML: `ctx.fillRect/fillText/...` on canvas, `document.getElementById('score').textContent`, `Audio.play()`, `console.warn`.

### External "API" surface

- The game code itself exposes no HTTP/RPC API.
- `Auto-PR-Solution.md` documents an out-of-band integration with `https://api.github.com/repos/<owner>/<repo>/pulls` via the MCP server `aafm-server.js` (`AAFM_GITHUB_TOKEN` Bearer auth). This is tooling, not part of the game runtime.

## 4. Database Interactions and Queries

There is no database. The single persistent store is the JSON file `highscores.json`.

- "Query top 10": `HighScoreManager.get_leaderboard()` -> `load_scores()` returns the (already top-10) list as persisted.
- "Insert score": `add_score(score, name, ...)` -> load -> append -> sort desc -> truncate to 10 -> save.
- "Check qualification": `is_high_score(score)` -> `len(scores) < 10 or score > scores[-1]['score']`.
- "Validate name": `validate_name(name)` -> `1 <= len(name) <= 20 and name.isalnum()`.

There is no indexing, no concurrent-write protection, and no transactional guarantee beyond the `open(..., 'w')` followed by `json.dump`.

## 5. State Management and Data Persistence

### In-memory state

- Pygame: storm/inv/super-food state are locals in `main()`. `food_particles` is a module-level global list.
- Tkinter: storm/inv/super-food state are attributes on the `SnakeGame` instance.
- Console: state lives as locals in `main()`.
- HTML: state lives as module-level `let` variables in the `<script>` block.

### Persisted state

- `highscores.json` — only written by `HighScoreManager.save_scores`. Always overwritten in full (no append, no partial update).
- No other on-disk state. The Pygame `INVINCIBILITY_MUSIC_PATH`, Tkinter `INVINCIBILITY_MUSIC_PATH`, and HTML `INVINCIBILITY_MUSIC_PATH` reference an audio file but never write to disk.

### Reset semantics

- Pygame "restart" reinitialises all locals when SPACE is pressed at game over.
- Tkinter `restart_game` -> `reset_game` reinitialises instance attributes and cancels all `after` callbacks.
- HTML `init()` reinitialises module variables, clears all interval/timeout handles, and conditionally shows rules screen.
- Console game ends on `game_over`; no in-process restart loop.

## 6. Data Validation and Sanitization

- Player name: `HighScoreManager.validate_name`: `1 <= len(name) <= VALID_NAME_MAX_LENGTH (20)` AND `name.isalnum()`. `add_score` raises `ValueError` if validation fails.
- Score floor: `score = max(0, score - 3)` after mine impact.
- Mine placement: validated by `is_valid_mine_position` (five constraints) before insertion.
- Food placement: rejection sampling — re-roll until `pos not in snake_body`.
- Direction: reversing is rejected by `change_direction` guard.
- Tkinter name dialog: `name.upper()` applied after validation.
- JSON load: malformed files yield `[]` via `except (json.JSONDecodeError, IOError)`.
- JSON save: write failures yield `False` via `except (IOError, OSError)`; `add_score` then re-loads to avoid surfacing an unpersisted candidate list.
