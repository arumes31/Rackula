# Half-Width Device UI Design

**Date:** 2026-01-20
**Status:** Approved
**Related:** #764 (drop mechanics - completed)

## Overview

Enable users to create and visualize half-width devices for rack-mount equipment that occupies only half the rack width (side-by-side PDUs, KVMs, small equipment on shelves).

## Mental Model

Half-width devices are standalone devices with `slot_width: 1` that occupy the left or right half of a U slot. Two half-width devices can share the same U position (one left, one right). This is simpler than container-based approaches and works well for:

- Side-by-side PDUs and KVM switches
- Small equipment on shelves (Pi, NAS, routers) placed as independent devices

## Decisions

| Decision             | Choice                                 | Rationale                                             |
| -------------------- | -------------------------------------- | ----------------------------------------------------- |
| UI for setting width | Checkbox ("Half-width device")         | Consistent with existing "Full Depth" toggle          |
| Visual rendering     | Flush to side (left or right rail)     | Clearest indication of slot, matches physical reality |
| Palette indicator    | "½" badge next to height               | Users need to identify half-width when browsing       |
| Details view         | Show "Width: Half" for half-width only | Reduce noise, full-width is assumed default           |
| Width editing        | Not allowed after creation             | Simpler, avoids migration complexity                  |

## Implementation

### 1. AddDeviceForm.svelte

Add checkbox after "Full Depth" toggle:

```svelte
<label>
  <input type="checkbox" bind:checked={isHalfWidth} />
  Half-width device
  <span class="helper">Occupies left or right half of rack width</span>
</label>
```

On submit, map to `slot_width`:

```typescript
slot_width: isHalfWidth ? 1 : 2;
```

### 2. RackDevice.svelte

Calculate width and position based on device type and placement:

```typescript
const isHalfWidth = deviceType.slot_width === 1;
const deviceWidth = isHalfWidth ? interiorWidth / 2 : interiorWidth;
const deviceX =
  isHalfWidth && placedDevice.slot_position === "right"
    ? RAIL_WIDTH + interiorWidth / 2
    : RAIL_WIDTH;
```

Apply to device rectangle and scale label/image accordingly.

### 3. DevicePalette.svelte

Add "½" indicator next to device height for half-width devices:

```svelte
{#if device.slot_width === 1}
  <span class="half-width-badge">½</span>
{/if}
```

Style with muted/secondary text to avoid visual clutter.

### 4. Device Details

Show width only for half-width devices:

```svelte
{#if deviceType.slot_width === 1}
  <dt>Width</dt>
  <dd>Half</dd>
{/if}
```

## Files Affected

| File                                      | Change                                                       |
| ----------------------------------------- | ------------------------------------------------------------ |
| `src/lib/components/AddDeviceForm.svelte` | Add checkbox, wire to `slot_width`                           |
| `src/lib/components/RackDevice.svelte`    | Calculate width/position from `slot_width` + `slot_position` |
| `src/lib/components/DevicePalette.svelte` | Add "½" badge for half-width devices                         |
| Device details component                  | Show "Width: Half" for half-width                            |

## No Changes Needed

- Schema (`slot_width` already defined)
- `CreateDeviceTypeInput` (already has `slot_width`)
- Collision detection (already slot-aware)
- Drop mechanics (implemented in #764)

## Out of Scope

- Quarter-width or other fractional widths
- Editing width of existing device types
- Container-based shelves
- Starter library audit (separate issue)

## Testing

- Create half-width device via form
- Verify "½" badge appears in palette
- Drag half-width device, verify split-line preview
- Drop at left/right positions, verify correct rendering
- Place two half-width devices at same U (left + right)
- Verify full-width devices unchanged
