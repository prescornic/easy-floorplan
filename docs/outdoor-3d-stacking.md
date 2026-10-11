# Outdoor Courtyard, Multi-Floor 3D Stacking & Roof Technical Specification

Technical design specification for implementing outdoor courtyard areas, multi-floor vertical 3D stacking (leveraging Home Assistant native floors), custom outdoor zone assignments, and roof rendering with ambient daylight integration.

---

## 1. Feature Architecture Overview

```
                        +--------------------------------+
                        |  Home Assistant Floor Registry |
                        |  (Native HA Floor Ordering)    |
                        +---------------+----------------+
                                        |
                                        v
+------------------------+  +------------------------+  +------------------------+
| Courtyard & Outdoor    |  | Multi-Floor 3D Stack   |  | Roof & Elevation       |
| Area Layer             |  | Elevation Calculation  |  | Geometry Engine        |
| - Custom Outdoor Zones |  | - Native HA Z-Offsets  |  | - Gable / Hip / Flat   |
| - Plant & Light Sensors|  | - Stacked Wall Envelope|  | - Roof Pitch & Shading |
+-----------+------------+  +-----------+------------+  +-----------+------------+
            |                           |                           |
            +---------------------------+---------------------------+
                                        |
                                        v
                        +--------------------------------+
                        |  Ambient Daylight & Shadow     |
                        |  Polygon Generator             |
                        |  (Courtyard & Multi-Story)     |
                        +--------------------------------+
```

---

## 2. Core Components & Module Breakdown

### 2.1 Data Models & Configuration (`src/types.ts`)

- **Outdoor & Courtyard Layer**:
  - Add optional `isOutdoor?: boolean` or `kind?: "courtyard" | "indoor"` to floor structures.
  - Allow defining custom outdoor zones (e.g. Lawn, Patio, Swimming Pool, Terrace, Driveway).
- **Native HA Floor Stacking**:
  - Use Home Assistant's native floor ordering/hierarchy (`floor_id` / floor index) to establish vertical ordering without duplicate manual indices.
  - Optional `elevationOffset?: number` per floor to accommodate non-standard ceiling heights.
- **Roof Specifications**:
  - Define `RoofConfig` interface supporting roof type (`gable`, `hip`, `flat`), pitch angle, overhang, and color/texture fallbacks.

### 2.2 3D Projection & Elevation (`src/projection.ts` & `src/projection-joints.ts`)

- **Cumulative Z-Base Calculation**:
  - Update `DisplayFrame` and `IsoWallInput` to compute baseline Z-offsets (`z0`, `z1`) per floor based on HA floor order:
    \[
    z_0(f) = \sum_{k < f} \text{wallHeight}(k)
    \]
  - Stack exterior and interior walls across visible floors to render a complete multi-story building shell under isometric (`projection: "iso"`) view.
- **Roof Solid Generation**:
  - Implement `roofSolids(...)` returning `IsoSolid` polygons with sloping triangular/quadrilateral faces for gables and hips.

### 2.3 Ambient Daylight & Shadow System (`src/ambient-daylight.ts` & `src/render.ts`)

- **Courtyard Surface Shading**:
  - Extend daylight polygon calculation so sun shadows cast by multi-story walls, roof eaves, and outdoor fences render naturally onto courtyard ground surfaces.
- **Light Occlusion across Levels**:
  - Ensure lamp light and outdoor sunlight interact correctly with elevated balconies, railings, and stacked floor slabs.

### 2.4 Interactive Visual Editor (`src/editor.ts` & `src/editor-forms.ts`)

- **Courtyard & Zone Tools**: Add drawing tools and form specifications for creating outdoor boundaries and styling zone fills.
- **Roof Property Editors**: Provide controls for toggling roof visibility, selecting roof style, and setting pitch/overhang options.

---

## 3. Implementation Status & Feature Matrix

All core phases have been fully implemented, verified, and integrated into `easy-floorplan`:

- [x] **Phase 1 — Outdoor Courtyard & Custom Zones**:
  - `isOutdoor?: boolean` floor toggle in config and visual editor.
  - Outdoor area zone kinds (`lawn`, `patio`, `terrace`, `driveway`, `garden`, `pool`) with distinct background fills and styling.
  - External boundary wall extraction: `filterExternalIndoorWalls(...)` and `filterExternalOpenings(...)` to isolate building shells.
