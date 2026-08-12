---
name: cc-figma-fixed-canvas
description: "Generate a Figma component from a fixed-canvas contract. Use when the user wants to build or update a Figma component from a contract definition with pixel-bbox slots — e.g. 'build the fixed-canvas Figma component from contract', 'generate backdrop+overlay component', 'create component from badge layout contract'. PREREQUISITE: If token bindings are required, run cc-figma-tokens first; tokens={} is valid for this contract shape and skips token binding entirely. Requires figma-use and figma-generate-library skills."
---

# cc-figma-fixed-canvas — Component Contracts Fixed-Canvas Skill

Generate Figma components from fixed-canvas contracts. This skill reads a contract with pixel-bbox `slots[]` and `rendering.baseCanvas`, places slot frames at absolute coordinates, wires slot content to component properties via `propRefs`, and builds any variant matrix from remaining enum props.

**Prerequisites — load these skills first:**
- `figma-use` — Plugin API syntax rules (mandatory before every `use_figma` call)
- `figma-generate-library` — Design system workflow and component creation patterns
- `cc-figma-tokens` — Only required if the contract's `tokens` object is non-empty. Contracts with `tokens: {}` are valid; skip token binding in that case.

**Always pass `skillNames: "cc-figma-fixed-canvas"` when calling `use_figma` as part of this skill.**

**Contrast with `cc-figma-component`:**
- `cc-figma-component`: Handles `composition.slots` (Figma structural children, auto-layout, HUG sizing)
- `cc-figma-fixed-canvas`: Handles top-level `slots[]` (pixel bboxes, FIXED sizing, no auto-layout)

---

## Global Rules

### Frame sizing defaults to FIXED (opposite of cc-figma-component)

The base canvas and all slot frames use `FIXED` sizing at the dimensions specified in the contract. Figma's default (HUG) does NOT apply here.

```js
node.layoutMode = 'NONE';
node.layoutSizingHorizontal = 'FIXED';
node.layoutSizingVertical = 'FIXED';
node.resize(width, height);
```

This applies to: the base canvas frame, every slot child frame, variant duplicates, the Generation Notes frame. Absolute positioning within the canvas is achieved with `.x` and `.y` properties — no auto-layout.

---

## 1. Configuration

Before anything else, read `.component-contracts` from the project root. Never output `FIGMA_ACCESS_TOKEN` in any response.

```
FIGMA_ACCESS_TOKEN=...       # never output this value
FIGMA_FILE_KEY=...           # use for all use_figma calls
TOKENS_DIR=...               # root directory of token files (if tokens != {})
CONTRACTS_DIR=...            # root directory of contract files
```

---

## 2. Contract Structure

A fixed-canvas contract is a JSON file at `{CONTRACTS_DIR}/{id}.contract.json`. Read it fully before planning anything.

Key sections used by this skill:

**`slots[]`** — array of pixel-bbox content injection points:
```json
{
  "slots": [
    {
      "id": "backdrop",
      "role": "background",
      "kind": "image",
      "bbox": { "x": 0, "y": 0, "w": 832, "h": 468 },
      "fixedSize": true,
      "isRequired": true,
      "propRefs": ["backgroundImageUrl", "backgroundImageAlt"]
    },
    {
      "id": "title_overlay",
      "role": "title_overlay",
      "kind": "image",
      "bbox": { "x": 40, "y": 267, "w": 360, "h": 160 },
      "fixedSize": true,
      "isRequired": true,
      "propRefs": ["titleOverlayImageUrl", "titleOverlayAlt"]
    }
  ]
}
```

**`rendering.baseCanvas`** — the outer canvas dimensions:
```json
{
  "rendering": {
    "baseCanvas": {
      "width": 832,
      "height": 468
    },
    "outputFormats": ["jpg"]
  }
}
```

**`propRefs`** — props keys that supply this slot's content:
- `kind: "text"` + `propRefs: ["label"]` → wired to a TEXT component property
- `kind: "image"` + `propRefs: ["imageUrl", "imageAlt"]` → documented in Generation Notes (Figma cannot bind URL dynamically, runtime concern only)
- No `propRefs` → placeholder only, flagged in Generation Notes

**`props`** — component properties, some of which drive the variant matrix. Enum props not consumed by `propRefs` become variant axes (unless overridden with `x-figma.variantAxis: false`)

**`tokens`** — Tier 3 token bindings (same format as `cc-figma-component`). May be empty (`tokens: {}`), which is valid for fixed-canvas contracts and skips token derivation entirely.

---

## 3. Slot Rendering Rules

