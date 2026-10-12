# SD2 Mod Studio's own file formats

The files SD2 Mod Studio reads and writes to pass mods, units and divisions between people. They are documented
here so that other tools can read and write them. The field lists come from the app's C# classes (format version 1
of each); the classes are the reference.

| Extension | What it holds | Container | Main entry | JSON names |
|---|---|---|---|---|
| `.sd2units` | a unit pack: new units with their game data, models and names | zip | `pack.json` | camelCase |
| `.sd2unit` | one unit, built for a pack | zip | `unit.json` | camelCase |
| `.sd2pkg` | a share package: a mod's **source** (optionally with a play-only generated copy) | zip | `manifest.json` | PascalCase |
| `.sd2mod` | a **generated** mod, ready to play | zip | `manifest.json` | PascalCase |
| `.sd2div` | a division with its battlegroups | one JSON file | | PascalCase |
| `.sd2bg` | one battlegroup | one JSON file | | PascalCase |

## Common rules

- **JSON.** UTF-8. The app serialises with .NET's `System.Text.Json`: GUIDs as `"xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"`,
  times as ISO 8601 (`"2026-10-12T09:30:00Z"`), `null` written out for empty optional values.
  - **camelCase** files (`pack.json`, `unit.json`) are read case-insensitively.
  - **PascalCase** files are read **case-sensitively**: write the names exactly as listed (`PackageId`, not
    `packageId`). Unknown properties are ignored.