- [x] **Phase 2 — Multi-Floor 3D Stacking via HA Floors**:
  - `resolveFloorLevel(...)` detects floor elevation from Home Assistant native floor registry (`haFloor`) and floor array order.
  - Level-based vertical stacking: $z_{\text{offset}} = (L - L_{\text{base}}) \times \text{outdoorWallHeight}$.
  - Shifted 3D solids via `shiftIsoSolidZ(solid, zOffset)` for exterior walls, doors/windows, floor slabs, and roofs.
  - Full-height door and window wall headers/lintels above openings (`z \in [sill + height, wallHeight]`).
  - Inter-floor slab solids ($Z \in [zOffset - 6, zOffset]$) for upper levels with automatic dead-space boundary fallbacks.
  - Cumulative multi-story canvas headroom scaling in `_frame(...)`.
- [x] **Phase 3 — Extended Ambient Daylight & Building Shadows**:
  - Outdoor direct sunlight (`renderOutdoorSunlight`) providing ambient sky daylight across open ground.
  - Directional building wall shadows (`sunShadowPolygon`) cast across the outdoor floor based on sun azimuth (`dir`) and elevation (`reach`).
  - Suppression of indoor window beam cutouts on outdoor ground surfaces.
- [x] **Phase 4 — Roof Geometry Engine & Interactive Focus**:
  - 3D Roof solids supporting `flat`, `gable`, `hip`, and `shed` (with `top`, `bottom`, `left`, `right` slope directions).
  - Configurable roof pitch, elevation, overhang, and color picker (`ha-selector-color`) in editor properties.
  - Scope constraint: 3D roofs render exclusively on outdoor floor views (`isOutdoor: true`), keeping indoor floor views unobstructed.
  - Interactive 3D roof buttons (`.fp-iso-roof-button`) allowing clicking on roofs to zoom into rooms.
  - **Dynamic Roof Auto-Hide**: Zooming or focusing into any area (`_zoomedAreaId === area.id`) automatically hides its roof, exposing internal room furniture, lights, state badges, and sensors.

---

## 4. Testing & Quality Assurance

- **Node Unit Tests (`npm test`)**:
  - Validates `shiftIsoSolidZ` and multi-story Z-offset transformations.
  - Validates roof polygon generation for flat, hip, gable, and directional shed roofs in `src/projection.test.ts`.
  - Validates `renderOutdoorSunlight` building shadow polygons and sun reach calculations.
- **Browser Tests (`npm run test:browser`)**:
  - Real Chromium tests validating interactive 3D roof buttons, gesture capture, floor switching, and zoom focus.
- **Type Checking (`npm run typecheck`)**:
  - Verified 0 errors across dual TypeScript projects (`tsconfig.json` and `tsconfig.node.json`).

---

## 5. Architectural Deep-Dive: Implemented Functionalities

### 5.1 Multi-Level 3D Building Shell & Slabs
When viewing an outdoor floor (`isOutdoor: true`), `_renderIsoLayer` groups indoor floors by their resolved HA floor level. For every level above the outdoor ground plane ($L > L_{\text{base}}$):
1. **Floor Slabs**: Extrudes horizontal slab boxes ($Z \in [zOffset - 6, zOffset]$) using either the level's area polygons or boundary wall loops (`deadSpaces(lvlWalls, [])`).
2. **Exterior Walls & Openings**: Filters out internal partition walls and draws full-height 2.5m outdoor walls with lintels spanning above door/window openings.
3. **Roofs**: Extrudes roof geometry at the top elevation of each level.
4. **Depth Ordering**: `sortIsoSolids` sorts solids by Z-level and isometric depth, guaranteeing lower walls render first, followed by floor slabs, upper walls, and roofs.

### 5.2 Dynamic Roof Auto-Hide on Area Focus
To inspect what is inside a covered area (such as a patio or a room under a roof) without toggling configuration:
- In 3D isometric mode, clicking on a 3D roof solid triggers `_onAreaAction` and focuses into the area (`this._zoomedAreaId = area.id`).
- When an area is focused, `_renderIsoLayer` filters that area out from `roofSolids(unzoomedAreas(areas), ...)`.
- The roof seamlessly disappears and the camera zooms in, revealing all indoor details, furniture, sensors, and state badges.
- Tapping the area again, clicking the zoom reset button, or pressing <kbd>Esc</kbd> unzooms and restores the roof.

### 5.3 Outdoor Direct Sunlight & Building Wall Shadows
Indoor floors use window light beams cutting through shade masks. Outdoor floors instead use `renderOutdoorSunlight`:
- The courtyard/patio ground remains in open sunlight.
- The external building envelope walls project shadow polygons (`sunShadowPolygon`) swept along the sun's travel vector (`dir = sunLightDirection(cfg, azimuth)`).
- Shadow reach scales with sun elevation (`reach = min(w, h) * sunReachFraction * sunReachScale(elevation)`), causing shadows to realistically lengthen at dawn/dusk and contract at noon.

---

