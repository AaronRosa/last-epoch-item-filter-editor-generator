# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Setup
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt

# Run the UI editor (primary usage)
.venv/bin/python -m lefilter ui

# CLI subcommands
.venv/bin/python -m lefilter extract            # re-read game data into data/uniques.json
.venv/bin/python -m lefilter build              # generate filter XML from config.toml + template
.venv/bin/python -m lefilter build --install    # also copy the filter into the game's Filters folder
.venv/bin/python -m lefilter stats              # unique counts per category and LPL range
.venv/bin/python -m lefilter leveling           # preview what the [leveling] config section generates

# Run all tests
.venv/bin/python -m pytest

# Run a single test file
.venv/bin/python -m pytest tests/test_filterxml.py

# Translation tooling
.venv/bin/python packaging/translate_ui.py check     # find missing/broken translations
.venv/bin/python packaging/translate_ui.py keys -v   # list all translatable strings
```

Pass `--filters-dir <folder>` (before the subcommand) to use a folder other than the game's Filters folder — useful when running browser tests.

## Architecture

The codebase is a Python package (`lefilter/`) with a single-page web front end (`lefilter/web/`). There is no build step for the JS/CSS; the files are served directly.

### Data flow

1. **`extract.py`** reads Unity asset bundles from the user's Last Epoch install (via UnityPy + Cpp2IL + TypeTreeGeneratorAPI) and writes `data/uniques.json`, `data/lang/`, and `data/filter_icons/`. Nothing from the game ships with the repo.
2. **`rules.py`** reads `config.toml` (TOML, with `tomllib`) and the extracted data and produces `Rule` / `RuleSpec` dataclass instances — one per filter rule, in in-game top-first order.
3. **`filterxml.py`** serialises `Rule` objects to the game's XML format (lootFilterVersion 9, UTF-8 with BOM, CRLF). Rules are stored bottom-up in the file but this module exposes them top-first. `render_rule()` → opaque `str` block; `assemble()` + `write_filter()` → final file.
4. **`filterdoc.py`** is the *editor's* round-trip layer: it fully decodes every rule and condition into JSON-friendly dicts so the browser can display and edit them. `filterxml.py` keeps unknown rules as opaque blobs; `filterdoc.py` also keeps them as `{"raw": ...}`.
5. **`ui.py`** serves `lefilter/web/` on `http://127.0.0.1:8765` (stdlib `http.server` only — no framework) and exposes a JSON API the browser calls. `Api` holds all mutable state; filters are read from the game's Filters folder and `out/`.

### Key modules

| Module | Role |
|---|---|
| `gamedata.py` | Enum constants mirroring the game's IL2CPP metadata (equipment types, rarities, classes, etc.) |
| `rules.py` | Config parsing, `Rule`/`RuleSpec` types, unique-group planning, affix-rule and class-hide-rule generation |
| `filterxml.py` | Serialise rules → XML; read + merge base filters (opaque rule blocks) |
| `filterdoc.py` | Full parse/render of every rule and condition for the editor; must round-trip byte-for-byte |
| `leveling.py` | Leveling-section generator: weapon batches, armour/jewelry rules, tier stepping, endgame rares |
| `bis.py` | Best-in-slot section generator |
| `idols.py` | Idol and idol-altar rule generator |
| `sections.py` | Filter-level operations: place/reorder generated sections, find section boundaries |
| `starter.py` | New-filter template creation, three-way merge template updates, `add_missing_sections`, `refresh_generated` |
| `matcher.py` | Item-tester: evaluates filter rules against a described item, following the game's matching logic |
| `extract.py` | Game data extraction via UnityPy; schema recovery via Cpp2IL + TypeTreeGeneratorAPI |
| `game.py` | Find the Last Epoch install and its Filters folder via Steam library files |
| `paths.py` | `USER_DIR`, `DATA_DIR`, `CACHE_DIR`, `OUT_DIR`, `TEMPLATE_FILE`, `CONFIG_FILE` — differ between source checkout and PyInstaller bundle (`FROZEN`) |
| `i18n.py` | Editor i18n: loads `lefilter/web/i18n/<lang>.json`; `UI_GAME_TERMS` marks strings that come from the game install, not the JSON files |
| `affixtext.py` | Formats affix values as the game displays them (percent, rounding, plus-sign, hidden values) |

### File paths (source checkout vs packaged app)

`paths.py` sets `USER_DIR = SOURCE_DIR` when running from source, so `data/`, `.cache/`, `out/`, `templates/`, and `config.toml` all live in the project root. In the PyInstaller bundle, `USER_DIR` is `%APPDATA%/Last Epoch Item Filter Editor` (Windows) or `~/.local/share/last-epoch-item-filter-editor` (Linux).

### Filter XML conventions

- Rules are stored **bottom-up** in the XML; `<Order> 0` is the top rule. All Python code uses top-first ordering and `filterxml`/`filterdoc` reverse internally.
- The file is UTF-8 **with BOM** (`\ufeff`), CRLF line endings, no trailing newline.
- The game allows at most 200 rules (`MAX_RULES = 200`).
- Generated rules are prefixed `[A] ` (uniques/sets/class-hide/affix), `[L] ` (leveling), `[E] ` (endgame rares), `[I] ` (idols), `BIS - ` (best-in-slot).

### Web front end

`lefilter/web/` contains `index.html`, `app.js`, and `app.css` — no build step. The JS calls the Python server's JSON API endpoints. Translations live in `lefilter/web/i18n/<lang>.json`; the English text is the key.

### Releases

GitHub Actions (`.github/workflows/release.yml`) builds Windows and Linux executables with PyInstaller when a `v*` tag is pushed.
