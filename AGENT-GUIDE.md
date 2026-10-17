# Adding Domination to ReSkate: a guide for the maintainers' coding agent

Three independent changes, applied as patches. Each is one commit. Read this whole file before starting; it is short.

| Order | Folder | Change | Needs |
|---|---|---|---|
| 1 | `patches/1-domination/` | **Domination**: the mode, from the console (`domination ...`) and, when 2 is in, its own tab; per-player play counts | nothing |
| 2 | `patches/2-custom-multiplayer/` | **Custom Multiplayer**: modes as tabs on the Multiplayer page, hosting and finding games by mode, dropdown rows | nothing |
| 3 | `patches/3-server-stats/` | **Server: play stats**: dedicated servers count games by mode and map (`data/stats.json`, `stats`) | nothing |

Any of them can be merged, in any order. Domination's tab needs no extra patch: `domination_menu.cpp` is built against Custom Multiplayer only when `Extension/CustomModes/custom_modes.h` exists (`cmake/Runtime.cmake` sets `DINGOSDK_CUSTOM_MODES`; a change to that file re-runs CMake). The patches in this folder are against `main` at `20d58a4`; `patches/for-2.0.7/` holds the same three against `v2.0.7`.

## 1. Apply

```
git checkout -b domination main && git am -3 patches/1-domination/*.patch
git checkout -b custom-multiplayer main && git am -3 patches/2-custom-multiplayer/*.patch
git checkout -b server-stats main && git am -3 patches/3-server-stats/*.patch
```

Always pass `-3`. If `main` has moved and `git am` stops, the conflict is one of the kinds in section 3; resolve it, `git add`, `git am --continue`.

## 2. What each change touches outside its own folder

Everything new lives in `Extension/Domination/` (change 1) and `Extension/CustomModes/` (change 2). The lines added to existing files are few, and each is listed here so a conflict can be understood at a glance.