### For `kind: "image"`

Create a **placeholder rectangle** — actual URLs are runtime concerns and cannot be bound at Figma generation time.

1. Create a rectangle node with dimensions from `bbox.w` × `bbox.h`
2. Position absolutely: `node.x = bbox.x`, `node.y = bbox.y`
3. Set sizing: `layoutSizingHorizontal = 'FIXED'`, `layoutSizingVertical = 'FIXED'`
4. Add a text label showing the slot id: `child.characters = slot.id`
5. Fill color: if a placeholder/neutral Semantic token exists, bind it (Phase 0 validation); otherwise, use a fixed neutral gray (`#E5E7EB`, Tailwind `gray-200`) — **do not leave it colorless**
6. Use 2pt stroke in `#9CA3AF` (Tailwind `gray-400`) for visual separation
7. Document in Generation Notes: "Image slot — runtime content injected via prop [propRef1], [propRef2]. Figma shows placeholder; actual rendering driven by props."

### For `kind: "text"`

Create a **text node**:

1. Create a Text node with dimensions from `bbox.w` × `bbox.h`
2. Position absolutely: `node.x = bbox.x`, `node.y = bbox.y`
3. Set sizing: `layoutSizingHorizontal = 'FIXED'`, `layoutSizingVertical = 'FIXED'`
4. Add placeholder text: `node.characters = "{" + slot.id + "}"`
5. Apply text properties (font, size, weight, color) by binding to Semantic tokens if available in `tokens.typography` / `tokens.color.text.*`; otherwise use defaults (e.g., system font, 14px, `#1F2937` gray-800)
6. If `propRefs` is present, the text content will be wired to a component property (Phase 3)

---

## 4. propRefs Wiring Rules

### TEXT slots with propRefs

If `kind: "text"` and `propRefs: ["label"]` (single string prop):

1. Define a TEXT component property matching the prop name from the contract (e.g. `"dayBadge1Text"`)
2. Bind the slot's text node `characters` to this property
3. The text content is fully driven by the component property at runtime

If `propRefs` has multiple entries (e.g. `["dayBadge1Text", "dayBadge1Variant"]`), only the first entry is used for TEXT binding — the remaining entries are documented in Generation Notes as "metadata props not bound to text content."

### IMAGE slots with propRefs

Image content cannot be dynamically wired to component properties in Figma's Plugin API — `instanceSwapPreferredComponent` and instance property binding do not support external URLs.

**No component property is created for image `propRefs`.** Instead:
1. Document in Generation Notes: "Image slot [id] expects props [propRef1], [propRef2]. At runtime, the consuming app injects assets. In Figma, this is a static placeholder."
2. If the contract specifies multiple image props (e.g. `titleOverlayImageUrl` + `titleOverlayAlt`), list both in the note — these are runtime concerns, not Figma wiring.

### Slots with no propRefs

If a slot lacks `propRefs`:
1. Create the placeholder frame (image or text as above)
2. **Do NOT** create a component property
3. Document in Generation Notes: "Slot [id] has no propRefs — content source undefined. Check contract."

---

## 5. Variant Matrix — Enum Props

After wiring `propRefs`, inspect the contract's `props` section for `type: "enum"` entries that are **not** consumed by any slot's `propRefs`.

### Variant axis rule — opt-out model

By default, every unbound enum prop becomes a **variant axis** in the component set. To exclude an enum prop from the variant matrix, add `x-figma.variantAxis: false` to the prop in the contract:

```json
{
  "props": {
    "dayBadge1Variant": {
      "type": "enum",
      "values": ["weekend", "weekday"],
      "required": true,
      "x-figma.variantAxis": false,
      "description": "..."
    }
  }
}
```

If `variantAxis` is not specified (default), the enum **becomes a variant property.**

### Variant naming and binding

For each variant value combination:
1. Duplicate the base canvas frame
2. **Immediately after duplication, reset sizing to FIXED on both dimensions** — duplication may preserve or lose sizing modes; re-apply explicitly:
   ```js
   duplicate.layoutSizingHorizontal = 'FIXED';
   duplicate.layoutSizingVertical = 'FIXED';
   ```
3. Apply variant-specific token overrides (if any token depends on the variant value)
4. Name each variant: `"Variant=Weekend, State=Default"` (title-case, Property=Value format)
5. Before calling `combineAsVariants`, run a FIXED enforcement pass:
   ```js
   variants.forEach(v => {
     v.layoutSizingHorizontal = 'FIXED';
     v.layoutSizingVertical = 'FIXED';
   });
   ```
