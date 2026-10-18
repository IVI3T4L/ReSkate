# Submitting to ReSkate: three pull requests

This was written for the ReSkate maintainers and their coding agents. **Agents: start with [AGENT-GUIDE.md](AGENT-GUIDE.md)** — the exact steps, the files each change touches, what a conflict can look like and how to check the result. Every patch in this folder is made against ReSkate `main` at `99467f4` ("Official mods: an installed one can be uninstalled on the OFFICIAL tab", 2026-10-17), 11 commits after the official `v2.1.0` release (`ba1245a`). Pictures of everything are in [pictures/](pictures/).

**One patch set, three changes, two bases checked:**
- **`1-domination/`, `2-custom-multiplayer/`, `3-server-stats/`**: against `main` at `99467f4`. Use these for the pull requests.
- **On `v2.1.0` too:** 1 and 3 apply to the release as they are; 2 applies with `git am -3` (one line was added beside ours in `Runtime/client_tick.cpp` after the release; keep both). Network protocol (52) and game build (25414733) are the same on both.
- **Tested on `v2.1.0` and on `main` (`99467f4`):** with all three applied, ReSkate's whole test suite (52 of 53 pass on each; the one not run is ReSkate's own `server_config` test, which doesn't compile on either base, see Building and testing), `fuzz 1000` (0 of 1000 rounds disagreed, 28,274 checks), and in game in an offline test copy with the `v2.1.0` build and then the `main` build: a Red-bowl park round with 3 practice bots on each (every copy agreed), and on `main` also a Rolling Waves (San Vansterdam) round with 3 bots (agreed), `domination parks`, `domination stats`, and `domination tp` then `domination scan` (31 x 31 points, all hit).

| Folder | What | Depends on |
|---|---|---|
| `1-domination/` | **Domination**, a custom multiplayer mode, played from the console | nothing |
| `2-custom-multiplayer/` | **Custom Multiplayer**: custom modes plug in as tabs on the Multiplayer page, plus hosting and finding games by mode | nothing |
| `3-server-stats/` | **Server: play stats**: dedicated servers count the games played on them, by mode and map (`data/stats.json`, `stats`) | nothing |

- **Independence:** all three build and work alone. 1 and 2 merge cleanly in either order (checked both ways, with `git merge` and with `git am -3`; both orders give the same tree). They add their lines in different spots of the shared files: the command list, `Extension/Multiplayer/Session/session.cpp`, `Runtime/client_tick.cpp` and `cmake/Runtime.cmake`.
- **Domination's tab comes with 1:** `Extension/Domination/domination_menu.cpp` is compiled with Custom Multiplayer's API only when `Extension/CustomModes/custom_modes.h` is in the tree (`cmake/Runtime.cmake` sets `DINGOSDK_CUSTOM_MODES`); without it, the tab's registration does nothing. So whichever of 1 and 2 lands second, the tab appears with no further patch (checked: 1 alone has no tab; 1 and 2 together have it).
- **3 touches only `Server/`** (and one line each in `cmake/Apps.cmake` and `CMakeLists.txt`), and applies over 1 and 2 or without them.
- **Wire version:** Domination's messages are now version 4 (teams is the only option bit, 1). Copies built from earlier versions of these patches and current ones don't play rounds together; each side is told why, once per player.

## How to apply (agents: follow exactly)

```
git checkout -b domination main
git am -3 path/to/1-domination/*.patch

git checkout -b custom-multiplayer main
git am -3 path/to/2-custom-multiplayer/*.patch

git checkout -b server-stats main
git am -3 path/to/3-server-stats/*.patch
```

Use `-3`: once one of the two is in `main`, the other's patch touches lines next to the first one's in `Runtime/client_tick.cpp`, and plain `git am` stops there. With `-3` it applies cleanly. If `main` has moved further and a conflict remains, it can only be "two changes added a line next to each other"; keep both lines.

## Building and testing

