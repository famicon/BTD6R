# BTD6 → 2D (Rust + Bevy)

A 2D tower-defence remake of *Bloons TD 6*, written in Rust with Bevy. Map geometry,
round composition, bloon stats, tower names and **all artwork** are pulled straight out
of the retail Unity/IL2CPP install at `..\generic td` and converted into ordinary
`assets/` files. No Bloons art is re-drawn by hand.

```
cargo run --release
```

Requires Rust 1.99+ and a GPU with Vulkan/DX12. On first start the game builds its
data files from the shipped bundles (see *Extraction*); after that it just loads
`assets/data/*.json`.

---

## What is authentic and what is not

| Data | Source | Fidelity |
| --- | --- | --- |
| 90 maps, track splines, spawn points, splitter graph, bounds | `MapSet.json` / scene bundles | **Real**: every route point, splitter and area is copied verbatim |
| 140 campaign rounds (498 spawn groups, 14,724 bloons) | `RoundSet.json` → `DefaultRoundSet` | **Real** |
| 47 bloon types (Red…BAD, camo / regrow / fortified variants) | bloon class dumps | **Real** names, sprites, RBE, speed, immunities |
| 10 towers, 3 upgrade tiers each | `TowerSet` + upgrade dumps | Names, icons, prices, ranges and damage from the game's data; a few derived numbers are curated |
| Art | `*.bundle` texture atlases, 1,783 sprites | **Real**, converted to PNG at load time |
| Map preview / card art | `MapSelect` sprites | Real (used only as menu card art) |

### The one thing that is *not* decoded: `sim_model`

The gameplay tuning numbers (exact per-shot damage curves, attack cooldowns, pierce
tables, MOAB-class interactions) live in a single 17 MB Unity `TextAsset` called
`sim_model` inside the shared assets bundle. Its header is self-describing:

```
e5 67 05 00 | 09 00 00 00 | 78 08 00 00 (2168 class descriptors)
5 5 3 10 10 20 10 10 | c3 24 00 00 (9411) | then int32 tag/value token pairs
```

It contains 2,168 class descriptors and Pascal-style (1-byte-length) strings such as
`WeaponModel_Weapon`, `CircleFootprintModel_Circle Footprint` and `AttackModel_Attack_`.
The layout is a custom binary serializer, and the field **order** is not recorded in the
file, so decoding it requires either the IL2CPP metadata field ordering or
brute-force fitting. That work is not done; `tools/build_game_data.py` therefore ships a
curated table for the tower/upgrade numbers (marked "curated" in that file) and for the
few bloon properties that `sim_model` would otherwise provide.

Everything else in this repository — geometry, rounds, sprites, UI icons — is derived
mechanically from the install.

---

## Controls

| Input | Action |
| --- | --- |
| Mouse | Place tower (pick from the shop, click the field), select a tower |
| `Space` | Start the next round |
| `Tab` / `X` | Cycle game speed ×1 → ×2 → ×3 |
| `C` | Cycle targeting mode of the selected tower (First / Last / Close / Strong) |
| `U` / `S` | Upgrade / sell the selected tower |
| `Esc` / `R` | Back to the map menu |

---

## Project layout

```
src/
  main.rs      app setup, AppState (Menu / Playing), camera + UI scaling, enter_game
  data.rs      serde structs for the generated JSON, asset root discovery, loading
  game.rs      simulation: bloons, towers, projectiles, rounds, targeting, leaks, rings
  pathing.rs   arc-length parameterised track sampling, clearance queries
  theme.rs     per-theme palettes + procedurally rendered battlefield texture
  ui.rs        menu grid, HUD, shop, detail/upgrade panel, geometric hit-testing
  capture.rs   optional scripted screenshot harness (BTD62R_CAPTURE=<dir>)

tools/         Python extraction pipeline (UnityPy) — see below
extracted/     intermediate dumps: data/, maps/, sprites/, raw/
assets/        what the game actually loads (data JSON + PNG sprites)
```

### Coordinates

Every map is normalised so its longest side is `MAP_WIDTH = 1000` world units and
centred on the origin. Track half-width is 15 units, base bloon speed 60 units/s.
The camera always shows 1250 units vertically. The battlefield texture (1400²) is drawn
from the real splines, so track and path always agree by construction; world→image is
`uv = (500 ± v) / WORLD_SPAN`.

UI is authored in a fixed 1600×900 virtual space and scaled with `bevy::ui::UiScale`,
so every layout rect is a pure function of the viewport. Clicks are hit-tested
geometrically against those rects instead of relying on the picking backend.

---

## Extraction pipeline

All scripts run inside the local `.venv` (Python + `UnityPy`) and read the bundles in
`..\generic td\generic td_Data\StreamingAssets\aa\StandaloneWindows64\{Full,Half}`.
They are idempotent and are only needed when regenerating data.

| Script | Purpose |
| --- | --- |
| `scan_bundle.py` | Walk a bundle and list object types |
| `extract_data.py` | Dump ScriptableObjects as JSON; builds the `path_id → class` resolver (11,658 entries) → `extracted/data/*.json` |
| `extract_maps.py` | Parse `MapSet` + map scenes → `extracted/maps/*.json` (routes, splitters, areas, bounds) |
| `export_sprites.py` | Cut 1,783 sprites out of atlases → `extracted/sprites/` + `sprites_manifest.json` |
| `inventory.py` | Full asset census (131k MonoBehaviours, 10k textures, 45 TextAssets) |
| `build_game_data.py` | Merge everything into `assets/data/{bloons,rounds,towers,maps,loc}.json` and copy the art the game loads |

Notes that cost time to discover:

* The bundles ship IL2CPP **type trees**, so `MonoBehaviour` fields can be read
  semantically without the metadata file.
* The only text assets in the whole build are `sim_model`, 22 localisation files
  (`loc_English.bin` …) and `atlas_manifest`.
* Map names come from the localisation `<Maps>` section; loc ids do not match display
  names (`Tutorial` is "Monkey Meadow").
* Map ids are sanitised to lowercase alphanumerics (`#ouch`/`-ouch` → `ouch`), and
  sprite file names have non-`[A-Za-z0-9 -_.[]()]` characters replaced by `_`.
* There is no plain `Ddt` sprite in the build; the `DdtIcon` is used instead.
* Fitting the official map *previews* onto the track geometry does not work — they are
  stylised marketing renders (Bazaar's art barely shows the track). The battlefield is
  therefore rendered procedurally from the splines, in the palette of the map's theme.

---

## Build notes

* `bevy = 0.20`, edition 2021, MSVC toolchain.
* `.cargo/config.toml` redirects `target-dir` to
  `%TEMP%\opencode\btd62r-target`, because OneDrive locks Bevy's incremental files
  (`Access is denied`, os error 5) inside the synced Desktop folder.
* `src/capture.rs` is inert unless `BTD62R_CAPTURE` is set:

  ```powershell
  $env:BTD62R_CAPTURE="$env:TEMP\shots"; cargo run
  ```

  It renders the menu, the battlefield and two combat moments, then exits — used to
  verify the game visually without a human at the keyboard.

---

## Licensing

The code here is a fan reimplementation. Every image, name and piece of game data
remains the property of Ninja Kiwi; this repository contains no original art and is not
distributed as a standalone game. Point the pipeline at a copy of the game you own.
