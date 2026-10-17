# Domination for ReSkate: notes, pictures and patches

Not code to merge: the material behind the pull requests for **Domination** and **Custom Multiplayer**. The fork's `domination-2.0.7` and `custom-multiplayer-2.0.7` branches hold the same two changes on `v2.0.7`.

- [AGENT-GUIDE.md](AGENT-GUIDE.md): for a maintainer's coding agent. How to apply each change, every line it adds to existing files, the conflicts that can happen, and how to check the result.
- [SUBMITTING.md](SUBMITTING.md): what was tested and how, the commit messages, and the branches.
- [pictures/](pictures/): the pictures in the pull requests.
- [patches/](patches/): each change as a `git am` patch against `main` (`364c619`), and the same against `v2.0.7` in `patches/for-2.0.7/`.
  - `1-domination/` and `2-custom-multiplayer/` are the two pull requests.
  - `3-server-stats/` (dedicated servers counting what is played on them) is ready but not opened as a pull request yet.
- [packs/](packs/): the ten built-in parks as two park-pack mods (`Domination_Parks_San_Vansterdam`, `Domination_Parks_Skate_3_Improved`), ready for Thunderstore. Made with `Extension/Domination/tools/make_pack.py`.
- `pr-*.md`: the pull request texts as written.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
