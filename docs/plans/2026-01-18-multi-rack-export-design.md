# Multi-Rack Export Design

**Date:** 2026-01-18
**Milestone:** v0.7.1
**Status:** Approved

## Overview

Enhance the export system to handle many racks gracefully, with rack selection, auto-switching between composite and per-file exports, and context menu integration.

## Problem Statement

Current export lays out all racks horizontally in a single row. With many racks:

- Images become too wide to view or print
- No way to select specific racks for export
- Layout/presentation quality degrades at scale

## Design Decisions

### Rack Selection

- Checklist in ExportDialog when layout has 2+ racks
- All racks checked by default
- Select All / Deselect All convenience buttons
- Minimum 1 rack required to export

### Auto-Switch Logic

| Selected Racks | Image Export (PNG/SVG/JPEG) | PDF Export                    |
| -------------- | --------------------------- | ----------------------------- |
| 1-3 racks      | Single composite image      | Single page, side-by-side     |
| 4+ racks       | ZIP with one image per rack | Multi-page, one rack per page |

UI shows info message when 4+ selected:

- `ℹ️ 5 racks selected — will export as ZIP with one image per rack`

### Output Formats

**ZIP file contents (4+ racks):**

- Filename: `layout-name-2026-01-18.zip`
- Contains: `rack-slug-front.png`, `rack-slug-rear.png`, etc.

**Multi-page PDF (4+ racks):**

- One rack per page
- Rack name as page header
- Auto landscape/portrait per rack based on aspect ratio
- No table of contents (keep it simple)

### Preview Behavior

**1-3 racks:** Current composite preview

**4+ racks:** Paginated preview with navigation:

```
┌─────────────────────────────┐
│   [Rack preview]            │
└─────────────────────────────┘
      ◀  1 of 5  ▶
      Main Rack (42U)
```

### Context Menu Integration

Enhance existing `RackContextMenu.svelte` with new options:

```
┌─────────────────────┐
│ Add Device          │
│ Export...           │  ← NEW
│ Focus               │  ← NEW
├─────────────────────┤
│ Edit Rack           │
│ Rename              │
│ Duplicate Rack      │
├─────────────────────┤
│ Delete Rack         │
└─────────────────────┘
```

- Same menu used on canvas and sidebar RackList
- **Export...** opens ExportDialog with that rack pre-selected
- **Focus** pans + zooms canvas to fit the rack

### Bayed Rack Groups

Context menu on bayed groups works the same:

- Export pre-selects all racks in the group
- Focus shows the entire group

## Files to Modify

**New dependency:**

- `jszip` (~45KB gzipped)

**Modified files:**

| File                     | Changes                                                     |
| ------------------------ | ----------------------------------------------------------- |
| `ExportDialog.svelte`    | Rack checklist, preview pagination, info messages, progress |
| `export.ts`              | `generateSingleRackSVG()`, `exportAsZip()`, multi-page PDF  |
| `types/index.ts`         | `ExportOptions.selectedRackIds?: string[]`                  |
| `RackContextMenu.svelte` | Add `onexport`, `onfocus` props and menu items              |
| `RackList.svelte`        | Wrap rack items with RackContextMenu                        |

**New file:**

- `src/lib/utils/zip.ts` — JSZip wrapper

## Issue Breakdown

### Issue 1: Multi-rack export with selection and auto-switch

- Rack selection checklist in ExportDialog
- Auto-switch: 1-3 racks composite, 4+ racks per-file
- ZIP generation (jszip) for multi-image exports
- Multi-page PDF (one rack per page)
- Preview pagination with navigation
- Progress indicators
- Pre-selection API (`selectedRackIds` prop)

### Issue 2: Context menu Export integration

- Add `onexport` prop to RackContextMenu
- Add "Export..." menu item
- Wire up in canvas and sidebar RackList
- Opens ExportDialog with that rack pre-selected

### Issue 3: Focus on Canvas

- Add `onfocus` prop to RackContextMenu
- Add "Focus" menu item
- Implement pan + zoom to fit logic
- Wire up in canvas and sidebar RackList

**Implementation order:** 1 → 2 → 3

## Out of Scope

- Canvas position preservation (WYSIWYG export)
- Topology-preserving compaction
- Custom grid layout options
- Export presets/templates
- Batch rename in ZIP
- PDF table of contents