6. Call `combineAsVariants` to create the component set
7. Rename to `{ComponentName}/{PrimaryVariantValue}` (e.g. `Banner/Widget`)
8. **After `combineAsVariants`**, manually grid-layout variants (same pattern as `cc-figma-component` Phase 3:8)

### Token-mapped enums with missing paths

If an enum prop has `tokenMapped: true` but no Semantic token value is defined in `tokens`, do **NOT** guess or substitute a fallback token. Instead:
1. Flag in Generation Notes: "Variant [prop] is tokenMapped but has no token path in contract. Manual bind required."
2. Proceed with the variant (leave color/fill unbound for that variant)

---

## 6. Workflow

### Phase 0 — Discovery (read-only)

1. Read `.component-contracts` configuration
2. Read the contract file from `CONTRACTS_DIR`
3. **Routing decision:** Confirm the contract has:
   - Top-level `slots[]` array (distinguishes this skill from `cc-figma-component`)
   - `rendering.baseCanvas` with `width` and `height`
   - Either `composition.slots` is absent OR `composition.slots` is empty
   - If any of the above fails, **stop and inform user**: "This contract appears to be a composition-based component (cc-figma-component), not fixed-canvas. Route to cc-figma-component skill instead."
4. Inspect the Figma file — check for existing Semantic/Primitive collections and any existing component with this name
5. If `tokens` is non-empty: verify that Semantic and Primitive collections exist — if not, stop and tell the user to run `cc-figma-tokens` first
6. If `tokens: {}`, skip to step 7 — token binding is not applicable
7. Validate all `slots[]` entries:
   - Check that no two slots have identical `bbox.x, bbox.y, bbox.w, bbox.h` (overlap detection)
   - Confirm all `bbox` values are non-negative and within `rendering.baseCanvas` bounds
8. Inspect all `propRefs` across slots — confirm that referenced props exist in the contract's `props` section
9. Identify unbound enum props (those not consumed by `propRefs` and without `x-figma.variantAxis: false`) — these become variant axes
10. Present a complete plan to the user:
    - Component name and base canvas dimensions
    - Slot count by kind (image/text), with propRefs status
    - Variant matrix (if any enum props)
    - Any missing Semantic variables (if tokens != {})
11. **Await explicit user approval before proceeding**

### Phase 1 — Page setup

1. Create a dedicated page for the component: `{ComponentName}` (e.g. `GUXHorizontalPoster`)
2. Switch to that page with `await figma.setCurrentPageAsync(page)`
3. Return the page ID

### Phase 2 — Base canvas frame + slot placement

1. Create the base canvas frame with dimensions from `rendering.baseCanvas`:
   ```js
   const frame = figma.createFrame();
   frame.name = '{ComponentName}';
   frame.layoutMode = 'NONE';
   frame.layoutSizingHorizontal = 'FIXED';
   frame.layoutSizingVertical = 'FIXED';
   frame.resize(baseCanvas.width, baseCanvas.height);
   frame.x = 0;
   frame.y = 0;
   ```
2. For each slot in `slots[]`, create the appropriate child frame (image or text, per §3):
   - Absolute positioning: `child.x = bbox.x`, `child.y = bbox.y`
   - FIXED sizing immediately after creation
   - Label text and placeholder styling as specified in §3
3. If `tokens != {}`, apply token bindings to any child fill/stroke/text colors that have corresponding token entries
4. Validate: `get_screenshot` — confirm all slots are visible and positioned correctly within the base canvas bounds
5. **Await user checkpoint** (if debug mode enabled)

### Phase 3 — propRefs wiring (TEXT properties only)

For each slot with `kind: "text"` and `propRefs`:

1. Extract the first prop name from `propRefs` array (e.g. `"dayBadge1Text"`)
2. Confirm this prop exists in the contract's `props` section and is a string type
3. Define a TEXT component property on the base component:
   ```js
   const textProp = {
     name: propName,
     type: 'TEXT'
   };
   baseComponent.addComponentProperty(textProp);
   ```
4. Bind the slot's text node `characters` to this property:
   ```js
   textNode.setBoundVariable('characters', textVariable);
   ```
   (Use Semantic variable lookup if available; otherwise bind to the prop value directly — consult `cc-figma-component` §3 Tier 3 token rules if needed)
5. Document any remaining `propRefs` entries in Generation Notes: "[slot] wired to [first-propRef]; remaining props [second+] are runtime-only."

### Phase 4 — Variant matrix (if applicable)

If there are unbound enum props (with `variantAxis != false`):

