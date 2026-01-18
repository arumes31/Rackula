# Half-Width Device UX Design

**Date**: 2026-01-18
**Status**: Approved
**Milestone**: v0.7.1
**Parent Epic**: #159 (Flexible Device Layouts)

## Overview

Complete the half-width device functionality by wiring up existing infrastructure to user-facing interactions. The schemas, collision detection, and rendering are already implemented—this design covers the missing UX layer.

## Design Decisions

| Decision               | Choice                             | Rationale                                         |
| ---------------------- | ---------------------------------- | ------------------------------------------------- |
| Slot targeting         | Auto from cursor position          | Most intuitive; cursor over left half → left slot |
| Drag preview           | Split highlight with divider       | Clear visual affordance without clutter           |
| Occupied slot handling | Auto-snap to available             | Forgiving UX; reduces failed drops                |
| Repositioning          | EditPanel dropdown + drag-within-U | Maximum flexibility for users                     |
| Container placement    | Direct drop with slot overlay      | No extra clicks; hover reveals grid               |
| Slot overlay style     | Subtle dotted grid lines           | Non-intrusive; target slot highlights             |
| Keyboard navigation    | Arrow keys within U                | Consistent with existing up/down movement         |

## Core Interaction Model

### Half-Width Device Placement (Rack-Level)

When dragging a half-width device (`slot_width: 1`) over a rack U:

1. **Visual Split**: A vertical divider line appears in the middle of the U row
2. **Cursor Tracking**: The half under the cursor highlights in accent color
3. **Smart Snapping**: If one slot is occupied, auto-snap to the empty slot regardless of cursor position
4. **Drop Result**: Device placed with `slot_position: 'left'` or `'right'` based on target

Full-width devices (`slot_width: 2`, default) behave exactly as today—no split line, occupy entire U.

### Collision Rules (Already Implemented)

- Two half-width devices can share a U if one is `left` and one is `right`
- Full-width blocks both slots
- Same-slot placement is blocked

## Repositioning & EditPanel

### Two Methods for Changing Slot Position

1. **EditPanel Dropdown**: When a half-width device is selected:

   ```
   Position: [Left ▼]
             ├─ Left
             └─ Right
   ```

2. **Drag Within Same U**: Drag left-positioned device to right half of same U to swap.

### Validation

- Target slot occupied → toast "Right slot occupied by [Device Name]"
- Collision detected → block with feedback
- Undo/redo captures position changes atomically

### Container Children in EditPanel

```
Container: Shelf-1U-2slot
Slot: Bay 1 [▼]
      ├─ Bay 1
      └─ Bay 2
```

## Container Slot Placement

### Drag-to-Container Flow

1. **Hover Detection**: Cursor enters container bounds → subtle dotted grid lines appear
2. **Slot Targeting**: Slot under cursor highlights; incompatible slots stay muted
3. **Drop into Slot**: Device placed with `container_id` and `slot_id` set

### Visual Feedback

```
┌─────────────────────────────────────┐
│ Shelf-1U-2slot                      │
│  ┌─────────┬─────────┐              │
│  │  NUC    │ ░░░░░░░ │  ← Slot 2 highlighted
│  │ (Bay 1) │ (Bay 2) │
│  └─────────┴─────────┘              │
└─────────────────────────────────────┘
```

### Slot Compatibility

- Slots define `accepts: DeviceCategory[]` to filter compatible devices
- Incompatible slots show as unavailable during drag
- Empty `accepts` = accepts all categories

## Keyboard Navigation & Accessibility

### Arrow Key Navigation

- **Left/Right**: Move focus between slots in same U
- **Up/Down**: Move device vertically (existing behavior)
- **Tab**: Move to next device/container
- **Escape**: Exit container scope

### Screen Reader Announcements

```
"Server in left slot at U42, front face"
"NUC in Bay 1 of Shelf-1U-2slot at U10"
"Empty right slot available at U42"
```

### Focus Management

- After placement: Focus moves to newly placed device
- After repositioning: Focus stays on moved device
- After delete: Focus moves to nearest device or rack

## Implementation Scope

### Files to Modify

| File               | Change                                                                          |
| ------------------ | ------------------------------------------------------------------------------- |
| `Rack.svelte`      | Call `calculateDropSlotPosition()`, add split-line visual, pass `slot_position` |
| `Canvas.svelte`    | Pass `slot_position` through `handleDeviceDrop`                                 |
| `layout.svelte.ts` | Update `placeDevice()` to accept `slot_position`                                |
| `EditPanel.svelte` | Add slot position dropdown                                                      |

### Existing Infrastructure (No Changes Needed)

- `RackDevice.svelte` - Already renders half-width correctly
- `collision.ts` - Already has `doSlotsOverlap()`
- `dragdrop.ts` - Already has `calculateDropSlotPosition()`

### Out of Scope for v0.7.1

- Container zoom view (#763)
- Quarter-width devices
- Nested containers
- Pre-built container library

## Issue Breakdown

1. **Wire up half-width drop** - Rack.svelte + Canvas + placeDevice integration
2. **EditPanel slot control** - Dropdown + drag-to-reposition within U
3. **Container slot overlay** - Visual grid on hover during drag
4. **Keyboard slot navigation** - Arrow keys for slot movement
