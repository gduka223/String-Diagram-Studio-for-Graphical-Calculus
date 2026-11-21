# String Diagram Studio: contributor guide

## Running and source layout

Open `index.html` directly in a browser. The application remains a single file with no build step or local package dependencies. MathJax 3 loads from a CDN for live mathematical labels; editing, JSON files and LaTeX generation work independently of that network request. For HTTP testing, run `python -m http.server 8766` in the project directory.

The left rail contains all node, shape and strand tools and the shared Item style controls. The canvas toolbar contains zoom, movement, mirror, selection, deletion, history and export actions. The startup diagram is 8 by 8; default and reset zoom are 70%.

HTML, CSS, application data, event handling, SVG rendering, JSON validation and LaTeX generation live in `index.html`. Keep the directly openable workflow when making changes: splitting JavaScript into imported modules would require revisiting packaging and `file://` support.

Two embedded blocks have standalone counterparts:

- `default-diagram-source` mirrors `string-diagram-studio.json`.
- `latex-support-source` mirrors the preamble and macro definitions in `string-diagram-studio.tex`. The standalone file also contains the startup picture.

The embedded blocks are used at runtime because browsers cannot reliably fetch neighbouring files under `file://`. Keep the JSON copies, support code and standalone startup picture synchronized. **Download LaTeX Environment** generates complete source for the current diagram rather than fetching the standalone file.

The principal flows are:

1. A registered tool activates a mode and updates instructions and pressed states.
2. Click/pointer handlers change plain-data diagram families after taking an undo snapshot.
3. `renderGrid()` creates grid cells; `renderConnections()` draws shapes, strands, previews and overlays.
4. `scheduleMathTypeset()` coalesces asynchronous MathJax work. `scheduleArrowHeadRender()` refreshes arrowheads after label layout and zoom changes.
5. `latexForDiagram()` emits shapes, invisible label measurements, ordered strands and visible labels/nodes. `latexForConnectionCommand()` chooses the strand macro; `latexForConnection()` adds text-clearance measurements.
6. `diagramForSave()` / `formatDiagramSave()` serialize JSON. `validateSavedDiagram()` validates the entire file before `applySavedDiagram()` replaces the diagram and resets selection/history.

## Coordinates and state

Grid coordinates start at the top-left. A cell index is row-major: `row * columns + column`. A cell-centred node is at `(column + 0.5, row + 0.5)`.

**Editor cells are square.** `updateGridDimensions()` uses aspect ratio `columns / rows`, independently of export dimensions. Keep SVG viewBoxes, hit testing, shape ports and overlays in these same grid units. Equal-width/height boxes are square and circles are round at every zoom level.

**TikZ retains stretched axes:** `latexXScale = 1.5` cm and `latexYScale = 1` cm per grid step. Vertical coordinates are inverted by `rows - y`. Circle radii are physical centimetres in TikZ; `latexPoint()` adjusts circle-port coordinates at export so strands meet the circle boundary. Do not apply the LaTeX scale factors to the editor grid.

