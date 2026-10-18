# Domination for ReSkate: notes, pictures and patches

Not code to merge: the material behind the pull requests for **Domination** and **Custom Multiplayer**, built and tested against the official ReSkate 2.1.0 release and `main`.

- [AGENT-GUIDE.md](AGENT-GUIDE.md): for a maintainer's coding agent. How to apply each change, every line it adds to existing files, the conflicts that can happen, and how to check the result.
- [SUBMITTING.md](SUBMITTING.md): what was tested and how, the commit messages, and the branches.
- [pictures/](pictures/): the pictures in the pull requests.
- [patches/](patches/): each change as a `git am` patch against `main` (`99467f4`, after `v2.1.0`); they apply to `v2.1.0` too (see [AGENT-GUIDE.md](AGENT-GUIDE.md)).
  - `1-domination/` and `2-custom-multiplayer/` are the two pull requests.
  - `3-server-stats/` (dedicated servers counting what is played on them) is ready but not opened as a pull request yet.
- [packs/](packs/): the ten built-in parks as two park-pack mods (`Domination_Parks_San_Vansterdam`, `Domination_Parks_Skate_3_Improved`), ready for Thunderstore. Made with `Extension/Domination/tools/make_pack.py`.
- `pr-*.md`: the pull request texts as written.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
