# String Diagram Studio: contributor guide

## Running the editor

The application is intentionally a single, directly openable `index.html`; it has no build step or local package dependencies. The live math preview loads MathJax 3 from a CDN, so preview typesetting needs network access. Diagram editing and LaTeX generation are handled by the page itself.

## Source layout

The HTML, CSS, application state, event handling, SVG rendering, and LaTeX generation all live in `index.html`. Keep the app self-contained: splitting the JavaScript into modules would break the current `file://` workflow unless the launch and packaging model changed too. The 8-by-7 startup diagram is stored in `string-diagram-studio.json` and mirrored in the embedded `default-diagram-source` JSON block because a page opened from `file://` cannot reliably fetch a neighboring file. The canvas starts at 70% zoom, which is also the reset zoom.

The script separates work into these flows:

1. Tool activation updates the mode, instructions, and pressed state.
2. Pointer and click handlers modify the plain-data diagram families.
3. `renderGrid()` draws grid cells, then calls `renderConnections()` for SVG connections and shape overlays.
4. `captureState()` / `restoreState()` copy persistent diagram data for undo and redo.
5. `latexForDiagram()` serializes shapes, connections, and nodes; `latexForConnection()` handles each connection family.
6. `diagramForSave()` / `validateSavedDiagram()` serialize and validate standalone JSON save files before `applySavedDiagram()` replaces the current diagram.

## State and data model

Grid coordinates start at the top-left. A cell index is row-major (`row * columns + column`); a node's visual center is `(column + 0.5, row + 0.5)`. In exported TikZ, the vertical axis is inverted with `rows - y`, so diagrams continue to read bottom-to-top.

| Variables | Purpose and shape |
|---|---|
| `columns`, `rows` | Current grid dimensions. |
| `cells` | Row-major array of operation keys or `null`. Structural node tools currently use keys such as `"bullet"` and `"text"`. |
| `points` | Set of cell indices explicitly marked as orientation points. |
| `textNodeValues` | Map from text-node cell index to its LaTeX source; an empty string is a valid empty node. |
| `bulletColors` | Map from bullet cell index to a color key; missing entries mean black. |
| `connections` | Array of `DiagramConnection` families. Ordinary wires have `kind: null`; evaluation/coevaluation segments have `kind: "evaluation"` / `"coevaluation"`; smooth vertical/horizontal strands have `kind: "smooth-line"` / `"smooth-horizontal"`; all ordinary connections have `from` and `to`. Multiplication has two `sources` and one `target`. Each endpoint is either a cell index, a box-border endpoint `{ boxId, x, y }`, or a circle port `{ circleId, side, x, y }`; all connection types can use shape ports. Optional `color` customizes appearance. `layer` separates normal and raised connections; array order is the stacking order within a layer, and raising moves that connection to the front of the raised stack. The editor and LaTeX exporter use this same order. Multiplication uses left/right LaTeX variants only when the target is horizontally aligned with a source. |
| `boxes` | Array of `{ id, x, y, width, height, color?, text? }` objects. Coordinates and dimensions are in grid units; `id` remains stable when the array is reordered or resized. |
| `circles` | Array of `{ id, x, y, radius, color?, text? }` objects. `x`, `y`, and `radius` are grid units; circle wire endpoints are represented by cardinal ports. |
| `nextBoxId`, `nextCircleId` | Monotonic IDs for newly created shapes, used by shape-port connections. |
| `operationDefinitions` | Labels and display metadata keyed by operation key. `diagramColors` maps the color keys to CSS and LaTeX names. |
| `selectionModeInstructions`, operation instruction strings, `curvedConnectionTypes`, `circleBoundaryCursorTolerancePx` | Shared user-facing descriptions, recognized curved segment kinds, and the pixel tolerance used for circle-boundary cursor feedback. |
| `undoStack`, `redoStack` | Snapshots of persistent diagram content. Selection, active tool, and transient pointer gestures are deliberately not part of a snapshot. |
| `activeOperationKey` | Operation currently being placed or used to draw a special connection; empty in selection and shape-drawing modes. |
| `boxMode`, `circleMode`, `textNodeMode`, `groupSelectMode` | Current shape/text placement and group-selection modes. Group selection uses a drag rectangle and highlights selected nodes and shapes in green; a box or circle is included only when the rectangle contains all its invisible ports. |
| `selectedCellIndex`, `selectedBoxIndex`, `selectedCircleIndex`, `selectedConnectionIndex` | At most one selected diagram item, represented by an index into its family. |
| `groupSelectedCells`, `groupSelectedEndpoints` | Transient group selection: occupied cell indices and invisible box-border/circle-port endpoints selected by the drag rectangle. Grid-edge insertion remaps both sets so the selection and move controls remain active. |
| `moveUpButton`, `moveLeftButton`, `moveDownButton`, `moveRightButton` | Directional toolbar controls; boxes selected without other nodes move by half a tile, while cell nodes and mixed selections move by one tile. Controls are disabled when a move would leave the grid, collide with an unselected cell node, or violate a multiplication's source-row/target-range constraints. |
| `multiplicationSelection` | In-progress multiplication endpoints, in selection order. |
| `boxLabelDraft`, `circleLabelDraft`, `textNodeDraft` | Uncommitted editor contents for the selected label/node. |
| `boxStart`, `boxPreview`, `boxStartPointer`, `boxMoved` | Box-drawing start/preview coordinates, original pointer position, and drag threshold state. |
| `groupSelectStart`, `groupSelectPreview`, `groupSelectStartPointer`, `groupSelectMoved` | Group-selection drag start/preview coordinates, original pointer position, and drag threshold state. |
| `wireStartEndpoint`, `wirePreview`, `wireStartPointer`, `wireStartConnectionKind`, `wireMoved` | Wire gesture endpoints, preview position, pointer position, optional evaluation/coevaluation/smooth-line/smooth-horizontal kind, and drag threshold state. `wireStartEndpoint` is an endpoint, not necessarily a cell index. |
| `suppressNextClick` | Prevents a completed drag from also being interpreted as a click. |
| `canvasZoom` | Display scale only; it does not change diagram coordinates or exported measurements. |
| `mathJaxReady`, `mathTypesetScheduled`, `mathTypesetRunning`, `mathTypesetAgain` | Track MathJax availability and coordinate asynchronous typesetting while coalescing repeated render requests. |
| `toolRegistry`, `toolShortcuts` | Registered toolbar controls and their keyboard activation functions. |

