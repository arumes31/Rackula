# Data Directory Structure Refactor

**Issue:** #570 (Developer-Friendly Data Format)
**Date:** 2026-01-22
**Status:** Draft

## Overview

Refactor the Rackula API's data storage from flat files to a folder-per-layout structure, making layouts self-contained, portable, and git-friendly.

## Motivation

1. **Git-friendliness:** Each layout as a self-contained folder enables individual version control (one repo per layout, or layouts as subdirectories)
2. **Portability:** Copy/move a layout folder and everything travels together (YAML + assets)
3. **File manager clarity:** Folders make layouts visually distinct when browsing in Finder/Explorer

## Data Structure

### New Folder Structure

```
/data/
├── My Homelab-550e8400-e29b-41d4-a716-446655440000/
│   ├── my-homelab.rackula.yaml
│   └── assets/                              # only if custom images exist
│       └── custom-nas/
│           ├── front.png
│           └── rear.png
├── Server Closet-7c9e6679-7425-40de-944b-e07fc1f90ae7/
│   └── server-closet.rackula.yaml           # no assets folder (no custom images)
```

### Naming Conventions

| Component     | Format                          | Example                                            |
| ------------- | ------------------------------- | -------------------------------------------------- |
| Folder        | `{Human Name}-{full UUID}/`     | `My Homelab-550e8400-e29b-41d4-a716-446655440000/` |
| YAML file     | `{slugified-name}.rackula.yaml` | `my-homelab.rackula.yaml`                          |
| Assets folder | `assets/`                       | Only created when layout has custom images         |

### YAML Format

```yaml
metadata:
  id: 550e8400-e29b-41d4-a716-446655440000
  name: My Homelab
  schema_version: "1.0"
  description: "Basement setup for home automation and media server"

racks:
  - name: Main Rack
    height: 42
    devices:
      - ...

  - name: UPS Closet
    height: 12
    devices:
      - ...
```

### Metadata Fields

| Field            | Required | Purpose                                                |
| ---------------- | -------- | ------------------------------------------------------ |
| `id`             | Yes      | UUID - stable identity across renames/moves            |
| `name`           | Yes      | Human-readable layout name                             |
| `schema_version` | Yes      | Format version for future migrations                   |
| `description`    | No       | Optional notes about the layout (empty string default) |

## API Changes

### Routes

Routes change from slug-based to UUID-based:

| Before                           | After                                                      |
| -------------------------------- | ---------------------------------------------------------- |
| `GET /api/layouts/my-homelab`    | `GET /api/layouts/550e8400-e29b-41d4-a716-446655440000`    |
| `PUT /api/layouts/my-homelab`    | `PUT /api/layouts/550e8400-e29b-41d4-a716-446655440000`    |
| `DELETE /api/layouts/my-homelab` | `DELETE /api/layouts/550e8400-e29b-41d4-a716-446655440000` |

UUID-only routing provides stable URLs that don't change when layouts are renamed.

## Migration Strategy

### Reading Old Layouts

The API continues to read flat `.yaml` files at root level for backwards compatibility.

### Migration on First Save

When a user saves an old-format layout:

1. Generate UUID for the layout
2. Create folder `{Name}-{UUID}/`
3. Move YAML to `{slugified-name}.rackula.yaml` inside folder
4. Move assets from `/data/assets/{old-id}/` to `{folder}/assets/`
5. Delete old flat file and old assets folder

### No Startup Migration

Layouts remain in old format until user actively saves:

- Read-only access to old layouts works indefinitely
- Users can delay migration as long as they want
- External tools reading the old format continue working until layout is saved

## Rename Behavior

When a user renames a layout (e.g., "My Homelab" → "Server Closet"):

```
Before:
My Homelab-550e8400-e29b-41d4-a716-446655440000/
├── my-homelab.rackula.yaml
└── assets/

After:
Server Closet-550e8400-e29b-41d4-a716-446655440000/
├── server-closet.rackula.yaml
└── assets/
```

**What changes:**

- Folder name prefix (human-readable part)
- YAML filename (slugified name)
- Content inside YAML (`name` field)

**What stays the same:**

- UUID remains identical
- Assets folder structure unchanged
- API route still uses same UUID

**Git-friendliness:**

- `git mv` tracks this as a rename
- Commit history preserved
- External references via UUID still work

## Browser Export (Zip Format)

For browser-only users (no API), export creates a zip with the folder structure inside:

```
My Homelab-550e8400-e29b-41d4-a716-446655440000.zip
└── My Homelab-550e8400-e29b-41d4-a716-446655440000/
    ├── my-homelab.rackula.yaml
    └── assets/                          # only if custom images
        └── custom-nas/
            └── front.png
```

**Behavior:**

- Zip filename matches the folder inside
- Unzip creates the exact folder structure ready for:
  - Dropping into a git repo
  - Placing in API's `/data/` directory
  - Sharing with others
- Same format whether exported from API mode or browser-only mode

**Import:**

- User drags/drops or selects a `.zip` file
- App detects folder structure inside zip
- Extracts and loads the layout (with assets)

## Scope

### In Scope

| File                            | Changes                                                           |
| ------------------------------- | ----------------------------------------------------------------- |
| `api/src/storage/filesystem.ts` | Folder-based storage, UUID routing, migration logic               |
| `api/src/storage/assets.ts`     | Assets path now inside layout folder                              |
| `api/src/routes/layouts.ts`     | Routes use UUID instead of slug                                   |
| `api/src/schemas/layout.ts`     | Add `metadata` section with `id`, `schema_version`, `description` |
| `src/lib/utils/export.ts`       | Zip export with folder structure                                  |
| `src/lib/utils/import.ts`       | Import from new zip format                                        |
| `src/lib/schemas/*.ts`          | Frontend schema updates to match                                  |

### Out of Scope

- **Session storage:** Ephemeral, stays as-is
- **UI components:** No visible changes except export/import handles new format
- **File System Access API:** Browser support is only 32% (Chrome/Edge only), not viable as primary solution

## Browser Limitations

The File System Access API technically supports writing directories, but:

| Browser           | Support                |
| ----------------- | ---------------------- |
| Chrome/Edge/Opera | ✅ Full (desktop only) |
| Firefox           | ❌ None                |
| Safari            | ❌ None                |
| Mobile browsers   | ❌ None                |

With only 32% global coverage, this isn't viable for homelabbers who often use Firefox or Safari. Hence the zip-based approach for browser exports.

## Testing

Schema validation tests should be named `ankush-yaml-schema.test.ts` (or `.spec.ts` for E2E) in honor of Ankush who originally proposed YAML schema formalization.

Test coverage should include:

- Schema validates correctly formed layouts
- Schema rejects malformed metadata (missing id, invalid schema_version, etc.)
- Migration from old format preserves data integrity
- Round-trip: export → import produces identical layout

## Future Considerations

- **Layout-level description:** Schema supports it now, UI can surface it later (layout list subtitle, properties panel, search)
- **Additional metadata fields:** `author`, `tags`, `created`, `updated` deliberately omitted (YAGNI). Can be added later if clear use cases emerge.