1. For each variant value combination, duplicate the base canvas frame
2. Apply variant-specific token overrides (if the contract defines token variants)
3. Name and grid-layout as specified in §5
4. Validate: `get_screenshot` — confirm variants are laid out visibly

### Phase 5 — Canvas annotations (Generation Notes)

Create a `⚠️ Generation Notes` frame on the component page, below all components, documenting:

**Always include notes for:**
- Image slot placeholders and their runtime prop sources
- Slots with no `propRefs`
- Unbound enum props (if token path missing)
- Any slot with bbox outside base canvas bounds (clipping risk)
- `x-figma.variantAxis: false` enums (why they were excluded from variant matrix)
- Whitelist-matched non-content slots (e.g. `safe_area` marker)
- Any assumption made during generation

**Example:**
```
⚠️ Generation Notes

Backdrop slot (image): Supplied by props backgroundImageUrl + backgroundImageAlt.
Figma shows placeholder; actual rendering app-driven.

Title overlay slot (image): Supplied by props titleOverlayImageUrl + titleOverlayAlt.
Figma shows placeholder; actual rendering app-driven.

Variant axis "Day Badge Variant" (enum) has tokenMapped=true but no token path.
Manual bind required post-generation.

No issues with bbox placement — all slots fit within base canvas 832×468.
```

If there are no notes to add, write: "No issues — all slots placed correctly, propRefs wired or documented."

---

### Phase 6 — Final validation + cleanup

1. `get_metadata` — confirm structure: slot count, property definitions, variant count (if any)
2. `get_screenshot` — visual check: all slots visible, correct positioning, no overlaps, variants laid out clearly
3. Return structured summary:
   ```json
   {
     "component": "GUXHorizontalPoster",
     "baseCanvasDimensions": { "width": 832, "height": 468 },
     "slotCount": 2,
     "slots": [
       { "id": "backdrop", "kind": "image", "propRefs": ["backgroundImageUrl", "backgroundImageAlt"] },
       { "id": "title_overlay", "kind": "image", "propRefs": ["titleOverlayImageUrl", "titleOverlayAlt"] }
     ],
     "variantCount": 0,
     "propertiesDefined": [],
     "pageId": "..."
   }
   ```
4. **Cleanup** — always run this at the end. The only exception is debug mode.

   Delete the `scripts/` directory and any working files:
   ```bash
   npx rimraf scripts/
   rm -f .mcp-*.json .mcp-*.txt .*-phase*.js .*-phase*.json .*-args*.json .*-code*.js .*-code*.json .tmp-*.js
   ```

---

## 7. Known Constraints (shared with cc-figma-component)

Refer to **cc-figma-component §7-8** for common constraints. Fixed-canvas-specific additions:

- **FIXED sizing is non-negotiable** — Figma defaults new frames to `FIXED` at 100px. You must set both sizing lines explicitly on every frame, every slot child, every variant duplicate:
  ```js
  node.layoutSizingHorizontal = 'FIXED';
  node.layoutSizingVertical = 'FIXED';
  node.resize(width, height);
  ```
  Omitting either line or omitting `layoutMode = 'NONE'` results in auto-layout, which is incorrect for fixed-canvas.

- **Absolute positioning** — use `.x` and `.y` properties directly. Never add children to an auto-layout parent.

- **Non-content slot whitelist** — the following roles are recognized as non-content metadata markers and do NOT generate visual layers:
  - `safe_area` — guide marker for collision checks
  - (Additional roles may be added with explicit justification and documented here)

  All other slots generate layers (image rectangle or text node) even if `propRefs` is missing — they are flagged in Generation Notes for manual wiring.

- **Image slots cannot be wired to component properties** — Figma's Plugin API does not support URL binding. Document in Generation Notes instead.

- **Variable binding does not apply to image placeholders** — the placeholder rectangle is a visual guide, not a bound variable. Text slots follow `cc-figma-component` rules for variable binding (§7).

- **Same model as cc-figma-component for:** letter-spacing, rotation values in degrees (never radians), `setSharedPluginData` with `component_contracts` namespace, `eval` forbidden / use `new Function(code)()`, focus ring color conflicts, `atob` undefined.

---

## 8. Execution Modes

**Default (fast):** Run all phases sequentially without stopping. Present the Phase 0 plan and await approval, then execute Phases 1–6 without interruption.

**Debug mode:** Stop after each phase and await explicit approval before proceeding. Use when diagnosing issues. Activate by including "debug mode" in your prompt. Scripts are preserved in debug mode only — clean them up manually when done.

**Scripts location:** All working files MUST be written to `scripts/` subdirectory — never to the repo root.
