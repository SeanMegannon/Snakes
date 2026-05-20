# Code Context Pack

## 1. Key Functions and Classes (with purposes)

### `game_logic.py`

- `check_wall_collision(head, grid_width=GRID_WIDTH, grid_height=GRID_HEIGHT)` — returns True if head is outside `[0, grid_width) x [0, grid_height)`.
- `check_self_collision(head, body)` — returns `head in body[1:]`.
- `get_next_head(head, direction)` — returns `(head_x + dir_x, head_y + dir_y)`. No wrapping or bounds check.
- `calculate_new_score(current_score)` — returns `current_score + 1`.
- `is_valid_food_position(pos, snake_body)` — `pos not in snake_body`.
- `is_valid_mine_position(candidate, snake_body, snake_direction, food_pos, mines)` — five checks: not in body, not within Manhattan 10 of any segment, not on the forward straight-line path until off-grid, not equal to food, not already a mine.
- `spawn_mine(mines, snake_body, snake_direction, food_pos, score)` — mutates `mines` to match `1 + score // 5` count, up to 1000 attempts per mine, silently skips on exhaustion.

### `leaderboard.py` — `HighScoreManager`

- `__init__(filepath='highscores.json')` — stores path; no I/O.
- `_default_stats()` — `{"food_eaten": 0, "mine_shrinks": 0, "invincibility_count": 0}`.
- `migrate_legacy_entries(entries)` — `[{**defaults, **e} for e in entries]`.
- `_unwrap_scores(raw)` — extracts list from `{"scores": [...]}` envelope or migrates a bare list.
- `load_scores()` — reads JSON; on bare list, migrates and persists v2. On `JSONDecodeError`/`IOError`, returns `[]`.
- `save_scores(scores)` — writes `{"schema_version": 2, "scores": [...]}`; returns False on `IOError`/`OSError`.
- `validate_name(name)` — `1 <= len(name) <= 20 and name.isalnum()`.
- `is_high_score(score)` — True if fewer than 10 scores exist or `score > scores[-1]['score']`.
- `add_score(score, name, food_eaten=0, mine_shrinks=0, invincibility_count=0)` — validates name (raises `ValueError`), appends new entry with UTC ISO timestamp, sorts desc, truncates to top 10, persists. Returns the list (or `load_scores()` if save failed).
- `get_leaderboard()` — alias for `load_scores()`.

### `snake_game.py` (Pygame)

- `class Snake`:
  - `__init__()` — center-grid start, direction `RIGHT`, `animation_frames=5`, `color_index=0`.
  - `move()` — at `current_frame == 0` computes `next_head`; at `>= animation_frames`, commits insertion and pops tail unless `grow`.
  - `change_direction(new_direction)` — guards against reversing.
  - `check_collision()` — wall or self collision on `next_head`.
  - `eat_food(food_pos)` — sets `grow=True`, cycles `color_index`.
  - `get_interpolated_body()` — produces interpolated head for in-between frames.
- `class Food`:
  - `__init__(snake_body)`, `generate_position(snake_body)` — random until not in `snake_body`.
- Render functions: `draw_grid`, `draw_snake`, `draw_food`, `draw_score`, `game_over_screen`, `draw_rules_screen`, `draw_gradient_background`, `draw_mines`, `draw_storm_border`, `draw_explosion_cells`, `draw_warning_cell`, `draw_bonus_foods`, `draw_super_food`, `draw_food_particles`, `draw_pause_overlay` (referenced but not defined in current file).
- `create_food_particles(position)` — appends 10 particle dicts with `vel` and `ttl`.
- `_start_invincibility_music`, `_stop_invincibility_music` — wrap `pygame.mixer.music.load/.play/.stop` in `try/except`.
- `main()` — event loop, owns all storm/invincibility/super-food state as locals.

### `snake_game_tkinter.py`