- **Zip.** Standard zip (deflate). Paths inside use `/`. A path must be relative, with no empty, `.` or `..`
  segments, no `\` and no `:`.
- **File lists and hashes.** Where a manifest has a file list, every other entry in the zip must be listed, with
  its exact uncompressed `Size` and its SHA-256 as lower-case hex (`Sha256`); readers compare case-insensitively.
  Extra, missing or mismatched files make the whole file invalid.
- **Untrusted input.** Readers check sizes while decompressing (zip entries can lie), refuse unknown file types, and
  never write outside the target folder. The limits below are the app's.
- **`Format`** is 1 for every format here. A reader must refuse other values.

## Unit pack: `.sd2units`

A collection of new units released together. It is installed into the app (not into a mod); when a division using
its units is added to a mod, the app copies those units into the mod. A pack never carries the game's own units or
any divisions.

### Zip entries

| Entry | Content |
|---|---|
| `pack.json` | the manifest (below) |
| `README.md` | what the pack is (optional) |
| `CREDITS.md` | model sources, authors and licences (optional) |
| `assets/<mod path>` | model, texture and picture files, at the path they take in a mod, e.g. `assets/GameData/Assets/3D/Units/XX/Char/Example/Example.Ase2NdfBin`, `assets/AssetsModding/Units/Example_XX.png` |

Assets must lie under `GameData/Assets/` or `AssetsModding/` and be `.ase2ndfbin`, `.png`, `.dds` or `.tga`
(case-insensitive). Limits: `pack.json` ≤ 32 MB, each file ≤ 512 MB, all files ≤ 4 GB, ≤ 20,000 files.

### `pack.json`

| Field | Type | Notes |
|---|---|---|
| `format` | int | 1 |
| `kind` | string | `"unitpack"` |
| `packId` | GUID | the pack's identity; never changes between versions; not the empty GUID |
| `name` | string | 1–80 characters |
| `version` | int | ≥ 1; +1 for every change |
| `displayVersion` | string | shown to people, e.g. `"1.2"` |
| `author`, `description` | string | |
| `createdUtc` | time | |
| `gameDataRevision` | int or null | game data revision the pack was built against |
| `appVersion` | string | |
| `localOnly` | bool | holds models whose licence forbids wider sharing: never put on the Workshop |
| `units` | list of unit | below |
| `fragments` | object: key → fragment | each piece of game data once, shared by the units that list its key |
| `localisation` | object: token → text | names for `MODDING.csv`; tokens 1–10 characters of `A–Z a–z 0–9 _` |
| `files` | list of `{ path, size, sha256 }` | every zip entry except `pack.json` |

**Unit** (`units[]`):

| Field | Type | Notes |
|---|---|---|
| `name` | string | `[A-Za-z0-9_]{2,80}`; the game declaration is `Descriptor_Unit_<name>`; unique in the pack |
| `display` | string | display name |
| `side` | string | e.g. `"Allied"`, `"Axis"` |
| `nation` | string | country code |
| `deckId` | int | > 0, unique in the pack: the unit's deck-code id, the same in every mod |
| `fragments` | list of string | keys into the pack's `fragments` |
| `assets` | list of string | mod-relative paths (without `assets/`) of the unit's files; each must be in `files` |
| `card` | string or null | the unit list picture's mod path |
| `localisation` | list of string | tokens (keys of the pack's `localisation`) the unit uses |
| `licence` | `{ source, licence, credit }` | where the model comes from and its licence |

**Fragment** (`fragments.<key>`): a piece of game data a unit adds to one NDF file of a mod.

| Field | Type | Notes |
|---|---|---|
| `kind` | string | `append`, `insertAfter`, `beforeLast` or `newFile` |
| `file` | string | mod-relative path under `GameData/`, ending `.ndf` |
| `text` | string | the NDF text to add; must not contain `SD2ModStudio:` |
| `anchor` | string or null | required for `insertAfter` and `beforeLast` |

| Kind | What a tool does to the mod's `file` |
|---|---|
| `append` | adds `text` at the end, between block markers |
| `insertAfter` | adds `text` as a new line after the line holding the first occurrence of `anchor` (a line of the game's own data, e.g. a list entry next to which the unit belongs) |
| `beforeLast` | adds `text` as a new line before the last line that begins with `anchor` (e.g. a list's closing bracket line) |
| `newFile` | writes `file` with `text` (only for files the game does not have) |

Keys are free text; the tools use a `type:name` style such as `unit:Example_XX`, `ammo:Ammo_Example`,
`file:<path>`. Each applied piece is marked `SD2ModStudio:Pack:<packId as 32 hex digits>:<key>` so it can be
found and removed again. The unit's `deckId` is added to
`UnitIds` in the mod's `DeckSerializer.ndf` with its own marker.

**Checks a reader should make** (the app's): file list and hashes; asset paths and types; unique unit names and
deck ids; every fragment key and asset a unit lists exists; fragment kinds, files and anchors; `append` and
`newFile` texts parse as NDF; against the game data: no unit or `export` name the game already defines, `newFile`
only for files the game lacks, and every anchor present.

### Where the app keeps installed packs

`%LOCALAPPDATA%\SD2ModStudio\packs\<packId as 32 hex digits>\`: `pack.json`, `README.md`, `CREDITS.md`, `assets/…`, plus the app's
`CHANGES.txt` (one line per change: date, version, what changed), a cached unit list, and the previous version in
a folder of the same name plus `.previous` next to it.

## Unit package: `.sd2unit`

One unit as the unit-building tools produce it, to be imported into an installed pack (adding the unit or replacing
the pack's copy). Same checks as a pack, applied as if it were a pack holding just this unit.

| Entry | Content |
|---|---|
| `unit.json` | the manifest (≤ 8 MB) |
| `assets/<mod path>` | the unit's files, as in a pack |

`unit.json`:

| Field | Type | Notes |
|---|---|---|
| `format` | int | 1 |
| `kind` | string | `"unit"` |
| `packId` | GUID | the pack it was built for (a hint; it can go into any pack) |
| `packName` | string | |
| `createdUtc` | time | |
| `notes` | string | what changed in this build, shown before importing |
| `unit` | unit | as in `pack.json` |
| `fragments` | object: key → fragment | **exactly** the pieces the unit lists, no others |
| `localisation` | object: token → text | |
| `files` | list of `{ path, size, sha256 }` | |

## Share package: `.sd2pkg`

A mod's source, for a friend who also uses SD2 Mod Studio, so they can play it, change it and send it back. Only
the files that differ from the game's mod template are carried. Version 1 of the app writes and reads **only
whole mods** (`Kind = "mod"`).

### Zip entries

| Entry | Content |
|---|---|
| `manifest.json` | the manifest (≤ 8 MB); not in its own `Files` list |
| `CHANGES.md` | the sender's notes (UTF-8; listed in `Files`) |
| `project/<mod path>` | the mod's changed source files |
| `generated/<path>` | optional play-only copy of the generated mod (as in `.sd2mod`) |

- `project/` paths must start with `GameData/`, `CommonData/`, `AssetsModding/` or `SD2ModStudio/`, and have one of
  the types `.ndf .csv .json .md .txt .ini .dic .png .dds .tga .jpg .jpeg .ase2ndfbin`. The app's own
  bookkeeping is never included: `SD2ModStudio/packages.json`, `SD2ModStudio/last-generate.json`,
  `SD2ModStudio/backup/…`, `*.tmp`.
- `generated/` paths may have the types `.ndfbin .ini .dic .csv .png .jpg .jpeg .bin`; if there are any, one
  must be `Config.ini`.
- Limits: each file ≤ 512 MB, all ≤ 2 GB, ≤ 20,000 files. `.ndf` files up to 64 MB are parsed on import; a parse
  failure is shown as a warning, not an error.

### `manifest.json`

| Field | Type | Notes |
|---|---|---|
| `Format` | int | 1 |
| `PackageId` | GUID | stays the same for every version of the same mod; not the empty GUID |
| `Kind` | string | `"mod"` |
| `Name` | string | the mod's folder name: `[A-Za-z0-9_-]`, ≤ 64 characters, not starting with `-`, not a reserved name (`ModData`, `Utils`, `Backup`, `DeckBuilder`, Windows device names) |
| `Version` | int | ≥ 1; +1 for every export |
| `DisplayVersion` | string | e.g. `"1.3"`; defaults to `Version` |
| `BasedOnVersion` | int | the version this one was edited from (0 for the first) |
| `Author` | string | |
| `CreatedUtc` | time | |
| `GameDataRevision` | int or null | |
| `AppVersion` | string | |
| `ModId` | int64 | the mod's `ID`, so both players' generated copies match |
| `HasPlayCopy` | bool | `generated/` entries present |
| `Files` | list of `{ Path, Size, Sha256 }` | every zip entry except `manifest.json`, with its full zip path (`project/…`, `generated/…`, `CHANGES.md`) |

### Import decisions

The receiving app compares with the record of the installed copy (`SD2ModStudio\packages.json` in the mod):

| Situation | Result |
|---|---|
| no installed copy with this `PackageId` | install (or, if another mod has the name, ask: copy under another name, or replace) |
| same files (all `project/` hashes equal) | already have it |
| `Version` newer and `BasedOnVersion` ≥ installed version | update, unless the installed copy was changed locally since its record, then ask (fork) |
| `Version` older | older: do nothing unless asked |
| otherwise | fork: ask (replace mine / keep mine / import as a copy) |

On install, files the mod had changed that the package does not carry go back to the template's version; every
replaced file is first copied to `SD2ModStudio\backup\last-import\<time>\`. A copy gets a new `PackageId` and a new
local mod ID.

### `SD2ModStudio\packages.json` (in the mod)

| Field | Type | Notes |
|---|---|---|
| `PackageId` | GUID | |
| `Version` | int | |
| `DisplayVersion` | string | |
| `Author` | string | |
| `How` | string | `"exported"` or `"imported"` |
| `RecordedUtc` | time | |
| `Files` | object: mod path → SHA-256 | the `project/` files the package held, without the `project/` prefix |

## Generated mod: `.sd2mod`

A generated mod (the `Saved Games\…\mod\<Name>` folder) for someone who only wants to play it. Installing puts it
at `Saved Games\EugenSystems\SteelDivision2\mod\<Name>`; an existing folder is moved to a backup first.

| Entry | Content |
|---|---|
| `manifest.json` | below (≤ 64 KB) |
| `mod/<path>` | the generated files; types `.ndfbin .ini .dic .csv .png .jpg .jpeg .bin`; must include `mod/Config.ini` |

Limits: each file ≤ 512 MB, all ≤ 1 GB, ≤ 5,000 files. There are no hashes in this format.

`manifest.json`:

| Field | Type | Notes |
|---|---|---|
| `Format` | int | 1 |
| `Name` | string | the folder name, same rules as in `.sd2pkg` |
| `Id` | int64 | the `ID` from `Config.ini` |
| `ModGenVersion` | int | from `Config.ini` (0 if absent); the app warns when it differs from the receiver's game data revision |
| `ExportedUtc` | time | |
| `ExportedBy` | string | e.g. `"SD2 Mod Studio 0.6.0"` |
| `FileCount` | int | |
| `TotalBytes` | int64 | |

## Division: `.sd2div` and battlegroup: `.sd2bg`

Plain JSON files (not zips), at most 16 MB, nesting depth ≤ 32. Both use one wrapper:

| Field | Type | Notes |
|---|---|---|
| `Format` | int | 1 |
| `Kind` | string | `"division"` (`.sd2div`) or `"battlegroup"` (`.sd2bg`) |
| `AppVersion` | string | |
| `ExportedUtc` | time | |
| `Division` | division or null | set in `.sd2div`, with its battlegroups; `DisplayName` must not be blank |
| `Battlegroup` | battlegroup or null | set in `.sd2bg` |
| `DivisionKey`, `DivisionName` | string or null | `.sd2bg`: the division it was built from, to suggest where it goes |

Importing a division always gives it new identities (new key, GUIDs and text tokens; deck-code ids cleared), so an
import never collides with what is there. A battlegroup is imported into a division the user picks (the app
suggests `DivisionKey`); it gets a key unique in that division, and card lines for units that division lacks are
left out.

### Division

The same object is the app's division library file (`%LOCALAPPDATA%\SD2ModStudio\divisions\<Key>.json`) and the
copy kept in a mod (`SD2ModStudio\divisions\<Key>.json`, which also holds that mod's deck-code ids).

| Field | Type | Notes |
|---|---|---|
| `Version` | int | 1 (format of the division object) |
| `Key` | string | **required**. Used in game names (`Descriptor_Deck_Division_<Key>` …). The app makes it from the name: ASCII letters, digits and `_`, starting with a letter, ≤ 48 characters |
| `DisplayName`, `Description` | string | |
| `Alliance` | string | `"Allied"` or `"Axis"` |
| `CountryId` | string | e.g. `"US"` |
| `TemplateDivisionId` | string | **required**. An existing division whose emblem, phases, income and textures are reused |
| `MaxActivationPoints` | int | |
| `SlotCosts` | object: category → list of int | activation-point cost of each successive slot; list length = number of slots (at most 19 are shown in the game). Categories: `reco`, `infanterie`, `tank`, `support`, `at`, `dca`, `art`, `air`, `defense` |
| `Units` | list of unit entry | below |
| `TransportLimits` | object: transport unit → int | how many of that transport a battlegroup may hold |
| `Battlegroups` | list of battlegroup | below |
| `Incomes` | list of `{ Name, Values: [A, B, C] }` | the division's own income options (command points per phase, each 0–5000) |
| `Guid`, `HintGuid` | string (GUID) | the division's GUIDs in the game data |
| `NameToken`, `DescriptionToken` | string | localisation tokens (10 letters) |
| `SerializerId` | int or null | deck-code id of the division (set when written into a mod) |
| `ModifiedUtc` | time | |
| `DivisionId`, `EconomyListName`, `CostMatrixName` | string | written for information (derived from `Key`); ignored when read |

**Unit entry** (`Units[]`):

| Field | Type | Notes |
|---|---|---|
| `UnitId` | string | **required**. The unit's export name, `Descriptor_Unit_…` |
| `Cards` | int | number of cards of this unit |
| `UnitsPerCardByPhase` | [int, int, int] | units per card in phases A, B, C |
| `XpMultipliers` | [number, number, number] | units-per-card multiplier for experience levels 0–2; 0 = level not offered |
| `AvailableWithoutTransport` | bool | |
| `Transports` | list of string | transport unit export names allowed |
| `PackGuid` | string (GUID) | the card's GUID in the game data; kept so writing again is stable |
| `PackSerializerId` | int or null | deck-code id of the card (set when written into a mod) |

**Battlegroup** (`Battlegroups[]`, or `Battlegroup` in `.sd2bg`):

| Field | Type | Notes |
|---|---|---|
| `Key` | string | **required** |
| `Name` | string | |
| `DeploymentType` | int | index into the template division's deployment types |
| `CustomIncome` | [int, int, int] or null | one of the division's `Incomes` by its values; null = use `DeploymentType` |
| `Entries` | list of `{ UnitId, Quantity, AvailableFromPhase, ExperienceLevel, Transport }` | `UnitId` required; `AvailableFromPhase` 0–2 (A–C); `ExperienceLevel` 0–2; `Transport` a transport export name or null |

A minimal invented `.sd2bg`:

```json
{
  "Format": 1,
  "Kind": "battlegroup",
  "AppVersion": "0.6.0",
  "ExportedUtc": "2026-10-12T09:30:00Z",
  "Division": null,
  "Battlegroup": {
    "Key": "Example_BG",
    "Name": "Example",
    "DeploymentType": 0,
    "CustomIncome": null,
    "Entries": [
      { "UnitId": "Descriptor_Unit_Example_Tank", "Quantity": 2, "AvailableFromPhase": 0, "ExperienceLevel": 0, "Transport": null }
    ]
  },
  "DivisionKey": "Example_Division",
  "DivisionName": "Example Division"
}
```

## Status

- All six formats are written and read by the app. `.sd2units`, `.sd2pkg` and `.sd2mod` are covered by automated
  tests; `.sd2unit`, `.sd2div` and `.sd2bg` not yet.
- Share packages are at version 1: only the kind `mod` exists so far.
