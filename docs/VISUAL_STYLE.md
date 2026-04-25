## Overall visual direction

**Style:**
Hand-painted, slightly stylized realism. Not pixel art. Soft lighting, rounded shapes, and organic edges. Everything feels “grown” rather than manufactured.

**Theme:**
Nature-driven UI. Wood, vines, leaves, soil tones. The interface blends into the world rather than sitting on top of it.

**Color palette:**

* Dominant: greens (grass, foliage)
* Secondary: browns (dirt, wood UI)
* Accent: blue (water), orange (crops), gold (selection/highlight)
* UI contrast: darker desaturated browns/greens for panels

---

## Layering model (important for implementation)

Think in 4 main layers:

### 1. World layer (bottom)

* Tile grid (grass, dirt, tilled soil, water)
* Static objects (trees, rocks, bushes)
* Terrain transitions (soft blended edges between tiles)

### 2. Interaction layer

* Grid overlay (faint square lines)
* Tile highlight (hover/brush preview)
* Painted elements (seed placements, small sprouts)
* Cursor (context-aware, e.g. seed brush)

### 3. Game UI (HUD)

* Top inventory bar
* Right vertical tool vine
* Bottom right legend
* Bottom instruction strip
* Resource orbs (fertiliser/water)

### 4. Overlay/state indicators

* “WORLD MAP” vs “MARKET” mode label
* Tool info panel (top-right small box)
* Selected item highlight (glow, outline)

---

## Top bar (Inventory)

**Structure:**

* Horizontal wooden panel
* Tabs: `Seeds | Material | Etc`

**Visual behavior:**

* Active tab is brighter and slightly raised
* Items shown as square slots with:

  * Icon (carrot, wheat, etc.)
  * Count number
* Hover = subtle glow
* Selected = golden border + inner light

**Left side detail panel:**

* Shows selected item info:

  * Name
  * Short description
  * Growth time
* Styled as carved wood tooltip

---

## Right side toolbar (vine-based)

**Concept:**
A vertical vine growing upward, with tools as circular “buds”.

**Structure:**

* Organic vine spine
* Tools placed as round nodes:

  * Seed (active in screenshot)
  * Shovel
  * Watering can
  * Hand
  * Pickaxe
  * Bag

**States:**

* Active tool: glowing green/gold ring
* Inactive: muted brown/green
* Hover: slight pulse

**Important design trait:**
No hard edges. Everything is circular and organic.

---

## Center (World grid)

**Grid:**

* Square tiles, clearly visible but softly blended
* Each tile has:

  * Base type (grass, dirt, sand, water)
  * Optional overlay (tilled soil, crops)

**Terrain composition:**

* Natural clustering (forest, rocks, lakes)
* Soft transitions (no harsh borders)

**Farming area:**

* Tilled soil = darker brown squares
* Seeds = tiny green/orange sprouts
* Painted area shows repetition → confirms “brush tool” interaction

---

## Tool interaction (seed painting)

**Visual cues:**

* Cursor becomes a hand/brush
* Highlight square follows cursor
* Multiple tiles affected = brush behavior
* Planted seeds appear instantly as small sprouts

**Feedback:**

* Immediate visual change (no delay)
* Consistent tile-by-tile placement

---

## Mode indicator (top center)

**"WORLD MAP" vs "MARKET":**

* Large centered label
* Styled as carved wood sign
* Secondary button/tab for switching (e.g. “Market” on right)

**Purpose:**
Clear context separation between gameplay modes

---

## Bottom right (Tile legend)

**Structure:**

* Compact panel
* Title: “Tile Legend”

**Content:**
Grid of tile types:

* Grass
* Dirt
* Tilled Dirt
* Sand
* Water
* Rock
* Tree
* Bush
* Shallow Water

**Visual style:**

* Small icon + label
* Neutral background so it doesn’t dominate

---

## Bottom center (controls hint)

**Strip:**

* Thin horizontal UI bar

**Content:**

* Left click → Paint seeds
* Right click → Erase
* Shift → Larger brush

**Style:**

* Minimal, semi-transparent
* Low visual priority

---

## Resource orbs (bottom corners)

### Left: Fertiliser

### Right: Water

**Design:**

* Large glass spheres
* Liquid inside with shine/reflection
* Encased in wood + vines

**Behavior:**

* Fill level represents amount (120/120)
* Color-coded:

  * Green → fertiliser
  * Blue → water

**Role:**
Equivalent to “mana/energy” systems

---

## Micro-interactions and feedback

* Selection glow uses warm gold/green tones
* Hover = soft light, not sharp highlight
* No hard UI snapping; everything eases visually
* Game grid reacts instantly to input (important for brush feel)

---

## Key design principles visible here

1. **Diegetic UI**
   UI elements feel part of the world (wood, vines, organic shapes)

2. **Tool = brush metaphor**
   Direct manipulation, like Photoshop painting

3. **Clear spatial hierarchy**

   * Center = gameplay
   * Top = inventory
   * Right = tools
   * Bottom = support info

4. **Low cognitive load**

   * Immediate visual feedback
   * Minimal text needed

5. **Expandable system**
   Tabs, tools, legend all scale naturally as features grow

---

If you want, next step can be:

* turning this into a concrete UI spec (HTML/CSS layout or Unity/Blazor structure), or
* mapping each visual element to actual components in your ASP.NET + JS setup.
