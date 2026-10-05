# AgOpenGPS → Valtra ISOXML

Convert [AgOpenGPS](https://github.com/AgOpenGPS/AgOpenGPS) field exports into clean ISOXML (`TASKDATA.XML`) that a Valtra SmartTouch / Valtra Guide terminal can import from a USB stick.

> **Status:** tested against a set of 48 real AgOpenGPS fields (structure and geometry checked). Not yet confirmed on a physical Valtra terminal. If you try it, please open an issue with the result.

## Why this exists

AgOpenGPS can export `zISOXML/v3/TASKDATA.xml` and `zISOXML/v4/TASKDATA.xml` for every field, but the files often fail to import on tractor terminals. Problems found in real exports:

| Problem | Effect |
| --- | --- |
| Coordinates written with a decimal comma (`46,0396`) when the PC uses a Hungarian/European locale | Every position is invalid XML-schema-wise |
| IDs like `PFD-1`; the `GPN` reuses its `GGP0` id | Invalid and duplicated IDs |
| An empty "curve" guidance entry written for every AB line | Import errors or empty lines |
| Coordinates with up to 13 decimals | Schema allows 9 |
| UTF-8 BOM, `°` and accented characters in line names | Trouble on strict terminals |
| One file per field | Terminals read a single `TASKDATA.XML` |

## What it does

- Fixes all of the above.
- Merges all fields into one `TASKDATA/TASKDATA.XML`, and optionally writes one folder per field.
- **Estimates missing boundaries** for fields that were never boundary-recorded, from the driven track in `Elevation.txt` (outer edge of the track plus half the tool width from `Field.kml`). Fields whose track is scattered over separate areas are skipped.
- **Curve lines** can be converted to straight AB segments (default, within 0.25 m of the curve), kept as thinned curves, or removed.
- Output for ISOXML **v4** (recommended) and **v3** (older, boundaries and straight lines only).

Estimated boundaries are approximations, not surveyed outlines. Check them on the terminal before relying on them for section control.

## Usage

### Browser tool (full feature set)

Open `aog-to-valtra.html` in a browser (or host it with GitHub Pages), drop your `Fields.zip` onto it, review the table, and download the zip. Everything runs locally; nothing is uploaded. It loads JSZip from cdnjs, so it needs an internet connection the first time.

Expected zip layout (the normal AgOpenGPS `Fields` folder):

```
Fields.zip
└── My Field/
    ├── Elevation.txt
    ├── Field.kml
    └── zISOXML/
        ├── v3/TASKDATA.xml
        └── v4/TASKDATA.xml
```

A single `TASKDATA.xml` can also be dropped in.

### Python script (fixes only)

```bash
python3 aog_to_valtra.py Fields.zip out_dir
```

Requires Python 3.8+ and no extra packages. The script applies the file fixes and the merge, but does **not** estimate boundaries or convert curves; use the browser tool for those.

## Output layout

```
ISOXML_v4/
├── all_fields/TASKDATA/TASKDATA.XML      <- all fields in one file
└── per_field/<field>/TASKDATA/TASKDATA.XML
ISOXML_v3/
└── (same structure)
```

## Importing on the tractor

1. Format a USB stick as **FAT32**.
2. Copy the `TASKDATA` folder to the **root** of the stick. The file name must be uppercase: `TASKDATA.XML`.
3. Import it from the SmartTouch terminal (Valtra Guide / TaskDoc).

Try v4 first; use v3 only if v4 is rejected. If the terminal still complains, run the file through the [AEF Taskdata Validator](https://www.aef-online.org/aef-news/aef-taskdata-validator.html) and open an issue with the report.

## Known limitations

- Not verified on real Valtra hardware.
- Only field boundaries and guidance lines are converted. Headlands, flags, obstacles, coverage and tasks are not exported.
- Boundary rings are written as exported by AgOpenGPS (counter-clockwise, not repeated at the end).
- Guidance heading, accuracy and offset attributes are not written.

## Repository contents

| File | Purpose |
| --- | --- |
| `aog-to-valtra.html` | Browser converter (boundary estimation, curve handling) |
| `aog_to_valtra.py` | Command-line converter (file fixes and merge only) |
