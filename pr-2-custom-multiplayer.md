# Custom Multiplayer: ReSkate's own modes as tabs on the Multiplayer page

A way for ReSkate's own game modes to plug into the pause menu. Each mode gets a tab on the Multiplayer page, after the page's own tabs (SERVER BROWSER, HOST LOBBY, ...), which are otherwise unchanged.

![The DOMINATION tab on the Multiplayer page](pictures/11-multiplayer-tab.jpg)

## What's in it

- **`custom_modes::add`:** a mode registers an id, name and tagline, plus a panel built from plain data:
  - settings, adjust rows and **dropdowns** (whole names, one full-width row per choice);
  - status lines;
  - buttons, each running one of the mode's own console commands.

  Every mode is drawn the same way.
- **Hosting a game for a mode,** from the tab or with `modes host <id> [public|code] [players]`:
  - loads the mode's map first if needed;
  - advertises the mode in the Steam lobby (`rs_mode`).
- **Dedicated servers** advertise a mode with `[server] mode = "<id>"`, which becomes the `g` server-browser tag. The mode itself runs on the players' games, so servers need nothing else.
- **Finding games:** the tab lists the lobbies and servers that advertise the mode, and joins them.
- **Offline:** the Multiplayer page shows only when a mode is installed, and then only the modes' tabs, since a mode can be played alone.
- **Older clients** ignore `rs_mode` and the `g` tag.

No mode is in this change. Until one registers, the Multiplayer page looks as before. `Extension/CustomModes/README.md` shows how a mode plugs in.

## Tested

- **Unit tests:** server tags and lobby data with a mode (`multiplayer_protocol`, `multiplayer_lobbies`).
- **Tab order:** the pause menu's tab order is checked (`native_menu_browser`).
- **In game:** with Domination plugged in (#1), offline.
- **Live:** a Domination game hosted from the tab (join code only) on one PC, joined from another with its code; the round ran for both. Finding games in the tab's list hasn't been tried with a second player yet.

## For reviewers and agents

- **Independent:** this works alone, and merges cleanly with "Domination" (#1) in either order.
- **On 2.0.7 too:** the same change against `v2.0.7` is on the notes branch (`patches/for-2.0.7/`), with a ready branch on the fork (`custom-multiplayer-2.0.7`).
- **Applying:** `AGENT-GUIDE.md` lists every line this adds to existing files.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