- Module-level helpers: `_format_row`, `format_leaderboard_message`, `_show_leaderboard_window` (Toplevel + monospace `Text`).
- `class SnakeGame`:
  - `__init__(root)` — builds Canvas, score and inv labels, instantiates `HighScoreManager`, binds keys, kicks off `game_loop`.
  - `reset_game()` — initialises snake/food/mines/storm/inv state; cancels prior `after` IDs via `hasattr` guards; resets stats counters.
  - `generate_food()`, `change_direction(new_direction)`, `move_snake()` (returns `True` on game over).
  - `draw()` — clears canvas, draws grid, storm border, explosion/warning cells, mines, bonus foods, super-food, snake, food, optional game-over and pause overlays.
  - `game_loop()` — calls `move_snake()`; on collision, schedules `handle_game_over` via `root.after(100, ...)`; redraws; reschedules itself with `root.after(GAME_SPEED, ...)`.
  - `prompt_for_name()` — `simpledialog.askstring`; validates via `HighScoreManager.validate_name`; loops until valid or cancelled.
  - `display_leaderboard()` — formats and shows in `_show_leaderboard_window`.
  - `handle_game_over()` — checks high score, prompts name, calls `add_score`, displays leaderboard.
  - `toggle_pause`, `_draw_rules`, `_dismiss_rules`.
  - `_toggle_mine_flash`, `_toggle_super_food_flash`, `_activate_invincibility`, `_tick_invincibility`, `_end_invincibility`.
  - `_try_spawn_mine` — inlined version of `spawn_mine` (does not import `game_logic`).
  - `_start_storm`, `_begin_warning_phase`, `_begin_explosion_phase`, `_advance_storm_phase`, `_end_storm`, `_toggle_border_flash`, `_toggle_warning_flash`.

### `snake_game_console.py`

- `class Snake` (wrapping movement via `% GRID_WIDTH/HEIGHT`).
- `class Food` (rejection sampling on snake body).
- `spawn_mine` (inline, with wrapped path projection and `visited` guard).
- `draw_game(...)` — clears terminal, builds a `GRID_HEIGHT x GRID_WIDTH` grid of characters, prints box-drawing borders and ANSI-colored cells, optional storm/pause/inv HUD.
- `show_rules_screen()` — prints fixed rules block; blocks on `input()`.
- `main()` — turn-based loop; reads `input().lower()` each tick.

### `snake_game.html` (JS)

- Functions: `startStorm`, `beginWarningPhase`, `beginExplosionPhase`, `endStorm`, `activateInvincibility`, `endInvincibility`, `drawRulesScreen`, `init`, `generateFood`, `trySpawnMine`, `update`, `draw`, `endGame`.
- Storm state as module-level `let` variables (`stormActive`, `stormQueue`, `stormPhase`, ...).
- Mine flash via `setInterval(MINE_FLASH_INTERVAL=200ms)`; storm border via `setInterval(STORM_BORDER_FLASH_INTERVAL=200)`; warning cell via `setInterval(WARNING_FLASH_INTERVAL=200)`; phase transitions via `setTimeout(WARNING_DURATION_MS=3000)` and `setTimeout(EXPLOSION_DURATION_MS=1000)`.

## 2. Important Algorithms and Business Logic

### Mine spawn validation (`is_valid_mine_position` in `game_logic.py`)

```
candidate must satisfy:
  - not in snake_body
  - for all (sx, sy) in snake_body: abs(cx-sx) + abs(cy-sy) > 10  (Manhattan)
  - candidate not on path: starting from head + direction, step by direction until off-grid
  - candidate != food_pos
  - candidate not in mines
```

Up to 1000 retries per mine in `spawn_mine`; on exhaustion, silently skip.

### Mine spawn (console variant) — `snake_game_console.py`

Identical to `game_logic.spawn_mine` except path projection uses modulo wrap and a `visited` set to terminate, because the console wraps at edges. Concretely: `px = (hx + dx) % GRID_WIDTH; py = (hy + dy) % GRID_HEIGHT; while (px,py) not in visited: ...`.

### Storm trigger and phases

