# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Afternoon Brainstorming** (午後激盪) is a two-player tactical card game implemented in Python with Pygame. Players build 12-card decks and battle on a grid board; the first player to lead by 10 points wins. The game supports local play, LAN multiplayer (host/join), and replay playback.

All source code lives under `FOS brainstorming/` (note the space in the directory name — always quote or escape it in shell commands).

## Commands

All commands must be run from inside `FOS brainstorming/`:

```bash
cd "FOS brainstorming"

# Run the game
python main.py

# Run all tests
python -m pytest

# Run a single test file
python -m pytest tests/test_card_white.py

# Run a specific test class or method
python -m pytest tests/test_card_white.py::TestWhiteAp
python -m pytest tests/test_card_white.py::TestWhiteAp::test_ability_numbs_target
```

Dependencies: `pygame==2.6.1`, `matplotlib==3.10.8`, `pytest>=8.0`. Install with `pip install -r requirements.txt`.

## Architecture

### Entry point and game flow

`main.py` drives the top-level scene loop:
1. `start_screen` → choose mode (local / host / join / playback)
2. `screens/draft/draft.py` → deck selection phase (`DraftState`)
3. `screens/battling/battling.py` → combat phase (`GameState`)
4. `screens/end_game/end_game.py` → stats and charts

`GameState` (`core/game_state.py`) is the single source of truth for all mutable game data: board, players, score, tokens, combat queue, RNG, timers. It is a `@dataclass` and exposes `to_dict()` / `apply_dict()` for LAN serialization.

### Cards

`cards/base.py` — `Card` is the abstract base for every unit and spell. Key lifecycle hooks that subclasses override:

| Hook | When called |
|---|---|
| `deploy(gs)` | When the card is placed on the board |
| `attack(gs)` | When the card attacks (returns `bool` for success) |
| `on_hit(target, gs)` | After this card's attack hits a target |
| `on_settle()` | At end of turn for scoring/effects |
| `on_death(gs)` | When the card is killed |
| `update(gs)` | Each frame (for animation state only) |

Concrete factions live in `cards/card_white.py`, `card_red.py`, `card_blue.py`, `card_cyan.py`, `card_dark_green.py`, `card_fuchsia.py`, `card_green.py`, `card_orange.py`, `card_purple.py`.

**Card ID format:** `<JOB><COLOR_CODE>` — e.g., `ADCW` = White ADC, `APR` = Red AP, `ASSB` = Blue Assassin. Color codes: W=White, R=Red, G=Green, B=Blue, O=Orange, DG=DarkGreen, C=Cyan, F=Fuchsia, P=Purple.

**Registration:** Every card class must call `CardFactory.register("<ID>", ClassName)` at module level. `CardFactory.register_all()` (called once in `main.py`) imports all card modules to trigger registration. Use `CardFactory.create("ADCW", owner, x, y)` or `spawn_card(...)` from `cards/factory.py` to instantiate.

### Config-driven stats

All base stats (health, damage, special numeric values) live in `config/card_setting.json` under the faction name and class key. Card constructors pull defaults from `shared/setting.py` which loads all JSON configs at import time. Tests access the same `CARD_SETTING` dict to assert against config values rather than hard-coded numbers.

### LAN multiplayer

`core/network_layer.py` — `LANServer` and `LANClient` use a simple length-prefixed JSON protocol over TCP (port 5555). The server is authoritative: it executes every `GameAction`, serializes the full `GameState` via `to_dict()`, and broadcasts it to the client.

`core/battling_dispatcher.py` — `BattlingDispatcher` wraps action execution and handles the three modes (`local`, `lan_server`, `lan_client`). In LAN client mode it sends a `GameAction` JSON to the server and waits for a state push.

`core/game_action.py` — `GameAction` is the action datatype (player, action_type, optional board_x/y, hand_index). `ActionType` literals enumerate all valid actions.

### Rendering (Pygame)

`rendering/` contains stateless renderers that accept `GameState` and draw to the Pygame surface:
- `game_renderer.py` — top-level compositor
- `board_renderer.py` — grid and units
- `card_renderer.py` — individual card sprites
- `combat_animator.py` — plays `CombatEvent` sequences (lunge, hurt, float damage numbers)
- `draft_renderer.py`, `ui_renderer.py`, `end_game_renderer.py` — phase-specific UIs
- `sprite_registry.py` — loads and caches all image assets

`CombatEvent` (`shared/combat_event.py`) is a small dataclass queued in `GameState.pending_combat_events`. The animator consumes them independently of game logic, enabling deterministic logic replay without Pygame.

### Testing

Tests live in `tests/` and use pytest with no special plugins. The helper module `tests/helpers.py` provides:

- `make_game_state(rng_seed=42)` — creates a minimal `GameState` with no Pygame dependency
- `place_card(gs, CardClass, owner, x, y)` — spawns a card directly onto `on_board` (bypasses deploy effects)
- `do_attack(attacker, gs)` — temporarily clears `numbness` so attack effects can be tested regardless of initial state

Tests are purely logic-layer; there are no rendering or Pygame tests. Each `test_card_<faction>.py` file tests one faction's ability interactions.
