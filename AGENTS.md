# AGENTS.md

How to work in this repository. Read README.md for what decompdecomp does from a user's point of view.

## What this is

decompdecomp: AI agents decompile games the user owns into C that compiles, with the game's original compiler, to
the exact same bytes. Python 3.12+, managed with uv. A continuation of decomp.wtf (Sam Wolfs, i12bp8/decomp.wtf,
archived at 0.10.6): `origin` is n-delic/decompdecomp, `upstream` the archived original.

Naming: the project is **decompdecomp** in everything users read. Internal names stay as decomp.wtf left them, so
existing installs and projects keep working: the `decomp` command, the `decompnet` package and data folder,
`~/Decomp`, `.decomp.wtf.json` manifests, `.decomp-wtf/` state, the `decomp-wtf` git branch, WSL distribution and
services. Installs and updates come from GitHub (`REPO` in app/updates.py and the installers); the decomp.wtf domain
isn't ours, so nothing may point at it. `site/` is the website (decomp.ndelic.com, published to GitHub Pages by
`.github/workflows/site.yml` on every push to main that changes `site/public/`); `site/public/` also holds the
installers, which updates read from GitHub. Work on `main`: it's what users update from.

## Commands

```bash
uv sync
uv run decomp                 # the app (GUI in the browser); projects live in ~/Decomp or $DECOMP_HOME
uv run decomp mcp             # the MCP server agents connect to
uv run pytest -q              # the test suite; many tests skip when a compiler or tool isn't installed
uv run pytest tests/test_forks.py -q   # one file
```

## Never disturb the user's own decompdecomp

The user usually has decompdecomp running (installed with uv, port 6464, projects in `~/Decomp`) with agents working
in it. Testing a change must not touch it:
- Never run the checkout's `decomp` or `decomp mcp` against `~/Decomp`. A newer version that finds an older app
  running quits it and takes its place (cli.py), stopping its jobs. Give it a throwaway home:
  `DECOMP_HOME=<scratch dir> BROWSER=true DECOMP_TRAY=0 uv run decomp ...`.
- Tests already run with a throwaway `DECOMP_HOME` (tests/conftest.py) and stop the apps they start. When the user's
  app runs from this checkout's `.venv`, run them in another environment so uv doesn't rebuild it underneath:
  `UV_PROJECT_ENVIRONMENT=<scratch venv> uv run pytest -q`.
- To try a change on a real project, work on a copy: `sqlite3 "file:$HOME/Decomp/.decomp/decomp.db?mode=ro"
  ".backup <copy>/.decomp/decomp.db"`, copy the project folder, point `projects.root` in the copy at it. Read the
  live database only with `?mode=ro`.
- Don't restart, update or quit the user's app, and don't change its projects, without asking.

Compilers and tools (splat, objdiff, dtk, wibo, Ghidra, period compilers) are pinned with checksums in
`src/decompnet/toolchains.toml` and installed by `src/decompnet/toolchains.py`, never taken from PATH. Proprietary
compilers are only installed after the user accepts that in Setup; don't add code that fetches them silently.

## Two ways a game is decompiled

1. **ROM mode** (N64, PS1, PS2, GBA): `src/decompnet/rom/` splits the game (splat/spimdisasm, luvdis), and
   `rom/match.py` (MIPS) / `rom/match_arm.py` (ARM) *link* each compiled candidate at the function's real address
   and compare bytes. Driven by `app/engine.py`. Exported as a rebuildable tree by `rom/export*.py`.
2. **Fork mode** (public decomps forked by repository address, and GameCube/Wii/DS/PC started from scratch): a community-shaped
   project on disk with an `objdiff.json`; objdiff judges each unit. `forks/project.py` detects and builds projects,
   `app/forkengine.py` drives them. From-scratch importers generate such a project:
   `forks/scratch_dtk.py` (GameCube/Wii, dtk-template), `forks/scratch_dsd.py` (DS, ds-decomp, template in
   `forks/ds_template/`) and `forks/scratch_pe.py` (Windows PC: Ghidra finds functions, decompdecomp delinks them
   into COFF objects itself, template in `forks/pc_template/`). A new from-scratch platform that has a
   delinker/objdiff story belongs here. Fork-mode helpers that read objects with pyelftools must tolerate COFF
   (PC, Xbox): treat an `ELFError` as "no verdict".

Shared pieces: `network.py` (claims, attempts, notes, shared header), `compiler.py` (sandboxed compiler runs),
`scoring.py` (capstone disassembly and diffs for every arch), `elfobj.py` (ELF objects), `classes.py` (C++ classes
from the vtables constructors store, MIPS: the `get_class` tool and the class lines of `get_function`),
`compilermap.py` (which compiler built each part of a MIPS game; the judge's survey is `engine.survey_compilers`;
`app/rangecheck.py` tests a suggested range on its neighbours before it applies), `app/tools.py` (the MCP
tools and the agent prompt), `app/window.py` + `app/tray.py` (the app's own window and its tray icon; tests set
`BROWSER=true` and `DECOMP_TRAY=0` so they open neither), `app/guide/*.md` (know-how agents read through `guide(topic)`), `app/static/` (GUI).

## Conventions

- Match the surrounding code: plain-English docstrings and comments that say *why*, often citing the game or issue
  that motivated a rule ("#22: ..."). No new dependencies for a few lines; capstone, pyelftools, httpx are there.
- Heavy external tools run through `limits.capped(...)` (memory cap) with a timeout.
- Errors shown to the user or agent are `ForkError` / `NetworkError` with a sentence they can act on.
- Every behavior change gets a test in `tests/`. Tests that need a real game or a proprietary compiler skip when it
  isn't there; small homemade games/objects live in `projects/demo-*` and are built by their own scripts.
- Releases bump the version in five places together: `pyproject.toml`, `src/decompnet/__init__.py`,
  `site/public/install.sh`, `site/public/install.ps1`, `uv.lock`. Commit subjects read
  `decompdecomp X.Y.Z: <what users get, in one sentence>`, with a body explaining why. Pre-releases are
  `X.Y.Z-alpha` (`-beta`, `-rc`); `updates.version_key` orders them before X.Y.Z.
- A release is a `vX.Y.Z` tag on main. `.github/workflows/release.yml` builds the `.deb` and the AppImage
  (`packaging/linux/build.sh`, which also runs locally) and attaches them to the GitHub release; it can be run by hand
  for a pushed tag (`gh workflow run release -f tag=vX.Y.Z`). Packaged copies set `DECOMP_PACKAGE` and don't update
  themselves.
- What's planned is in ROADMAP.md, with a table of which feature works on which platform: when a feature ships or
  reaches another platform, update the table and move its item to done.

## Never

- Commit, download or publish game data (ROMs, discs, executables, extracted code or assets). Generated projects
  keep them out of git (`orig/`, `extract/`, `build/`).
- Let agents edit a project's build setup: `forks/project.py` (`EDITABLE`, `FORBIDDEN_PARTS`) limits them to
  source and header files on purpose.
- Weaken the judge: a match means byte-identical after linking (ROM mode) or objdiff 100% (fork mode).