- Trigger: in food-eat path, if `not storm_active and len(mines) >= STORM_TRIGGER_COUNT (=10)`, begin storm.
- Start: shuffle `mines` into `storm_queue`; clear `mines`; pick a `super_food_mine_index = randint(0, len-1)`; pop first from queue into `storm_current_mine`; enter `'warning'` phase.
- Warning -> Explosion transition (after `WARNING_DURATION`):
  - If `super_food_mine_counter == super_food_mine_index`, set `super_food_position = storm_current_mine`.
  - Increment `super_food_mine_counter`. Append `storm_current_mine` to `bonus_foods`.
  - Enter `'explosion'` phase.
- Explosion -> next Warning / End (after `EXPLOSION_DURATION`):
  - If queue non-empty, pop next; else end storm.
- During `'explosion'`: blast cells are the 3x3 neighborhood around `storm_current_mine`, clamped to grid; if any snake segment is in a blast cell and not invincible, end storm and game over.

### Mine collision penalty (all versions)

- Remove mine from `mines`.
- `shrink = min(3, len(snake_body))`; `del snake_body[-shrink:]`.
- `score = max(0, score - 3)`.
- If snake length reaches 0, game over.

### High-score migration

`HighScoreManager.load_scores` detects a bare list (legacy) and writes back a v2 envelope on first load: `{"schema_version": 2, "scores": [...]}`. Migration adds missing stats keys via `{**self._default_stats(), **e}`.

### Snake colour cycle

Each food/bonus consumption increments `color_index = (color_index + 1) % len(SNAKE_COLORS)` in Pygame, Tkinter, Console, and HTML.

## 3. Code Patterns and Conventions

| Pattern | Where | Example |
|---|---|---|
| Module-level UPPERCASE constants | All frontends | `MINE_FLASH_INTERVAL`, `STORM_TRIGGER_COUNT`, `INVINCIBILITY_DURATION_MS` |
| Plain list for mine state | All | `mines = []` |
| Boolean flag state machine | All | `storm_active`, `paused`, `game_over`, `invincibility_active` |
| Accumulator timing | Pygame | `mine_flash_accumulator += dt` |
| `root.after(ms, fn)` self-rescheduling | Tkinter | `self._flash_after_id = self.root.after(MINE_FLASH_INTERVAL, self._toggle_mine_flash)` |
| `setInterval` / `setTimeout` | HTML | `mineFlashTimer = setInterval(...)`, `stormPhaseTimer = setTimeout(...)` |
| Tick-based timing | Console | `WARNING_TICKS = 3`, `EXPLOSION_TICKS = 1` |
| `hasattr` guard before timer cancel | Tkinter | `if hasattr(self, '_flash_after_id'): self.root.after_cancel(...)` |
| Immediate-mode rendering | All | Clear-then-draw every frame |
| Defensive try/except around audio | Pygame, Tkinter | `try: pygame.mixer.music.load(...) except Exception as e: print(..., file=sys.stderr)` |
| Defensive `.catch`/try around audio | HTML | `invincibilityAudio.play().catch(...)` |
| Validation-by-isalnum + length | leaderboard | `1 <= len(name) <= 20 and name.isalnum()` |
| Schema-versioned JSON envelope | leaderboard | `{"schema_version": 2, "scores": [...]}` |
| Source-text testing (no import) | tests | `test_food_color.py` reads `snake_game.py` and regex-matches `FOOD_COLOR` |

## 4. Critical Code Paths and Workflows

### Pygame per-frame path (`main()` in `snake_game.py`)

1. Process events. If `show_rules`, dismiss on any key/click.
2. `snake.move()` — advances animation frame; commits position when frame counter resets.
3. `dt = clock.get_time() / 1000.0` — frame delta.
4. Update mine-flash, super-food-flash, invincibility timers.
5. If storm active, update border flash, warning flash, phase elapsed; transition phases.
6. Check mine collision against `snake.next_head`.
7. Check bonus food, super food, regular food consumption.
8. Check `snake.check_collision()`.
9. Draw everything; `pygame.display.flip()`; `clock.tick(60)`.

