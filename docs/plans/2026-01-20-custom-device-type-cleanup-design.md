# Custom Device Type Lifecycle Management

**Date:** 2026-01-20
**Status:** Draft
**Related:** [Discussion #569](https://github.com/RackulaLives/Rackula/discussions/569)

## Problem

Custom device types accumulate in `layout.device_types[]` over time. Users create types for one-off needs, experimentation, or intentional library building. Currently:

- No visibility into which types are unused (0 placements)
- No bulk cleanup mechanism
- Users resort to manual YAML editing to prune unused types

## Design Goals

- **Discovery:** Make unused types visually identifiable
- **Low friction:** Quick delete for individual types
- **Bulk cleanup:** Efficient way to clean many at once
- **Gentle nudges:** Prompt at natural moments (export/save)
- **User control:** Setting to disable prompts

---

## 1. Visual Indicator (Trashcan)

Display a trashcan icon next to custom device types with zero placements.

**Where:** `DevicePaletteItem` for custom types only. Icon appears on the right side of the device row.

**Behavior:**

- Only appears on custom device types (not starter library or brand packs)
- Only appears when type has 0 placements across all racks
- Updates reactively when placements change

**Styling:**

- Muted color by default
- Hover state brightens, tooltip: "Delete unused device type"
- Absence of trashcan implicitly communicates "in use"

**Technical:** Compare `layout.device_types[]` slugs against `getUsedDeviceTypeSlugs()`.

---

## 2. Quick Delete with Toast

Clicking the trashcan immediately deletes the type and shows a toast with undo.

**Flow:**

1. User clicks trashcan icon
2. Device type removed from `layout.device_types[]`
3. Toast appears: "Deleted 'Custom Server'" with [Undo] button
4. Toast auto-dismisses after 5 seconds
5. Clicking Undo restores the device type

**Toast behavior:**

- Bottom-right corner position
- Multiple deletes stack or batch: "Deleted 3 device types" [Undo All]
- Undo also available via Ctrl+Z

**Edge cases:**

- Undo works even if user navigates within app
- Closing app dismisses pending undos (data already saved)

---

## 3. Export/Save Cleanup Prompt

Before export or save, offer to clean up unused types.

**Trigger conditions:**

- User initiates export (Ctrl+E) or save (Ctrl+S)
- At least 1 unused custom device type exists
- "Prompt to clean up unused device types" setting is ON

**Dialog:**

```
┌─────────────────────────────────────────────┐
│  Clean Up Device Library?                   │
│                                             │
│  You have 3 unused custom device types.     │
│  Would you like to remove them before       │
│  saving?                                    │
│                                             │
│  ☐ Don't ask again                          │
│                                             │
│  [Review & Clean Up]  [Keep All]  [Cancel]  │
└─────────────────────────────────────────────┘
```

**Actions:**

- **Review & Clean Up** → Opens bulk cleanup dialog
- **Keep All** → Proceeds with export/save
- **Cancel** → Aborts operation
- **Don't ask again** → Disables setting

---

## 4. Bulk Cleanup Dialog

Checklist dialog for selective batch deletion.

**Access:**

- Settings → Device Library → "Clean Up Unused Device Types" button
- "Review & Clean Up" from export/save prompt

**Dialog:**

```
┌─────────────────────────────────────────────────┐
│  Clean Up Device Library                        │
│                                                 │
│  3 unused custom device types                   │
│                                                 │
│  [Select All]  [Deselect All]                   │
│                                                 │
│  ☑ Custom Server         (Server)              │
│  ☑ My UPS                (Power)               │
│  ☐ Test Switch           (Network)             │
│                                                 │
│  [Delete Selected (2)]              [Cancel]    │
└─────────────────────────────────────────────────┘
```

**Features:**

- Checkbox per device, all selected by default
- Device name + category shown
- Select All / Deselect All buttons
- Delete button shows count
- Single undo operation for all deleted

**Empty state:** "No unused device types found."

**After deletion:** If from export/save prompt, proceeds with original operation.

---

## 5. Settings

Toggle to control prompt behavior.

**Location:** Settings → Device Library section

```
Device Library
─────────────────────────────────────────
☑ Prompt to clean up unused device types
  Shows a prompt before export/save when unused
  custom device types exist.

[Clean Up Unused Device Types]
  Opens dialog to review and delete unused types.
```

**Default:** ON (enabled)

**Persistence:** localStorage

**Interaction:** "Don't ask again" checkbox sets this to OFF.

---

## Implementation

### Components to Create/Modify

| Component                    | Change                                      |
| ---------------------------- | ------------------------------------------- |
| `DevicePaletteItem.svelte`   | Add trashcan icon for unused custom types   |
| `Toast.svelte`               | Create toast component with undo support    |
| `CleanupPromptDialog.svelte` | Pre-export/save prompt                      |
| `CleanupDialog.svelte`       | Bulk cleanup checklist                      |
| `SettingsPanel.svelte`       | Device Library section with toggle + button |
| `layout.svelte.ts`           | Add `getUnusedCustomDeviceTypes()` helper   |
| `preferences.ts`             | Add `promptCleanupOnSave` setting           |

### Integration Points

- Hook into export flow (before `exportLayout()`)
- Hook into save flow (before `saveLayout()`)
- Use existing `deleteDeviceTypeRecorded()` command for undo

### Not Included (YAGNI)

- Automatic cleanup (too risky)
- "Unused for X days" tracking (adds complexity)
- Filter view in palette (trashcan + dialog sufficient)
- Per-device-type "keep" flag (overcomplicates)

---

## Issues

1. **#830** - [feat: Add trashcan indicator for unused custom device types](https://github.com/RackulaLives/Rackula/issues/830)
2. **#831** - [feat: Add cleanup prompt on export/save](https://github.com/RackulaLives/Rackula/issues/831)
3. **#832** - [feat: Add bulk cleanup dialog in Settings](https://github.com/RackulaLives/Rackula/issues/832)