**Change 1, Domination**
| File | What is added |
|---|---|
| `CMakeLists.txt` | `DINGOSDK_BUILD_DOMINATION_TESTS`: the two test programs (rules, and whole multiplayer rounds) |
| `cmake/Runtime.cmake` | Domination's `.cpp` files in `dingosdk_runtime` (`domination_tools.cpp` included: the park-making commands), and `DINGOSDK_CUSTOM_MODES` on `domination_menu.cpp` when `Extension/CustomModes/custom_modes.h` exists |
| `cmake/UI.cmake` | `domination_overlay.cpp` in the overlay library |
| `Extension/Console/commands.cpp/.h` | `register_domination_commands` |
| `Runtime/client_tick.cpp` | `domination::on_client_tick()` and `domination::set_level()` |
| `Runtime/runtime_init.cpp` | `domination::start(base)` and `domination::register_custom_mode()` (its tab; does nothing without Custom Multiplayer) |
| `Extension/Multiplayer/Session/session.cpp` | in `relay_throwdowns`: Domination's messages (first byte `0xD0`) go to `domination::receive`, the rest to the throwdown relay as before; `domination::take_outgoing()` is sent with the relay's |
| `Extension/UI/Overlay/overlay_render.cpp` | Domination's overlay drawn, and `domination_frame` keeps the overlay drawing while a round is on screen |
| `Extension/Profile/local_profile_objects.cpp`, `local_profile_runtime.h` | `teleport_local_skater_heading` (the start teleport, facing the park) |
| `Extension/Objects/local_placements_runtime.cpp/.h` | the heading the teleport above uses |
| `Extension/Multiplayer/Test/mesh_tests.cpp` | stubs for the three Domination calls the session makes (like the relay's stubs there) |

No protocol, relay or server change: Domination's messages ride inside the session's throwdown packets.

**Change 2, Custom Multiplayer**
| File | What is added |
|---|---|
| `Extension/UI/Overlay/hub_page.cpp` | the modes' tabs on the Multiplayer page and their panel (settings, adjust rows, dropdowns, status, buttons; PLAY ONLINE: host and find games by mode) |
| `Extension/UI/Overlay/overlay.h`, `Extension/UI/NativeMenu/native_menu.cpp` | the Multiplayer page shows offline when a mode is installed |
| `Engine/Game/Multiplayer/session_model.h` | `MultiplayerLobby::mode`, `valid_mode_id` |
| `Extension/Multiplayer/Session/session*.cpp/.h`, `Steam/steam_lobbies.*`, `Steam/steam_server_browser.cpp` | the advertised mode (`rs_mode` lobby data, the server's `g` tag) read and written |
| `Server/*` | `[server] mode = "<id>"` advertised as the `g` tag |
| `Runtime/client_tick.cpp` | `custom_modes::tick()`, and the modes' panels in the overlay model |
| `Extension/Console/commands.cpp/.h` | `register_custom_modes_commands` (`modes list|host|cancel`) |
| `cmake/Runtime.cmake` | `custom_modes.cpp`, `custom_modes_commands.cpp` |
| `Extension/Multiplayer/Test/lobby_tests.cpp`, `protocol_tests.cpp` | the mode in lobby data and server tags |

Change 1 also adds, inside `Extension/Domination/` only: `parks/` (the built-in parks as files, and their specs), `tools/` (park-making and packing scripts: Python, not built) and `MAKING-PARKS.md`. `domination_parks.h` is generated from `parks/` by `tools/make_builtin.py`; regenerate it rather than editing it.

Play counts (change 1) use `profile_runtime::local_value` / `set_local_values` (keys `ReSkate.Domination.*`), the store Hall of Meat keeps its bests in. Nothing is sent anywhere.

**Change 3, Server: play stats** (new: `Server/server_stats.*`, `Server/Test/server_stats_tests.cpp`)
| File | What is added |
|---|---|
| `Server/server_host.cpp/.h` | the `PlayStats` member; `stats_.relayed(...)` next to `activity_.throwdown(...)` in the throwdown branch; `stats_.tick` and the save in `Host::tick`; a save in `Host::stop` |
| `Server/server_commands.cpp` | `stats`, and its help line |
| `Server/server_engine.*`, `Server/GUI/server_gui.cpp` | `ServerStatusSnapshot::played`, and the overview's "Played Here" row |
| `cmake/Apps.cmake` | `Server/server_stats.cpp` in both `dingosdk_server` targets |
| `CMakeLists.txt` | the `server_stats` test (with the multiplayer tests), and `Server/server_stats.cpp` in the two server tests that build the whole host (`server_park_tests`, `server_load_tests`) |

It reads Domination's message header (`0xD0`, version, type, park, round) itself, so it needs no Domination code.

## 3. Conflicts you may meet, and the fix

- **Two lists grew at the same spot** (`cmake/Runtime.cmake` source lists, the command registrations, `client_tick.cpp` calls, includes at the top of `session.cpp`): keep both sides' lines.
- **`overlay_render.cpp`'s "keep drawing" condition** (`if (s.loaded_notice_posted && !notices_pending() && !chat_frame && ...) return;`): `main` adds its own `*_frame` names there over time. Keep all of `main`'s and add `!domination_frame`.
- **`relay_throwdowns` in `session.cpp`**: if `main` changed how the inbox is read, keep `main`'s code and route a message with `domination::wire::is_domination(message)` to `domination::receive(sender, message)` instead of the relay; send `domination::take_outgoing()` where the relay's outgoing messages are sent.
- **Never take a whole file from a patch** over `main`'s version (for example `cmake/Runtime.cmake`): `main` adds source files there, and dropping them breaks the link. Only add lines.

## 4. Check the result

Build (MSVC, as in the repository's presets):
```
cmake --preset vs2022-x64 -DDINGOSDK_BUILD_DOMINATION_TESTS=ON -DDINGOSDK_BUILD_MULTIPLAYER_TESTS=ON
cmake --build build/vs2022-x64 --config Release
ctest --test-dir build/vs2022-x64 -C Release -R "domination|multiplayer"
build/vs2022-x64/Release/dingosdk_domination_net_tests.exe fuzz 1000
```
Expect: `domination` all passed; `domination_net` 1,343 checks passed; `fuzz: 0 of 1000 rounds disagreed`; the multiplayer tests pass (`multiplayer_mesh` included). With change 3: `ctest -R server` passes (`server_stats` included).

In the game (offline is enough), from the console (`~`):
```
domination parks                     # ten parks, their maps, and whether each map is installed
domination set map next              # each map's first park; then:
domination set park next             # the next park on that map
domination start solo 60             # a round alone; on another map it loads that map first
domination status
domination stats                     # rounds played to the end: in all, with others, per map and park
```
Then the pause menu: Multiplayer > DOMINATION (with change 2) shows the Map and Park dropdowns, the round settings and Start round. On a dedicated server with change 3, `stats` lists what was played. The red-bowl park needs the Skate 3 map mod (Thunderstore `333-Skate_3_Improved`); the five San Vansterdam parks need nothing.

## 5. Where to read more

- `Extension/Domination/README.md`: the rules, the commands, park files and how to make one.
- `Extension/Domination/ARCHITECTURE.md`: the rules apart from the game (`domination_model`, `domination_sync`, `domination_wire`), every touchpoint outside the folder, one tick, the invariants that keep players in agreement, and how to change things safely (for example: a change to the messages bumps `wire::version`).
- `Extension/CustomModes/README.md`: how a mode plugs into the Multiplayer page.
- `SUBMITTING.md`: the pull request texts, what was tested and how, and the pictures.