DOM references are named after their element IDs (for example, `grid`, `wireLayer`, and `activeInstructions`). They are view objects, not diagram data.

## Diagram save files

Use **Save diagram** below the canvas to download `string-diagram-studio.json`, or **Open diagram** to load a previously saved JSON file. Version 2 is sparse: `grid` stores dimensions, and `nodes` contains only occupied cells as `{ cell, type }` entries, with optional `label` or `color` fields. Orientation points use `type: "point"`. Connection endpoints are stored directly, so invisible grid nodes need no separate empty-cell records; shape ports are reconstructed from shape geometry. Boxes, circles, shape IDs, and every connection type, endpoint, color, and layer are stored. Selection, active editing state, and undo/redo history are not stored. The importer continues to accept version 1 files with dense `cells` arrays.

Opening a file validates the whole document before changing the current diagram. Unsupported versions, malformed values, invalid node/shape references, and invalid multiplication constraints are reported without replacing the current diagram. A successful open resets selection and both history stacks. The importer accepts files up to 10 MB and grids up to 100,000 cells.

## Adding a tool

The in-source tool registry keeps the toolbar button, keyboard shortcut, pressed state, and activation callback together while preserving the single-file workflow.

1. Add a button to the appropriate tool group in `index.html`. Give it a unique ID, an accessible name, and an `aria-pressed` attribute if it is a mode button. Keep its visible key hint and title consistent with its shortcut.
2. Cache the element with the other DOM references.
3. Add one entry to the registry array passed to `registerTool`, containing `id`, `button`, optional `shortcut`, and `activate`. Registration wires both click and keyboard activation and updates the button's `aria-keyshortcuts`.
4. Implement activation using `activateToolMode()` for the standard mode lifecycle: active operation key, category, label, instructions, selection clearing, pressed state, and rerender. Keep a specialized activation callback when the gesture is not a standard mode.
5. For a new node type, add its metadata to `operationDefinitions` and handle its placement and display in `renderGrid()`. Include any per-node data in `captureState()` / `restoreState()` and in `latexForDiagram()`.
6. For a new connection family, define its endpoint data shape, extend `connectionEndpoints()` and `mapConnectionEndpoints()` if it has a different topology, then add its interaction, SVG rendering, selection/deletion, and LaTeX serialization. For new LaTeX commands, document and define them in `diagram-output-template.tex`.
7. Test placement, selection, undo/redo, grid-edge insertion, and exported LaTeX. Add the feature's keyboard shortcut to its visible key hint and accessibility text.

Press **G** or use **Group select**, then drag across the grid to select cell nodes and invisible box/circle ports in the rectangle. On touchscreens, double-tap, hold the second tap, and drag to start a canvas drag gesture; a single-finger swipe pans the canvas. Straight connections in selection mode require dragging on desktop or double-tap-drag on touchscreens, so a normal tap continues to select nodes. In evaluation, coevaluation, and smooth-connection modes, drag between endpoints or click/tap two endpoints in order. In multiplication mode, click/tap the two source endpoints and then the target. Group selection is transient and visually highlighted in green.

Press **Delete** or **Backspace** to remove the current selection. Deleting a group removes its selected cell nodes and fully selected shapes, along with every connection attached to them.

The canvas toolbar's **Delete** button uses the same action, stays disabled without a deletable selection, and is highlighted red when one is active. On narrow phone layouts, text-node marker boxes shrink to fit within one grid cell.

The four move arrows translate cell nodes and mixed selections by one tile; a selection consisting only of boxes moves by half a tile. Cell indices, shape-port endpoints, and all dependent connections move together; multiplication constraints are checked before enabling each direction.

Placing a box or circle returns to selection mode, selects the new shape, and shows its LaTeX label editor.

