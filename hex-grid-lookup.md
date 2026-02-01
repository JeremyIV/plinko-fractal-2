---
name: Hex Grid Peg Lookup
overview: Disable fractal/trajectory rendering behind a flag, implement hexagonal coordinate transformation to find the nearest peg on hover, and highlight that peg.
todos:
  - id: flags
    content: Add feature flags to disable fractal rendering and trajectory drawing
    status: pending
  - id: hex-math
    content: Implement pixelToHexGrid() with adapted transformation for our stretched grid
    status: pending
  - id: hex-round
    content: Implement hexRound() for cube coordinate rounding
    status: pending
  - id: hex-to-pixel
    content: Implement hexToPixel() to convert hex coords back to peg position
    status: pending
  - id: hover-handler
    content: Update mousemove handler to compute and store nearest peg
    status: pending
  - id: draw-highlight
    content: Add highlight rendering in drawOverlay()
    status: pending
isProject: false
---

# Hex Grid Peg Lookup Implementation

## Current Grid Parameters

From [index.html](index.html) lines 117-122:

```javascript
const pegSpacingX = 40;   // horizontal spacing
const pegSpacingY = 50;   // vertical spacing  
const pegStartY = 380;    // grid origin Y
// rowStartX computed per row, odd rows offset by pegSpacingX/2 = 20px
```

## Step 1: Add Feature Flags

Add flags at the top of the script section to disable expensive rendering:

```javascript
const ENABLE_FRACTAL_RENDERING = false;  // Toggle WebGL basin computation
const ENABLE_TRAJECTORY_ON_HOVER = false; // Toggle trajectory drawing
const ENABLE_HEX_DEBUG = true;            // Show nearest peg highlight
```

Wrap the `progressiveRender()` call and trajectory drawing code in conditionals using these flags.

## Step 2: Adapt Hex Math to Our Grid

Our grid is a **stretched honeycomb** (not regular hexagons). The transformation needs to account for:

- **pegSpacingX = 40**: horizontal distance between pegs in same row
- **pegSpacingY = 50**: vertical distance between rows
- **Offset = 20**: odd rows shifted right by half pegSpacingX

The standard pointy-top hex formula assumes regular hexagons. We need to scale coordinates:

```javascript
function pixelToHexGrid(px, py) {
    // Transform to grid-relative coordinates
    const x = px - gridOriginX;
    const y = py - pegStartY;
    
    // Scale to normalized hex space where spacing = 1
    // For our stretched grid: scaleX based on pegSpacingX, scaleY based on pegSpacingY
    const normX = x / pegSpacingX;
    const normY = y / pegSpacingY;
    
    // Apply axial coordinate conversion (adapted for our aspect ratio)
    // Standard formula: q = (√3/3 * x - 1/3 * y) / size
    // Our grid: row offset is 0.5 in normalized coords
    const q = normX - 0.5 * normY;  // Column, accounting for stagger
    const r = normY;                 // Row
    
    return hexRound(q, r);
}
```

## Step 3: Implement hexRound

Standard cube coordinate rounding (from user's reference):

```javascript
function hexRound(q, r) {
    const s = -q - r;
    let rq = Math.round(q);
    let rr = Math.round(r);
    let rs = Math.round(s);
    
    const qDiff = Math.abs(rq - q);
    const rDiff = Math.abs(rr - r);
    const sDiff = Math.abs(rs - s);
    
    if (qDiff > rDiff && qDiff > sDiff) {
        rq = -rr - rs;
    } else if (rDiff > sDiff) {
        rr = -rq - rs;
    }
    
    return { q: rq, r: rr };
}
```

## Step 4: Convert Hex Coords Back to Peg

```javascript
function hexToPixel(q, r) {
    const row = r;
    const col = q + Math.floor(r / 2);  // Adjust for stagger
    
    // Check bounds
    const rowPegCount = pegsPerRow - (row % 2);
    if (row < 0 || row >= pegRows || col < 0 || col >= rowPegCount) {
        return null;
    }
    
    // Calculate pixel position
    const rowOffset = (row % 2) * (pegSpacingX / 2);
    const rowStartX = (glCanvas.width - (rowPegCount - 1) * pegSpacingX) / 2;
    
    return {
        x: rowStartX + col * pegSpacingX + rowOffset,
        y: pegStartY + row * pegSpacingY,
        row: row,
        col: col
    };
}
```

## Step 5: Update Hover Handler

In the `mousemove` event handler:

```javascript
overlayCanvas.addEventListener('mousemove', (e) => {
    const pos = getMousePos(e);
    
    if (ENABLE_HEX_DEBUG) {
        const hex = pixelToHexGrid(pos.x, pos.y);
        const nearestPeg = hexToPixel(hex.q, hex.r);
        highlightedPeg = nearestPeg;  // Store for drawing
        drawOverlay();
    }
    
    // ... existing trajectory logic (guarded by flag)
});
```

## Step 6: Draw Highlighted Peg

In `drawOverlay()`, add highlight rendering:

```javascript
if (ENABLE_HEX_DEBUG && highlightedPeg) {
    ctx.beginPath();
    ctx.arc(highlightedPeg.x, highlightedPeg.y, pegRadius + 4, 0, Math.PI * 2);
    ctx.strokeStyle = '#ffff00';
    ctx.lineWidth = 3;
    ctx.stroke();
}
```

## Files to Modify

- [index.html](index.html): All changes in the `<script>` section

## Verification

When hovering over the canvas:

- The nearest peg should highlight with a yellow ring
- Moving the mouse should smoothly transition the highlight between pegs
- The highlight should correctly follow the honeycomb Voronoi boundaries