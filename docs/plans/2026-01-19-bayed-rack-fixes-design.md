# Bayed Rack Fixes Design

**Date**: 2026-01-19
**Status**: Approved

## Overview

Three issues with bayed rack rendering and interaction need to be fixed:

1. U measurement labels are offset by ~0.5U
2. Clicking a bay selects only that bay instead of the entire bayed group
3. Selection highlight has too much gap at the bottom

## Issue 1: U Measurement Offset

### Problem

The U labels in the center column between bays appear vertically shifted from the actual U slot positions by approximately 0.5U.

### Root Cause

`BayedRackView.svelte` line 131 calculates U label positions using `RACK_PADDING_HIDDEN` (4px):

```typescript
const yPosition =
  i * U_HEIGHT + U_HEIGHT / 2 + RACK_PADDING_HIDDEN + RAIL_WIDTH;
```

However, bayed racks display visible bay labels above each rack, which means the actual rack rendering uses `RACK_PADDING` (18px). This 14px mismatch causes the U labels to be misaligned with the U slots.

### Solution

Replace `RACK_PADDING_HIDDEN` with `RACK_PADDING` in the U label yPosition calculation.

### Files Changed

- `src/lib/components/BayedRackView.svelte` (line 131)

## Issue 2: Selection Selects Individual Bays

### Problem

Clicking any part of a bayed rack should select the entire bayed group as a single unit. Currently, clicking a bay only selects that individual bay.

### Root Cause

The selection store (`selection.svelte.ts`) only tracks individual rack IDs via `selectedRackId`. When a bay is clicked, it stores that specific bay's ID. BayedRackView then evaluates selection per-bay, only highlighting the clicked bay.

### Solution

Introduce group-level selection:

1. **Selection Store**: Add `selectedGroupId` state and `selectGroup(groupId)` function
2. **Canvas**: When a rack in a bayed group is clicked, call `selectGroup()` instead of `selectRack()`
3. **BayedRackView**: Check `selectedGroupId` to apply selection styling to the entire view

### Files Changed

- `src/lib/stores/selection.svelte.ts` - Add group selection state and function
- `src/lib/components/Canvas.svelte` - Route bayed rack clicks to group selection
- `src/lib/components/BayedRackView.svelte` - Apply group-level selection styling

## Issue 3: Selection Highlight Bottom Gap

### Problem

The selection highlight (pink outline) has too much space between it and the rack content, especially at the bottom.

### Root Cause

The `.bayed-rack-view` container uses:

- `padding: var(--space-3)` (12px) on all sides
- `outline-offset: 2px` for the selection outline
- Flexbox `gap: var(--space-2)` (8px) between rows

This accumulated spacing creates a large visual gap between the content and the selection outline.

### Solution

Reduce padding and outline-offset to tighten the selection box:

- Change `padding: var(--space-3)` → `padding: var(--space-2)` (8px)
- Change `outline-offset: 2px` → `outline-offset: 0` or `1px`

### Files Changed

- `src/lib/components/BayedRackView.svelte` (CSS styles)

## Implementation Order

1. Fix U offset (simplest, isolated change)
2. Fix selection highlight gap (CSS-only change)
3. Implement group-level selection (requires coordination across multiple files)

## Testing Strategy

Visual verification required for all three fixes:

- U labels should be centered within their respective U slot areas
- Clicking any bay should highlight the entire bayed rack group
- Selection outline should have uniform, tight spacing around the content