| State | Purpose |
|---|---|
| `columns`, `rows`, `cells` | Dimensions and dense row-major node types or `null`. |
| `points` | Set of faint blue reference/orientation nodes. Their appearance is fixed and they are omitted from LaTeX. |
| `textNodeValues`, `textNodePositions` | Text expressions and optional half-cell positions away from the cell centre. Empty text is valid. `endpointFor()` resolves the actual position. |
| `bulletColors` | Bullet colour by cell; missing entries mean black. Bullet size is fixed. |
| `connections` | Ordinary `{ from, to, kind, ...appearance }` strands or `{ kind: "multiplication", sources: [a, b], target, ...appearance }`. |
| `boxes`, `circles` | Shapes with stable IDs, grid geometry, optional text, colour and stroke appearance. |
| `nextBoxId`, `nextCircleId` | Monotonic IDs. Shape-port connections refer to IDs rather than array indices. |
| `creationAppearance` | Defaults for new objects: colour, line style, width, arrows and arrow side. |
| `selectedCellIndex`, `selectedBoxIndex`, `selectedCircleIndex`, `selectedConnectionIndex` | Individual selection. |
| `groupSelectedCells`, `groupSelectedEndpoints` | Transient selection of nodes and complete shapes. |
| `diagramClipboard`, `lastGridPointer` | Internal copied fragment and last canvas pointer position used by paste. |
| `undoStack`, `redoStack` | Diagram snapshots; selection, active tools and transient gestures are excluded. At most 100 pre-edit snapshots are retained. |
| `toolRegistry`, `toolShortcuts` | Buttons, shortcuts and activation callbacks. |
| `wireStartEndpoint`, `wirePreview`, `wireStartConnectionKind`, `wireMoved` | Strand gesture state; shape endpoints are supported as well as cell indices. |
| `boxLabelDraft`, `circleLabelDraft`, `textNodeDraft` | Uncommitted label edits, included in live previews and JSON saves. |
| `canvasZoom` | Display scale only; saved/exported measurements do not change. |

Endpoints are numeric cell indices, `{ boxId, x, y }`, or `{ circleId, side, x, y }`. Box ports snap to half-cell positions on the boundary. Circle ports are `north`, `south`, `east` or `west`, at the editor radius. The importer also accepts older east/west coordinates divided by `latexXScale` and normalizes them.

Ordinary connection kinds are `null` (straight), `evaluation`, `coevaluation`, `smooth-line` (vertical endpoint tangents) and `smooth-horizontal` (horizontal endpoint tangents). Multiplication has three endpoints. Use `connectionEndpoints()` and `mapConnectionEndpoints()` to traverse/remap every topology.

`layer: "up"` draws a strand above crossings with a white underlay. Normal strands draw first; array order determines stacking within each layer. Raising a strand moves it to the end of the raised stack. SVG and TikZ share this ordering.

## Tools, selection and transforms

Selection mode (`Space` or `Escape`) creates a blue reference node on an empty-cell click, selects objects individually and selects groups by dragging a rectangle. A shape joins a group only when all its invisible ports are inside. The explicit Group select shortcut (`G`) is also registered. Selection feedback is transient and does not become a saved style.

The Line tool (`L`), evaluation (`E`), coevaluation (`C`), Smooth V (`V`) and Smooth H (`H`) support two endpoint clicks or dragging. Missing reference nodes can be created at empty grid endpoints. On touchscreens, a single-finger swipe pans; double-tap, hold the second tap and drag to draw or select.

**Evaluation and coevaluation require distinct endpoints at the same height.** Invalid click pairs, drops, imports and moves are rejected. During a drag, a dashed horizontal guide marks the required height. An invalid preview turns red and displays “End at the same height”; it does not silently create a different connection on release. Tooltips, instructions and rejection messages describe this constraint.

Multiplication (`M`) takes two distinct sources at the same height and a target between their x-coordinates on a different height. `connectionGeometryIsValid()` centralizes these geometry constraints for interaction, movement and validation.

Boxes (`X`) are drawn by dragging; circles (`O`) are placed in a cell. Placement returns to selection and shows the shape's label editor. Bullets (`B`) and text nodes (`T`) use cell placement. The text icon is a rounded box with a T; the box icon has a solid outline. Mirror buttons use icons, accessible names and hover titles.

Move arrows translate text-only and box-only selections by half a cell; other node and mixed selections move by one cell. Cell indices, text positions, shape ports and dependent connections move together. Controls disable directions that leave the grid, collide with unselected nodes or violate connection geometry.

`Ctrl/Cmd+C` copies the selected fragment into the editor's internal clipboard. `Ctrl/Cmd+V` pastes near the last canvas pointer position, clamped to the grid. Paste assigns new shape IDs, preserves appearance and internal connections, rejects collisions and selects the pasted items. Native text copying remains available in input fields.

