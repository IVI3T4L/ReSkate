# Server: play stats (what has been played here, by mode and map)

A dedicated server now counts the games played on it, by mode and map, and keeps the counts across restarts. Server owners can see what their server is used for. It is also a first step towards ReSkate-wide stats (see the proposal issue).

## What's in it

- **Counted from the packets the server relays anyway.** Nothing it sends changes, and nothing leaves the server.
  - **Throwdowns:** Jam, Spot Battle and S.K.A.T.E., by the series in the leader's offer, counted once at their start with the players in it. Coop challenges are counted at their start too.
  - **Domination rounds** (when Domination, #1, is in the game): a round is counted the first time any of its messages is heard, and each player the first time they send for it. The server reads only the messages' header (`0xD0`, version, type, park, round), so it needs no Domination code.
- **Per mode and map:** games, and players summed over them.
- **`data/stats.json`**, beside `bans.json`. It is saved at most once a minute while games are played, and when the server stops, written beside and moved over. An unreadable file is left as it is, and the log says so.
- **`stats`** on the console (or from an admin):
  ```
  Played here since 2026-10-16:
    Domination: 12 (41 players in all; Skate 3 Improved 9, San Vansterdam 3)
    Jam: 4 (9 players in all; San Vansterdam 4)
  ```
- **The server window:** a "Played Here" row on the overview ("Domination 12, Jam 4").

## Tested

- **Unit tests** (`server_stats`):
  - throwdowns: counted at their start, once; by series; a start without its offer; coop challenges;
  - Domination rounds: each player once; another park or round number is another round; older versions and short messages are not read; a round is forgotten after ten quiet minutes;
  - the file: saved, read back, and an unreadable one.
- ReSkate's whole test suite passes with it (52 of 52 on `main`, 46 of 46 on `v2.0.7`). It builds alone and with #1 and #2.
- **Not yet tried:** a live server with players on it.

## For reviewers and agents

- **Independent:** it touches only `Server/` (plus its source file in the server targets of `cmake/Apps.cmake` and `CMakeLists.txt`, and its test), and applies over #1 and #2 or without them.
- **Its own files:** `Server/server_stats.*` and `Server/Test/server_stats_tests.cpp`.
- **Lines added to existing files:**
  - `server_host.*`: the `PlayStats` member; `relayed()` in the throwdown branch; `tick()` and the save in `Host::tick`; a save in `stop`;
  - `server_commands.cpp`: `stats`;
  - `server_engine.*` and `GUI/server_gui.cpp`: the overview row.
- **Domination's header:** if a later Domination version moves the park or round, `domination_since` in `server_stats.cpp` and the header read there change with it.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