## 6. Configuration Guide & Practical YAML Patterns

### 6.1 Full Floor Roof with Sub-Area Focus (The Encompassing Roof Pattern)

When a floor (e.g. Ground Floor or 1st Floor) has multiple individual rooms (such as Living Room and Kitchen), you may want to generate a single unified roof (e.g., hip or gable) over the entire building footprint while still being able to focus on individual rooms and see their labels.

**Key Rule**: Place the encompassing roof area **first in the `areas` array** in YAML.

```yaml
floors:
  # ---------------------------------------------------------------------------
  # INDOOR FLOOR (e.g. Ground Floor)
  # ---------------------------------------------------------------------------
  - id: ground
    name: Ground floor
    haFloor: '1'          # Home Assistant floor ID
    walls:
      - { id: w1, x1: 100, y1: 100, x2: 900, y2: 100 }
      - { id: w2, x1: 900, y1: 100, x2: 900, y2: 500 }
      - { id: w3, x1: 900, y1: 500, x2: 100, y2: 500 }
      - { id: w4, x1: 100, y1: 500, x2: 100, y2: 100 }
      - { id: w_div, x1: 500, y1: 100, x2: 500, y2: 500 } # Interior partition wall
    areas:
      # 1. ENCOMPASSING ROOF AREA (MUST BE FIRST)
      # Covers the entire perimeter of the building and defines the 3D roof.
      # Because it is first, it renders beneath room labels and allows room clicks
      # on individual sub-areas that sit on top.
      - id: whole_house_roof
        name: House Roof
        showName: false        # Hide label so it doesn't overlap room names
        hasRoof: true
        roofType: hip          # hip, gable, shed, or flat
        roofPitch: 40
        roofElevation: 60
        roofOverhang: 12
        roofColor: '#8d6e63'
        points:
          - { x: 100, 'y': 100 }
          - { x: 900, 'y': 100 }
          - { x: 900, 'y': 500 }
          - { x: 100, 'y': 500 }

      # 2. INDIVIDUAL ROOM AREAS (AFTER THE ENCOMPASSING AREA)
      # These areas retain their individual colors, labels, sensors, and focus zoom.
      - id: living_room
        name: Living Room
        showName: true
        color: var(--fp-color-living)
        opacity: 0.15
        points:
          - { x: 100, 'y': 100 }
          - { x: 500, 'y': 100 }
          - { x: 500, 'y': 500 }
          - { x: 100, 'y': 500 }

      - id: kitchen
        name: Kitchen
        showName: true
        color: var(--fp-color-kitchen)
        opacity: 0.15
        points:
          - { x: 500, 'y': 100 }
          - { x: 900, 'y': 100 }
          - { x: 900, 'y': 500 }
          - { x: 500, 'y': 500 }

  # ---------------------------------------------------------------------------
  # OUTDOOR FLOOR (3D View of entire house + courtyard)
  # ---------------------------------------------------------------------------
  - id: outside
    name: Outside & Courtyard
    haFloor: '0'          # Base level
    isOutdoor: true       # Activates outdoor mode: renders 3D building stack + roofs + shadows
    walls:
      - { id: fence_1, x1: 0, y1: 0, x2: 1000, y2: 0 }
    areas:
      - id: patio_zone
        name: Patio
        showName: true
        zoneKind: patio
        hasRoof: true          # Patio can have its own shed/pergola roof
        roofType: shed
        roofPitchSide: top
        roofPitch: 20
        roofElevation: 60
        roofColor: '#a1887f'
        points:
          - { x: 800, 'y': 460 }
          - { x: 1000, 'y': 460 }
          - { x: 1000, 'y': 600 }
          - { x: 800, 'y': 600 }

      - id: lawn_zone
        name: Garden Lawn
        showName: true
        zoneKind: lawn
        points:
          - { x: 0, 'y': 0 }
          - { x: 1000, 'y': 0 }
          - { x: 1000, 'y': 460 }
          - { x: 0, 'y': 460 }
```

### 6.2 Benefits of this Setup
1. **Full Building 3D Exterior**: When viewing the Outside floor in 3D isometric view, the card renders the stacked building shell with its complete hip/gable roof as well as the patio shed roof.
2. **Indoor View Integrity**: When switching to the Ground floor view, roofs are automatically excluded so rooms remain open and visible.
3. **Interactive Inspection**: Clicking on the patio's 3D roof zooms directly into the patio, hiding its roof and revealing internal furniture, heaters, or lighting sensors.
4. **Layering & Hit-Testing**: Placing `whole_house_roof` at index 0 ensures that individual room areas (`living_room`, `kitchen`) take precedence for tap gestures, room focus cycling, and label centering.