Mirroring affects the whole diagram. Top/bottom reflection exchanges evaluation/coevaluation and north/south ports; left/right reflection exchanges east/west ports. Text remains readable and arrow direction remains attached to the same logical start/end. Both operations are undoable.

Delete/Backspace and the toolbar Delete action remove the selection and attached connections. Grid-edge insertion remaps all dependent data. **Trim canvas** removes empty outer rows/columns and translates retained geometry together. Clear, delete, move, trim, mirror, paste and appearance changes participate in undo/redo.

## Appearance controls

`selectedStyleTargets()` declares what a selection can edit. Group styling includes a strand only when all its endpoints belong to selected nodes/shapes. **Only the control the user changes is applied:** changing line style must preserve mixed widths, arrow directions and enabled states.

| Object | Colour | Line style and width | Arrows |
|---|---|---|---|
| Ordinary strands | Yes | Yes | Start, end or both ends |
| Multiplication | Yes | Yes | No |
| Boxes/circles | Yes | Yes | No |
| Bullets | Yes | No; fixed 12px diameter / 0.09cm TikZ radius | No |
| Reference nodes | No; always faint blue | No | No |
| Text nodes | Label editor | No | No |

Reference nodes are protected in individual and group styling. Legacy JSON point colours are accepted, discarded on load and omitted from new saves. Their group selection is indicated by the cell without recolouring or resizing the marker. Bullet selection does not change its diameter or stroke appearance.

Solid/dashed/dotted choices are SVG icon buttons with accessible names, tooltips and `aria-pressed`. They update the hidden `stroke-style` backing select and dispatch its change event. A mixed selection has no pressed style icon. Controls with no eligible target are disabled or hidden. With no selection they edit creation defaults.

Supported widths are string values `"0.4"`, `"0.8"`, `"1.2"`, `"2"` and `"3"` (pt in TikZ). Strand appearance includes `lineStyle`, `strokeWidth`, `arrow` and `arrowSide`. The side is `"start"`, `"end"` or `"both"`; `arrow: false` disables arrowheads independently of the stored side.

## Arrowheads and text clearance

Both views use a small filled **Stealth** head. `arrowHeadDimensions()` defines shared proportions in points for stroke width `w`: length `3.5 + 1.25w`, width `2.6 + w`, inset `0.7 + 0.15w`. The editor uses 1.1 screen pixels per point, matching its stroke scale. This replaces the former large triangles measured as fractions of a cell. TikZ loads `arrows.meta` for the matching shape.

`appendConnectionArrowHead()` samples the SVG path tangent at each enabled end. At a text endpoint it finds the first exit from the actual label rectangle, including a 2px gap, and puts the head there. `scheduleArrowHeadRender()` coalesces refreshes after rendering, zoom, resize and MathJax label layout, keeping heads small and clear of labels.

TikZ measures text in TeX rather than inferring dimensions from browser pixels. `latexForDiagram()` first emits invisible named nodes (`text-<cell>`) with the same expressions and padding as the final labels. For arrow ends attached to nonempty text, `latexForConnection()` emits `\diagramArrowTrim` in the strand's inward tangent direction. `shorten <=` / `shorten >=` use the measured boundary distance plus 1pt. Visible white-backed labels are drawn last. This handles arrivals from above/below, either side and diagonally, including tall/wide formulas.

Cap/cup export orders endpoints left-to-right. `latexStyleArgument()` reverses single-arrow direction and shortening sides when required; both-ended arrows retain both tips. Smooth curves with a zero endpoint tangent export as the equivalent straight segment so TikZ can orient and shorten their arrows correctly.

## JSON files

Save downloads sparse version 2 `string-diagram-studio.json`. `grid` stores dimensions; `nodes` contains occupied cells as `{ cell, type }`, optionally with text `label`, bullet `color` or explicit text `x`/`y`. Reference entries use `type: "point"` with no appearance. Shapes, IDs and every supported connection endpoint, colour, layer, stroke and arrow setting are preserved. Selection, clipboard, creation defaults and history are not saved.