**Trim canvas** removes empty outer rows and columns around cell nodes, numeric connection endpoints, boxes, and circles. It translates retained cell indices, shape coordinates, and shape-port endpoints together, and the operation is undoable.

The registry handles tool-button wiring and mode highlighting; it is not a runtime plug-in loader. New diagram semantics still need explicit interaction, rendering, history, and export support.

## LaTeX template reference

`diagram-output-template.tex` defines the named colors and TikZ macros emitted by the editor. Its final TikZ picture should match the startup diagram stored in `string-diagram-studio.json`. **Download LaTeX Environment** in the page header opens the complete standalone source containing the support commands and current diagram; the panel can copy the source or download it as a `.tex` file.

The page is intentionally usable from `file://`, so the LaTeX Environment panel reads its support code from the embedded `latex-support-source` text block in `index.html` rather than fetching the `.tex` file. Keep that block synchronized with the macro definitions in `diagram-output-template.tex`. The startup JSON is also embedded in `index.html` for direct-file use; keep the JSON file and embedded copy synchronized, and update the template picture when the startup diagram changes.

All connection macros draw 2pt colored strands with 4.5pt white underlays in their `O` variants. Most raised underlays are shortened by `0.12cm` at each end. The `\sLineO` and `\hLineO` underlays are deliberately unshortened: shortening only the white Bézier would change its endpoints, so it would no longer perfectly overlap the colored curve at crossings. Box and circle outlines also use 2pt strokes. All macros rely on TikZ's default line caps and joins; none applies rounded caps or joins.

Bullet nodes use 12px markers on the canvas and `0.09cm` radius in LaTeX so colored structural nodes remain visible; keep both sizes aligned.

Diagram coordinates use `x=1.5cm,y=1cm`; TikZ lengths specified directly in centimeters, such as circle radii and shortened underlays, remain physical lengths.

| Macro | Arguments and behavior |
|---|---|
| `\schwarz`, `\grau`, `\gruen`, `\blau`, `\orange` | Named colors corresponding to the editor's black, grey, green, blue, and orange choices. Keep these names synchronized with `diagramColors` in `index.html`. |
| `\diagramDrawMult` | Internal 7-argument multiplication renderer: target x-offset or literal `center`; TikZ style; left source x; source y; horizontal source span; signed vertical target displacement; color. |
| `\dMult` | `[\text{target offset or center}]{left x}{source y}{source span}{signed target height}{color}`. The optional target offset defaults to `center`. |
| `\dLMult` | Same argument order as `\dMult`, with default target offset `0`; renders the left-leaning source arc. |
| `\dRMult` | Same argument order as `\dMult`, with default target offset `1`; renders the right-leaning source arc. |
| `\dMultO`, `\dLMultO`, `\dRMultO` | Raised multiplication variants. They paint a white underlay, shortened by `0.12cm` at both ends, then draw the colored connection. |
| `\dPairing` | `{left x}{endpoint y}{horizontal span}{curve-height parameter}{color}{optional label}`. The curve opens upward in TikZ coordinates; its apex is at `y + height - 0.5`. |
| `\dCopairing` | Same arguments as `\dPairing`; the curve opens downward, with apex at `y - height + 0.5`. |
| `\dPairingO`, `\dCopairingO` | Raised variants that draw a white underlay shortened by `0.12cm` at both ends, then draw the colored pairing or copairing. |
| `\sLine` | `{x1}{y1}{x2}{y2}{color}`. Draws a cubic Bézier strand with control x-coordinates at the endpoints and y-coordinates interpolated by 45/55 and 55/45. |
| `\sLineO` | Same arguments as `\sLine`; draws an unshortened white underlay along the exact same curve, then the colored strand. Matching endpoints keep it centered at crossings. |
| `\hLine` | `{x1}{y1}{x2}{y2}{color}`. Draws a cubic Bézier strand with control y-coordinates at the endpoints and x-coordinates interpolated by 45/55 and 55/45. |
| `\hLineO` | Same arguments as `\hLine`; draws an unshortened white underlay along the exact same curve, then the colored strand. Matching endpoints keep it centered at crossings. |
| `\dLine`, `\dLineO` | `{x1}{y1}{x2}{y2}{color}`. Straight wire and its raised white-underlay variant. |
| `\dCurve`, `\dCurveO` | `{x1}{y1}{c1x}{c1y}{c2x}{c2y}{x2}{y2}{color}`. Cubic Bézier wire and its raised white-underlay variant. |

The editor uses the `O` suffix to indicate a connection raised above crossings. Every exported segment family has an O-variant: `dMult`, `dLMult`, `dRMult`, `dPairing`, `dCopairing`, `dLine`, `dCurve`, `sLine`, and `hLine`. The exporter emits only the selected command; each O-macro owns its white underlay and colored stroke.

The **Smooth V** and **Smooth H** connection tools use **V** and **H**, respectively. Their live SVG previews and final strands share the exported geometry: `\sLine` keeps control x-coordinates at the endpoints for vertical tangents, while `\hLine` keeps control y-coordinates at the endpoints for horizontal tangents. In both, the other control coordinates interpolate the endpoint values by 45/55 and 55/45.