- **Domination:** `cmake --preset vs2022-x64 -DDINGOSDK_BUILD_DOMINATION_TESTS=ON`, then build and run `ctest -R domination`.
  - Optional: `-DDINGOSDK_DOMINATION_PARK=<park.json>` also checks the red-bowl park's file against its built-in copy.
  - `ctest -R domination` runs both `domination` (park files, every built-in park, rules, trick reading, the wire format) and `domination_net`: whole multiplayer rounds between up to 16 simulated players over a bad network, and a dozen random rounds with random trouble (1,343 checks with MSVC, a few seconds; GCC's random numbers differ, so its count does too). Both also build and pass with GCC 13 (MinGW, `-std=c++20`), so a Linux CI can run them.
  - `dingosdk_domination_net_tests fuzz <n> [first seed]` plays n random rounds with random trouble and names the seeds of any whose players disagree (`trace <seed> <player>` tells one player's events in one of them). With MSVC, 11,000 seeds: 0 disagreements. With GCC, one false alarm at seed 2588 (28% loss): a check in the test's own timing window, while the players agreed.
  - `Extension/Domination/ARCHITECTURE.md` is the map for maintainers and agents: the rules apart from the game, every touchpoint outside the folder, one tick, the invariants that keep players in agreement, and how to change things safely.
- **Custom Multiplayer:** `-DDINGOSDK_BUILD_MULTIPLAYER_TESTS=ON`, then build the targets `dingosdk_multiplayer_tests` and `dingosdk_multiplayer_lobby_tests`, and run `ctest -R "multiplayer_protocol|multiplayer_lobbies|native_menu_browser"` (the last checks the pause menu's tab order).
  - That option also builds two music probe tools, which failed to link in our tree. They're unrelated to these changes.
- **Whole suite on `main` (`99467f4`) and on `v2.1.0`:** with all three patches, 52 of ReSkate's 53 tests pass on each. The 53rd, `server_config`, is `Not Run` on both: its own test file, `Server/Test/server_config_tests.cpp` (untouched by these patches; from the tool-locks merge `2f50601`, in `v2.1.0`), doesn't compile with MSVC's warnings-as-errors (C4456: a second local named `written` at line 179 hides the one at line 53). Not ours to fix in these patches (`multiplayer_mesh` included: Domination adds stubs for the session's two Domination calls to its test, as it does for the throwdown relay). The probe tools `dingosdk_app_audio_probe` and `dingosdk_external_player_probe` don't link on `main` itself.
- **In game, offline:** checked with 1 and 2 together, in a separate offline copy of the game. On the current code:
  - rounds on Skate3Improved (the red-bowl park) and on San Vansterdam;
  - `domination start` for a park on another map loads that map, then starts;
  - the Map and Park dropdowns (picked from the console in tests; drawn in the menu), a park whose map is missing included;
  - the red-bowl park chosen by default;
  - round end and the results screen (awards, time held, who owns what);
  - practice-bot rounds (the developer tools in `test-build`, not in these patches: made-up players running the same multiplayer rules inside the game). When the bots leave, the game checks every bot's copy of the round against the player's: all agreed (owners, every claim, end, results).

  On earlier builds of the same features:
  - the park loads from the built-in copy;
  - the start teleport and heading;
  - the 3-2-1 countdown;
  - spot outlines measured on the ground, and edge arrows to spots you don't own;
  - teams falling back to free-for-all alone; with bots: automatic joining, teams handed out, team colours, the feed and team standings;
  - `modes list` lists Domination;
  - offline, hosting refuses with "multiplayer is off in offline mode";
  - the red-bowl park's 20 spots, the four mounds and the curb included, with their outlines measured on the ground.
- **Built-in San Vansterdam parks:** five parks, placed where the game puts its own skate challenges: Rolling Waves (12 spots), Bonski Park under the bridge (8), Hartcore Plaza (12), FlowTopia (9) and El-One Courts (5). Each spot was checked in game with picture tours.
- **Built-in Skate 3 map mod parks:** four more next to the red-bowl park, found by a height-map survey of the map's south half and checked in game: Hillside park (9 spots: a deep bowl, a pool, the full pipe, hips, a pyramid and ledges), Industrial bowls (7), Sunken banks (6) and Loans plaza (6). The red-bowl park keeps its 20 spots.
- **Played by hand:** solo rounds, and the trick reading checked against recordings of the play.
- **Unit tests** for multiplayer: claims delivered in any order give the same board; a late joiner given one whole board gets the same owners and the same replay (hold times, steals); hold times, steals, the best trick and the winner (a tie on spots goes to the total of points, not the time held); teams (no steal from a teammate, team rows, even team picks that never move a placed player); every message kind round-trips, malformed ones are refused, and a full board fits a throwdown packet. Park files: the optional map and package fields, and every built-in park parses.
- **Network simulation** (`Test/domination_net_tests.cpp`), against `domination_sync.h`, the module that holds every multiplayer rule:
  - a clean round; only the host starts rounds; a solo round gives way; a guest who leaves isn't pulled back in; the host's stop ends a round for everyone;
  - 6 players with 20 to 400 ms lag, 10% loss, 5% duplicates, clocks 10 s off and drifting, teams, a late joiner and a disconnect;
  - the host leaving mid-round (the next in line takes over);
  - two rounds started at once;
  - 16 players, a board too big for one message;
  - a guest who leaves and rejoins, or leaves and plays alone; round options reaching everyone; bad claims (outside the play, unknown spots, bad scores) refused; other versions and parks reported once;
  - a stop lost on the way; a stop 0.2 s before the end with clocks apart; teams handed out at the very end; a stop sent once by a controller who then changed maps; the round's authority leaving;
  - a player cut off across a stop who comes back, still running the old round, as the next one starts;
  - 20 random networks, and a dozen random rounds with random trouble.

  In every case the players still in the round end with the same round, claims, owners, teams, end time and results. The simulation found two real bugs, both fixed: a stopped round ended at slightly different moments on each player, so time held and results could differ; and `domination join` didn't put a player who left back into the round.
- **Code review:** an independent review of the whole mode found more problems, each now fixed. Each multiplayer fix has a network test that failed before it:
  - a player who left and then played alone was pulled back in;
  - a stop sent only once (a player who missed it diverged), and a stop racing the natural end;
  - teams handed out at the very end were lost on some copies;
  - a player who left stayed the round's authority for about 35 s;
  - huge park files could stall the game.

  A second pass, verifying those fixes, found more gaps, now fixed:
  - leaving is now its own message, so it can't be taken for a stop while rosters differ;
  - every player whose round was stopped says the stop again after the end, so a stop sent once by a controller who then changed maps still reaches everyone (a new test failed before).
- **In game, the level change:** a round ended by loading another map stops on its own park.
- **In game, a tour of every spot (window pictures at each of the 20):** spot outlines were measured as the map loaded, while the skater was far away and no collision was loaded there, so almost every point fell back to the spot's lowest height and the lines lay under the ground. They are now measured as the skater nears each spot, filled in past walls, and evened out; pictures before and after: `domination/work/pictures/outlines-before-after.jpg`. A review of that change found three small things, fixed (stray lines beside the camera, dark notches at corners, the fill-in clock). Other players' colours are now never alike (they were picked from the id).
- **Fuzz test:** random rounds with random trouble (players dropping out of the session and back, leaving and rejoining, cut off a while, the round stopped early) found two ways copies could disagree, both fixed: a player cut off across the end came back to silence and kept another board or end (rounds over now keep saying their end, slowly, for 3 minutes, and late claims are always taken), and a late claim that was once its player's best changed the replay without the results being worked out again. 26 of 300 rounds disagreed before. Widening the fuzz (a second round soon after the first; the host dropping out) found two more, also fixed: a player cut off across a stop who came back still running the old round could pull everyone back into it just as the next round started (copies now never rejoin a round that ended for them), and the same claims heard in a different order could leave different (harmless) histories on the board (each player's history is now their bests in the order they were made). A review of those fixes found more, also fixed: leaving had started to mark a round as ended (a player who left, played alone and asked back in could not get back); late claims after the end made toasts and sounds; a lost "I left" message could leave the others counting the leaver as the round's authority; only the authority said the round runs, so a rejoin could miss it while the authority changed hands. Now, with MSVC, 0 of 11,000 seeds disagree.
- **Outlines, after the tester's look:** they were drawn over walls and popped in and out with distance. Now each nearby spot's edge is a faint glow, hidden behind walls and ramps (rays from the camera, with ReSkate's own `local_surface_hit`), fading in and out near the skater, and cut out exactly where the skater's body is on screen so they read as behind them; checked with picture tours.
- **Real tricks:** a tester's two recorded play sessions replayed against the 20-spot park (with the replay test in the developer tools, not in these patches): all 21 tricks count for a spot.
- **Code review of Custom Multiplayer:** fixed on the modes' page:
  - two games with the same name acted as one control;
  - joining now uses the server browser's own rule;
  - a failed join from the Multiplayer page covered this page;
  - refused host requests weren't explained;
  - smaller layout points.
- **Play counts (1):** a round with practice bots and a solo round counted (`domination stats`: 1 with others, 2 in all, Skate 3 Improved 2, Red-bowl park 2), and the counts were still there after a restart.
- **Server play stats (3):** unit tests (`server_stats`), and the server's other tests pass; built on `main` and `v2.1.0`, alone and with 1 and 2. A full build found two server tests that build the whole host without the new file; fixed, and again for `server_lock_tests`, new on `main` after `v2.1.0`.
- **Park-making and packs (1):** in a test copy, `domination tp` then `domination scan` hit all 961 points; `render_scan.py` drew it; `build_park.py` rebuilt Sunken banks from its spec to the shipped file exactly; `make_builtin.py` regenerates `domination_parks.h` byte for byte from `parks/`; the San Vansterdam pack, unzipped into `Mods`, loaded and its (edited) park replaced the built-in one; a pack switched off in the launcher's mod list loads nothing.
- **Earlier rebase, on `main` `364c619`** (two rebases since the first base; the last resolved two conflicts: test options added beside ours in `CMakeLists.txt`, and the Multiplayer page split into `hub_frame`, given its area): all three apply cleanly; a Hillside park round with 3 practice bots agreed on every copy, on Skate 3 Improved 1.1.3 (all 48 Skate 3 spots also checked on 1.1.3: the skater stands on each).
- **Live, two players:** played online between two PCs (two Steam accounts, ReSkate 2.1 built from these branches, a join-code lobby hosted from the DOMINATION tab): rounds ran for both players.
- **Rebased on `main` `99467f4`, after the official `v2.1.0` release** (conflicts: `road_rash` gained a field on the line next to Custom Multiplayer's in `client_tick.cpp`; server tool locks landed beside the play stats' lines in `server_host.cpp` and `server_commands.cpp`; main's lines kept, ours added): all three apply cleanly on `main` and on `v2.1.0` (2 with `git am -3` there); the suite, `fuzz 1000` and the practice-bot rounds above were run after the rebase, on both bases.
- **Not tested yet:**
  - server play stats on a live server with players;
  - the menu sounds on taking or losing a spot (the log can't show them; heard only in play);
  - finding games by mode in the tab's list with a second player (hosting from the tab and joining by code was played).

## The pull request texts

These are the commit messages of the three patches, without the `Co-Authored-By` line. The pull request descriptions (with pictures) are in [pr-1-domination.md](pr-1-domination.md), [pr-2-custom-multiplayer.md](pr-2-custom-multiplayer.md) and [pr-3-server-stats.md](pr-3-server-stats.md). A proposal for ReSkate-wide stats on `api.reskate.dev`, to open as an issue, is in [issue-stats-endpoint.md](issue-stats-endpoint.md).

### 1. Domination: a custom multiplayer mode

```
skate. 3's Domination: own the most spots in a park. The best single trick
landed on a spot owns it; a higher one steals it. Alone or with everyone in
the session; every round starts with every spot open. Played from the
console (`domination ...`), and from its own tab on the Multiplayer page when
Custom Multiplayer is in. ARCHITECTURE.md says where the game logic is and how to
change it safely.

- Rounds: `domination start` puts each skater at a park start facing the park
  (a different start per player), counts 3-2-1-GO (heard; GO the moment tricks
  count), then runs the clock; the session's host can start one from another
  map (everyone follows the host's map). Markers over the spots in their owner's
  colour (no two players alike), each nearby spot's edge a faint glow on the
  ground (hidden behind walls: rays from the camera), arrows at the screen's edge
  to the nearest spots you don't own, a timer, spot list, standings, and a feed
  of every trick and every spot changing hands (the spot flashes; a take or a
  loss is heard, with the game's own menu sounds); standing on a spot says whose
  it is and the score to beat.
- Results: the winner (the most spots; a tie goes to the higher total of
  points), standings (spots, time held, steals, points), the best trick, the
  most steals, the longest held, and who owns each spot.
- Parks: several, on any map. `domination parks` lists them with their map
  and whether that map is installed here (a map mod's Thunderstore package is
  named when it isn't, or when it is switched off); `domination set map
  next|previous` and `domination set park <id>|next|previous` pick the next
  round's (default: the red-bowl park when the Skate 3 map mod is installed). A
  park on another map: the round starts once that map has loaded. In a session
  the players on the map follow the park whoever runs the rounds picked.
- Settings (`domination set`): the map and park, the round's length, everyone or solo, and teams
  (red against blue, off by default): handed out as even as they can be, a late
  joiner to the smaller team, nobody moved once placed.
- The DOMINATION tab on the Multiplayer page (domination_menu.cpp), built when
  Custom Multiplayer is in the tree (cmake/Runtime.cmake sets
  DINGOSDK_CUSTOM_MODES): Map and Park dropdowns (every map with a park, marked
  when it isn't installed or is switched off; then that map's parks), round
  length, players, teams, start / stop / play alone / back into the round, and
  the standings while one runs. Without Custom Multiplayer, the console alone.
- Play counts: each round played to the end counts in the player's profile
  (ReSkate.Domination.Rounds, .Park.<id>, .Online, .Won, through
  profile_runtime::set_local_values); `domination stats` and the tab show them,
  in all, per map and per park. Nothing leaves the player's PC.
- Tricks are read from skate.'s ScoringHUDViewModel (ActionActive, LineScore,
  Multiplier, IsSlam) and credited to the spot the skater spent most of the trick
  on (else where it landed, else the spot left just before it); slams own nothing.
- Multiplayer (domination_sync.h, every rule apart from the game): the session's
  host runs the rounds (on a dedicated server, or with the host gone, the lowest
  Steam ID), and every player on the map is in each round at once, late arrivals
  too; `domination join` puts a player who left back in. Messages tagged 0xD0
  ride in the session's throwdown packets (relayed unread; no server, relay or
  protocol change). Every player sends the whole board it knows every 5 s, and
  after the end often for 10 s then now and then for 3 minutes (with the stop
  and the teams), so every copy ends with the same claims, owners, teams and
  results whatever order messages came in, a player cut off across the end
  included; clocks follow the round's
  authority; a stopped round ends at the stopper's moment on every copy; two
  rounds started at once merge; claims are checked; players on another
  Domination or park version are told why.
- Tested: Test/domination_net_tests.cpp runs up to 16 simulated players over a
  bad network (lag, reordering, loss, duplicates, clock offsets and drift, late
  joins, leaves and rejoins, the host leaving, two rounds at once, bad claims),
  and `fuzz <n>` plays random rounds with random trouble (11,000 agree). The
  rules and both test programs also build and pass with GCC 13.
- Park files: JSON with the level, the map's name and (for a map mod) its
  Thunderstore package, spots (polygon or capsule, height range, marker) and
  starts. Built in: five San Vansterdam parks where the game puts its own skate
  challenges (Rolling Waves 12 spots, Bonski Park under the bridge 8, Hartcore
  Plaza 12, FlowTopia 9, El-One Courts 5), and five on the Skate 3 map
  mod: the red-bowl park (20 spots), Hillside park (9), Industrial bowls (7),
  Sunken banks (6) and Loans plaza (6). Every spot checked in game. Maps can ship their
  own. tools/ holds the script that made the red-bowl park and the one that
  builds parks in.
- Code: the rules (domination_model, domination_sync, domination_wire) apart
  from the game side (domination.cpp: rounds; domination_files.cpp: parks and
  spot outlines; domination_hud.cpp: reading the score HUD;
  domination_overlay.cpp: drawing).
- teleport_local_skater_heading: a teleport facing a heading.
- Tests: DINGOSDK_BUILD_DOMINATION_TESTS (ctest -R domination).

Not yet tried between two real players.
```

### 2. Custom Multiplayer: ReSkate's own multiplayer modes as tabs on the Multiplayer page, hosting and finding games by mode

```
- Each mode gets a tab on the pause menu's Multiplayer page, after the page's
  own (SERVER BROWSER, HOST LOBBY, ...), which is otherwise unchanged: the
  mode's panel on the left, PLAY ONLINE on the right. Offline, where the
  Multiplayer page is left out, it shows when a mode is installed, with only
  the modes' tabs (a mode can be played alone).
- custom_modes::add: a mode registers its id, name, tagline, a panel (settings,
  status lines, buttons as console commands of its own) and an optional hook for
  joining a game advertised for it. Every mode's panel is drawn the same way.
- Dropdowns: a setting with `choices` (and the Map row, with `map_choices`) is a
  dropdown: pressed, it lists its choices under it, one full-width row each, so
  long names stay readable; picking one runs its console command.
- Hosting a game for a mode (the page, or `modes host <id> [public|code] [players]`):
  loads the mode's map if needed, hosts once it is in, and advertises the mode
  (Steam lobby rs_mode). Dedicated servers: `[server] mode = "<id>"`, the `g`
  browser tag. Nothing is advertised once the session ends; offline, hosting
  says multiplayer is off.
- Finding games: the browser's lobbies and servers advertising the chosen mode,
  joined from the page; the mode's `joined` hook runs once after joining.
- MultiplayerLobby::mode, valid_mode_id, multiplayer::set_advertised_mode.
  Older clients ignore rs_mode and the g tag.
- Tests: server tags and lobby data with a mode (multiplayer_protocol,
  multiplayer_lobbies).

No mode is in this change: the Multiplayer page looks as before until one
registers (README.md shows how).
```

### 3. Server: play stats (what has been played here, by mode and map)

```
A dedicated server now counts the games played on it, from the packets it
relays anyway (nothing it sends changes), and keeps the counts in
data/stats.json beside bans.json across restarts:

- Throwdowns: Jam, Spot Battle and S.K.A.T.E. (by the series in the
  leader's offer; "Throwdown" when the server never heard it), counted once
  at their start with the players in it; coop challenges at theirs.
- Domination rounds (when Domination is in the game): a round is counted
  the first time any of its messages is heard (they all start u8 0xD0,
  u8 version, u8 type, u32 park, u32 round), and each player the first
  time they send for it. Older versions' messages are not read.
- Per mode and map: games, and players summed over them.
- `stats` on the console (or from an admin) lists them; the server
  window's overview has a "Played Here" row ("Domination 12, Jam 4").
- Saved at most once a minute while games are played, and when the
  server stops; written beside and moved over. An unreadable file is
  left as it is and said in the log.

Nothing leaves the server. Tests: server_stats (throwdowns, rounds,
the file).
```

## Pictures

In [pictures/](pictures/), all from the test copies (offline, practice bots standing in for other players):

| File | Shows |
|---|---|
| `01-round-in-progress.jpg` | A round: timer, the spot list in owners' colours, standings, spot labels, and the feed of who did what (a banner each in the player's colour) |
| `02-steal-and-feed.jpg` | Kush steals a spot from you, and you steal it back: the feed's banners |
| `03-results.jpg` | Results (they come in one after another): the winner's band, standings with spots, time held, steals and points, award cards, who owns each spot |
| `04-teams-round.jpg`, `05-teams-results.jpg` | A teams round (red against blue) and its results |
| `06-parks-san-vansterdam.jpg`, `07-parks-skate3improved.jpg` | The ten built-in parks from above, spots and starts drawn on a height map |
| `08-outlines-before-after.jpg` | Spot outlines before and after line-of-sight hiding |
| `09-spot-check-hillside.jpg`, `10-spot-check-bonski.jpg` | Spot checks in game: a teleport to each spot with its outline and label |
| `11-multiplayer-tab.jpg`, `13-menu-map-dropdown.jpg` | The DOMINATION tab on the Multiplayer page, its Park and Map dropdowns open |
| `12-pack-icons.png` | The two park packs' icons, drawn by `make_pack.py` from the parks' spots |

## Branches (local, not pushed)

In the worktree `D:\reskate-dev\ReSkate-upstream`:

| Branch | Commit | What |
|---|---|---|
| `pr-domination` | 3d81821 | Pull request 1 (fork branch `domination`, #279) |
| `pr-custom-multiplayer` | 36e2c55 | Pull request 2 (fork branch `custom-multiplayer`, #278) |
| `pr-server-stats` | 76183e4 | Pull request 3 |
| `integration` | b8e17bd | 1 and 2 merged |
| `v210-domination`, `v210-custom-multiplayer`, `v210-server-stats`, `v210-all-three` | 2723791, dea9f90, 58362ac, 55e5a85 | The same three on the `v2.1.0` release, and all three together there (the check that they work on the release; the pull requests stay on `main`) |
| `test-build` | c31f30d | `integration` plus a bot-only launcher change that keeps the test copy offline and isolated, plus Domination's developer tools (practice bots, test claims, park-making commands, the score HUD logger and its replay test). Never submit it. |

The developer tools also sit on their own on `domination-devtools`, a test-only branch. They are not part of these patches and are never submitted.