Version 1 dense `cells` files remain supported. The importer validates indices, types, labels, half-cell positions, shape IDs/geometry, port references, appearance values and geometry constraints before replacing the current diagram. Invalid files leave current work intact. Limits: 10 MB per file, 1000 per axis, 100,000 total cells and 100,000 connections. Successful open resets selection and both history stacks.

## LaTeX reference

Simplified strand macros share this prefix:

```tex
\command[<TikZ options>]{line width in pt}{color}{geometry...}
```

Options default to `solid`. The exporter adds dotted/dashed styles, an explicit Stealth shape, `<-` / `->` / `<->` and text shortening. Palette commands are `\schwarz`, `\grau`, `\gruen`, `\blau` and `\orange`.

| Macro family | Geometry after width and colour |
|---|---|
| `\dCap`, `\dCup` | `{left x}{endpoint y}{horizontal span}`. Endpoints share y. Control-point displacement is 0.6 times the span, upward for cap/downward for cup in TikZ coordinates. |
| `\dLine` | `{x1}{y1}{x2}{y2}`. |
| `\sLine`, `\hLine` | `{x1}{y1}{x2}{y2}`. Vertical/horizontal endpoint tangents; other control coordinates interpolate by 45/55 and 55/45. |
| `\dMult`, `\dLMult`, `\dRMult` | `{left source x}{source y}{source span}{signed target height}{target x offset}`. L/R variants align the target with a source. |
| `\dBull` | `{x}{y}{color}` only; fixed filled bullet. |
| `\diagramArrowTrim` | `{output length macro}{named text node}{distant physical x in cm}{distant physical y in cm}`. |

Every strand family has an `O` variant with a 4.5pt white underlay. Most underlays shorten by 0.12cm at both ends. Smooth underlays are unshortened so the white and coloured Béziers overlap exactly. Appearance options, including arrows and text clearance, apply to the coloured stroke. Keep SVG and macro geometry synchronized.

The previous `dPairing`, `dCopairing`, `dCurve` and old argument signatures are superseded. When inserting new picture code into an existing document, update its support commands from **Download LaTeX Environment** too.

## Extending the editor

1. Add an accessible left-rail button with unique ID, tooltip, pressed state and consistent shortcut hint; cache its DOM reference.
2. Register button, shortcut and activation callback through `registerTool()`. Use `activateToolMode()` when appropriate.
3. Define data/metadata. Extend endpoint helpers for a new topology, or node maps for new per-node data.
4. Update placement, hit testing, rendering, selection/deletion, moves, grid remapping, trim, mirror, clipboard, snapshots and save/import paths as appropriate. Appearance eligibility belongs in `selectedStyleTargets()`.
5. Add matching LaTeX serialization and macros. Preserve text clearance and arrow direction when reordering geometry.
6. Synchronize embedded and standalone `.tex` / `.json` sources. Preserve direct-file operation.

The registry wires internal tools; it is not a runtime plug-in loader. DOM references are view objects rather than persistent data.

## Verification

Chromium/Playwright checks for this update covered square cells at desktop/phone sizes and different zooms, grid growth, both-ended arrows and JSON round-trips, mixed group styles, immutable reference markers, colour-only fixed-size bullets, cap/cup drag feedback and rejection, text clearance, round circles/legacy ports, half-cell text moves, mirroring, copy/paste and undo. No JavaScript runtime errors occurred.

The startup diagram and an arrow fixture with tall/wide formulas, both arrow directions and reversed cup/cap ordering were exported and compiled with `pdflatex`. Inspect the PDF when changing arrow geometry or clearance: successful compilation alone does not establish visual correctness.

For future changes, exercise the affected browser workflow and its saved/exported result. Include mixed selections for control changes and all endpoint families for coordinate changes. Check direct-file and HTTP loading when changing resource handling. MathJax network failure must not prevent editing/exporting.

## Licence

`LICENSE.md` contains Creative Commons Attribution-NonCommercial 4.0 International. Preserve the existing licence and attribution in modified distributions.