Note: the event loop in `snake_game.py` contains an unconditional `snake.move()` call inside the `for event in pygame.event.get():` loop (lines 271-455), which means movement happens once per event rather than once per frame. This is observed verbatim in the source.

### Tkinter game loop (`game_loop`)

1. If `not self.game_over`, call `move_snake()`. If returns True, set `game_over = True` and schedule `handle_game_over` via `root.after(100, ...)`.
2. `self.draw()`.
3. `self.root.after(GAME_SPEED, self.game_loop)`.

Storm and flash are driven by independent `after` callbacks that reschedule themselves.

### Console game tick

1. Render frame (full screen clear).
2. If not paused: handle food/bonus consumption, storm timers, mine collision, self-collision, invincibility countdown.
3. Block on `input()` for direction or `p`/`q`.
4. `time.sleep(0.1)`.

Note: the source file has an indentation issue at the `storm_phase_ticks -= 1` line nested under `if not paused and storm_active:` (lines 290-291) — the inner line is not indented relative to the if header. This is verbatim in the file and would raise `IndentationError` on import.

### HTML game loop (`update`)

1. If `gameOver` or `paused`, return.
2. `direction = nextDirection`; compute `head`.
3. Wall collision (with invincibility bypass).
4. Self collision (with invincibility bypass).
5. Phase-2 explosion blast check (with invincibility bypass).
6. Mine collision (with invincibility bypass).
7. `snake.unshift(head)`.
8. Super-food, bonus-food, regular-food handling. Storm trigger after regular-food.
9. `draw()`.

## 5. Error Handling and Validation Patterns

- `HighScoreManager.load_scores`: swallows `json.JSONDecodeError` and `IOError`, returns `[]`.
- `HighScoreManager.save_scores`: swallows `IOError`/`OSError`, returns `False`.
- `HighScoreManager.add_score`: raises `ValueError` on invalid name; on save failure, returns `self.load_scores()` instead of the unsaved candidate list.
- Pygame `_start_invincibility_music`: `try/except Exception as e: print(..., file=sys.stderr)`.
- Pygame `_stop_invincibility_music`: `try/except Exception: pass`.
- Tkinter audio: `try: winsound.PlaySound(...) except Exception as e: print(..., file=sys.stderr)`; `winsound.PlaySound(None, winsound.SND_PURGE)` wrapped in bare `try/except Exception: pass`.
- HTML audio: `invincibilityAudio.play().catch(e => console.warn(...))` plus outer `try/catch`.
- Mine spawn: silently skip on 1000-attempt exhaustion (no exception, no log).
- Score floor: `score = max(0, score - 3)` after mine hit.
- Snake-length floor: `del snake.body[-shrink:]` with `shrink = min(3, len(snake.body))`; game over only when length reaches 0.
- Direction reversal guard: `if (new[0]*-1, new[1]*-1) != current_direction`.
- Tkinter name prompt: `messagebox.showerror` and re-prompt until valid or cancelled.

## 6. Performance-Critical Sections

- `draw_gradient_background` in `snake_game.py` issues 600 `pygame.draw.line` calls per frame.
- `draw_food_particles` in `snake_game.py` iterates a growing global list each frame; particles auto-expire at TTL=30 frames.
- `is_valid_mine_position` performs `O(len(snake_body))` work per attempt; up to 1000 attempts.
- `draw_explosion_cells` calls `random.choice(EXPLOSION_COLORS)` for up to 9 cells per frame during phase 2.
- HTML `draw()` redraws grid lines, all mines, all bonus foods, snake, food every frame using Canvas 2D API.
- Console `draw_game` does `os.system('cls' or 'clear')` per frame — significant overhead per tick.
- `HighScoreManager.load_scores` is called inside `is_high_score`, `add_score`, and `get_leaderboard` — each call reopens and re-parses the JSON file.

