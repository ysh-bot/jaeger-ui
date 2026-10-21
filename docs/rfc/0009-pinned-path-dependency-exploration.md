# RFC 0009: Pinned-Path Exploration for the Service Dependency Graph

* **Status**: Draft
* **Created**: 2026-10-21
* **Last Updated**: 2026-10-21

---

## TL;DR

Jaeger UI's System Architecture page draws the whole service dependency graph and lets the user narrow it to a focal service and a hop count. That answers "what does the system look like" and stops there. An incident responder asks different questions: which of this service's dependencies is worse than yesterday, which callers share a path with the one that is failing, and what changed. This RFC inventories a reference dependency explorer built for incident response, compares it capability by capability with Jaeger's two dependency views, and proposes bringing the parts that Jaeger's data can support into the System Architecture page. The core change is a shift from *filtering a whole-graph drawing* to *pinning services and drawing the path through them*: a search pane that pins, a drawing of the pins with their callers above and callees below folded into one group per side, a details card for the selection, and a time-comparison overlay on every drawn call. Layout stays on Graphviz `dot` through Plexus for the first milestones, which keeps this RFC independent of the rendering-library decision in RFC 0008 [\[1\]](./0008-graph-visualization-stack.md).

---

## Context & Problem

Jaeger UI has two dependency views.

- **System Architecture → DAG** (`packages/jaeger-ui/src/components/DependencyGraph`). The page fetches the aggregated dependency links for the configured lookback and draws every service as a node and every parent-child pair as an edge labelled with its call count. The user can pick a hierarchical (`dot`) or force-directed (`sfdp`) layout, choose a focal service and a depth, search by name, and open a two-item menu on a node (set focus, view traces). Above a configured service count the hierarchical layout is disabled and a focal service becomes mandatory.
- **Deep Dependency Graph** (`packages/jaeger-ui/src/components/DeepDependencies`). The view takes a focal service and optional operation, builds the graph from traced paths, and offers hop selection upstream and downstream, five density modes, an operations toggle, a node menu (set focus, view traces, focus paths through this node, hide node, view parents, view children), per-node decorations from an external source, and a side panel for the decoration detail.

Both views are maps. Neither tells the user what is *wrong*, because neither compares the graph with an earlier point in time, and neither ranks what it shows by anything other than structure. The whole-graph DAG also stops being readable well before the service count that most production deployments have, which is why the page falls back to a force-directed layout and then to a mandatory focal service.

The reference design examined for this RFC is an internal dependency explorer built for incident responders over a service graph of about 770 services and 9,600 observed calls. Its authors tried the whole-graph drawing first and removed it, with the note that nobody read it. What replaced it is the subject of this RFC.

---

## Scope

This RFC covers the System Architecture page. It proposes a new exploration mode for that page and the data the query service must expose to support it. It does not change the trace-level views, and it does not decide the rendering library; RFC 0008 [\[1\]](./0008-graph-visualization-stack.md) owns that question, and the design here is expressed so that it works on either outcome.

---

## Part 1: Inventory of the Reference Design

The inventory is grouped by the question each capability answers. Where a capability depends on data Jaeger does not have, the dependency is named, because that decides what transfers in Part 3.

### 1.1 Rendering and layout

The reference design uses no graph library. The view is a SolidJS component that emits SVG, and the layout is a hand-written Sugiyama pipeline of about 1,500 lines of TypeScript:

1. **Cycle breaking** by the Eades, Lin and Smyth feedback-arc-set heuristic [\[2\]](https://doi.org/10.1016/0020-0190(93)90079-O), so a service sits below its callers wherever the cycles allow.
2. **Layering** by longest path from the top over the measured call edges, with a caller that nothing calls placed directly above what it calls rather than at the top.
3. **Crossing reduction** by eight barycenter sweeps over the nodes *and* the dummy nodes that long edges pass through, followed by Gansner's transposition step [\[3\]](https://doi.org/10.1109/32.221135). Positions are fractions of the row rather than indices, so a node with eight neighbours in a wide row and a node with one neighbour in a narrow row compare on one scale.
4. **Coordinate assignment** by the network simplex method of Gansner et al. [\[3\]](https://doi.org/10.1109/32.221135) §4.2, implemented after dagre's version. Every call becomes an auxiliary node with two zero-length edges to its endpoints, weighted 1 for a call, 2 where one end is a dummy node, and 8 between two dummy nodes, so a long edge becomes one straight vertical and a node sits at the weighted median of its neighbours. The authors replaced Brandes–Köpf with this because the four-alignment average let long edges wander.
5. **Edge routing** as straight segments through the dummy-node stops with a 10 px rounded corner at each bend. Every call into a service ends in one 34 px vertical into the top of its box, so several callers read as one incoming arrow. A back edge climbs beside its callee, turns above it, and enters from the top like every other call.
6. **Disconnected components** are placed side by side with a fixed gap, and the drawing is centred in the scroll pane.

Two interaction features sit on top of the layout. **Animated re-layout** moves each node that survives a layout change from its old position to its new one, measured relative to the selected node, and fades in nodes that are new; edges redraw in place and fade in behind, and the animation honours `prefers-reduced-motion`. **Drag to pan** scrolls the pane in both directions on a press-and-drag on the background, and a drag longer than a few pixels swallows the click that ends it so a pan never pins a node by accident.

The only third-party drawing dependency in the application is Mermaid, used for diagrams on its documentation page, not for the graph.

**What this teaches for Jaeger.** The pipeline is textbook, and that is the point: nothing in it is specific to the domain. Six lessons carry over, and §3.2 builds on them.

1. **The transform earns the readability, not the layout.** The reference design's history is ten days of changes to *what is drawn* (fold the fan, absorb pass-through nodes, drop declared-only calls, lift shared neighbours into strips) and three changes to *how it is placed*. Any layered engine draws a 30-box graph well; no engine draws a 700-node graph with a 400-node strongly connected component readably. Jaeger's investment belongs in the pinned-path transform of §3.1, which is pure, testable and renderer-independent.
2. **Edge conventions do more for legibility than node placement.** Every call leaves the caller's bottom and enters the callee's top through one shared vertical, so a fan-in reads as one arrow; a long edge is one straight line; a back edge still enters from the top. Graphviz expresses most of this directly: `rankdir=TB`, `splines=polyline`, and `tailport=s` / `headport=n` on every edge. The one convention it does not enforce, a perfectly straight corridor, is the row Option A loses in §3.2 and is cosmetic.
3. **Structure comes from measured edges only.** Layering, ordering and the caller and callee lists are computed from calls that were observed; anything else that is drawn (a declared permission, an ownership link) is routed around that structure and never moves a node. For Jaeger this means the Dependencies edges decide the layout, and any overlay from decorations or SPM colours what is already placed.
4. **Determinism is a feature.** Ties in the crossing sweep break alphabetically, components are ordered by their smallest member, and the same pins produce the same picture on every reload; that stability is also what gives the re-layout animation an anchor. `dot` is deterministic for a given input order, so the transform must sort nodes and edges before handing them to Plexus.
5. **Graph transforms, not layout features, handle cycles.** The feedback-arc-set pass only decides which way a back edge points; folding a mutual call into one double-headed edge and dropping self-loops happen before layout. `dot` breaks cycles itself but would draw a mutual call as two edges, so the fold stays in the transform.
6. **Owning layout code is a recurring cost.** The coordinate assignment was rewritten twice in a week (Brandes–Köpf, then network simplex) and the edge router once (Manhattan rails, then straight segments), each time to fix a visual defect. RFC 0008 [\[1\]](./0008-graph-visualization-stack.md) names the same cost for Plexus. Jaeger should not add a second layout engine it owns; §3.2 keeps `dot` for that reason.

### 1.2 What is drawn: the pinned path

The drawing is never the whole graph. It is the set of **pinned** services, what calls them, and what they call, one hop each, with the fan folded:

- Each pin is a box with a numbered badge in pin order.
- With one pin, its callers fold into one `callers of X (N)` box above it and its callees into one `callees of X (N)` box below; a single neighbour is drawn as a node.
- With several pins, *one* `more callers (N)` group holds every unpinned caller of any pin and *one* `more callees (N)` group every unpinned callee, drawn as strips above and below the whole path. The services that two or more pins share are drawn as individual nodes inside those strips, ranked by how many pins they touch and then by whole-graph degree, up to a fixed count; a path explorer is usually looking for exactly what two services share, so those are never folded.
- Every drawn edge touches a pin. A neighbour's own calls to other neighbours are not drawn; a stub line under each node (`← in · → out`) counts the callers and callees that are not drawn.
- A pair of services that call each other is one edge with a head at both ends.
- A thin pass-through in front of a router is read as the router, so a chain `A → dispatcher → router` is one hop.
- A pin whose neighbourhood does not touch another pin's is a separate drawing beside the first.

**Following is pinning.** A click on an unpinned drawn service pins it, and its own callers and callees join the groups. A click on a pinned service shows its card; its `×` unpins it. A `+` handle above and below the selected pin lists its unpinned callers or callees to pick the next hop from. A click on a folded group box does not unfold it into the drawing; it opens a **picker** in the details column that lists the members with a filter, orders them by *worst now*, *most traffic* or *most important*, and lets the user pin one, pin several via checkboxes or Shift-click, or draw all.

### 1.3 Finding the way in: the search pane

With nothing pinned there is nothing to explore, so the way in is search. A pane docked on the left indexes every node of the whole graph over everything the snapshot knows about it: name, role, tier, architectural layer, owner, note, audience, business function, live findings, incidents, and pin state. A query is whitespace-separated terms that must all match; `field:value` scopes a term, `-term` excludes, and the bare words `issue`, `incident`, `pinned`, `focused` and `vendor` match states. Each result row's checkbox *is* the pin. Hovering a row previews its card. With no query the list shows the pins first, then services with a live finding or incident, then the heaviest by traffic rank. Facets under the input narrow the list by tier and team; a business-function facet also narrows the *drawing* to the calls that carry that function and dims calls with no function data rather than hiding them, so a filter never makes a real dependency vanish silently. A `Show issues` button pins every service with a live finding. The `/` key focuses the input.

### 1.4 Visual encoding

| Element | Encoding |
| --- | --- |
| Node role | Four stroke colours: router (fan-out high), hub (both high), shared (fan-in high), service. The threshold is a Poisson tail test at the graph's mean degree, and the legend quotes the number the build used. |
| Node aside | Traffic rank from ingress as `#N` when known, else fan-in count. |
| Pin | Dashed ring in the pin colour, solid when hovered or selected; numbered badge. |
| Group box | Dashed border; label carries the member count. |
| Data store or AWS resource | Distinct layer colour with a short dash. |
| External vendor | Distinct layer colour with a long dash. |
| Internal tooling | Dotted border. |
| Measured call | Solid line; the baseline-comparison verdict (much worse, worse, healthy, unmeasured) sets its colour. |
| Measured-only (mesh-observed) call | 5-4 dash. |
| Declared-only call (permitted but no traffic seen) | 1.5-4 dot, drawn only between two pins. |
| Back edge or mutual call | Dash-dot, heads at both ends. |
| Call with no function data under a function filter | Dimmed 3-3 dash. |
| Hover | What the hovered pin touches stays bright; everything else fades. |
| Arrowheads | One SVG marker per stroke colour, 15 × 12 units, because markers cannot inherit a path's stroke portably. |
| Legend | A chip in the drawing's corner that expands on hover or focus. |

### 1.5 Health, change and incident overlays

The overlays are what make the drawing an incident tool rather than a map. All of them are fetched for the pinned set only and refreshed every minute while the pins are live.

- **Edge and node health against a baseline.** Every drawn call and every drawn service is compared with the same minute 24 hours earlier on p99 latency and error fraction. Calls come from the caller's client-side histogram, services from their own server-side histogram. A call from a caller that does not emit the client metric is *unmeasured*, never *healthy*. The verdict colours the edge, sets its hover title, and a click on the edge opens an edge card with calls per second, p99 and error rate against the baseline.
- **Recent changes.** Deploys and configuration pushes for the pinned services, newest first, each linking to the deploy system. The status line counts them.
- **Recent alerts.** Alerts that fired lately for the pinned services, one episode per firing stretch, each linking to the alerting system.
- **Live findings.** The anomaly pipeline's current findings joined to the graph by service name, so the search pane can list them, the drawing can mark them, and the card can show how each developed.
- **Incidents.** Open incidents joined to the services by catalog field or by a person attaching one from the card, with links to the incident tool and its Slack channel.
- **External vendors.** Egress gateway access logs for the last 24 hours add one node per third-party domain, with connections, failures, denials and p99 connection time, and a few-minute live check per pinned service.
- **Data stores.** Declared cloud resources from the service catalog and measured TCP connections to database proxies and message clusters, each with a verdict from the cloud provider's metrics.

### 1.6 The details card

A click selects a node or an edge and fills a card in the column between the search pane and the drawing; hover never touches the card and shows a compact tip beside the node instead. The card holds facts (layer, role, tier, owner, system), the tier signals, callers and callees ranked by traffic, business functions, tool links keyed by the service name, team and on-call with a page-the-on-call action into an open incident, and the live sections above as collapsible panels whose open state is remembered per section. With nothing selected the card shows a two-column help legend for the drawing on screen.

### 1.7 State and persistence

Every display switch lives in `localStorage` and, when it differs from its default, in the URL, so a link carries the exact view and a revisit keeps the user's preferences. The URL wins on load. Pins, open groups and the function filter are URL state. A `Reset` on the status line is computed from the one table of preferences, so it cannot drift from the controls. A cap on the number of pins bounds the per-minute live fetches. `[` and `]` step through the pins, `Esc` closes the card and, pressed again, clears the pins.

### 1.8 The status line

Under the drawing one line says what is drawn (`Exploring from 7 pins · 21 services drawn · 11 calls`) and carries the live verdicts on the pins as chips: changes in the last 30 minutes, vendors called as usual, calls worse than the baseline.

---

## Part 2: Comparison

### 2.1 Capability matrix

Legend: 🟢 present · 🟨 partial or caveated · ❌ absent.

| Capability | Jaeger DAG | Jaeger DDG | Reference design |
| --- | --- | --- | --- |
| Whole-graph drawing | 🟢 | ❌ | ❌ ¹ |
| Focal service and hop count | 🟢 | 🟢 | 🟨 ² |
| Several focal services at once | ❌ | ❌ | 🟢 |
| Fan folded into one group per side | ❌ | 🟨 ³ | 🟢 |
| Shared neighbours of several foci drawn unfolded | ❌ | ❌ | 🟢 |
| Follow a path by clicking | 🟨 ⁴ | 🟢 | 🟢 |
| Pick next hop from a ranked list | ❌ | ❌ | 🟢 |
| Search over node attributes beyond name | ❌ | ❌ | 🟢 |
| Search result is the pin toggle | ❌ | ❌ | 🟢 |
| Call volume on edges | 🟢 | ❌ | 🟢 |
| Latency and error rate on edges | ❌ | ❌ | 🟢 |
| Comparison with an earlier window | ❌ | ❌ | 🟢 |
| Edge click opens a detail card | ❌ | ❌ | 🟢 |
| Node detail card | ❌ | 🟨 ⁵ | 🟢 |
| Hover emphasises the touched subgraph | ❌ | 🟢 | 🟢 |
| Node roles from degree statistics | ❌ | ❌ | 🟢 |
| Edge provenance (measured vs declared) | ❌ | ❌ | 🟢 |
| Mutual calls folded into one edge | ❌ | ❌ | 🟢 |
| Pass-through nodes absorbed into their router | ❌ | ❌ | 🟢 |
| External vendors and data stores as nodes | ❌ | ❌ | 🟢 |
| Deploys, alerts, incidents on the selection | ❌ | 🟨 ⁶ | 🟢 |
| Per-node external decoration | ❌ | 🟢 | 🟢 |
| Operation granularity | ❌ | 🟢 | ❌ |
| Density modes | ❌ | 🟢 | ❌ ⁷ |
| Hide a node | ❌ | 🟢 | ❌ |
| Layout choice (hierarchical vs force) | 🟢 | ❌ | ❌ |
| Animated re-layout | ❌ | ❌ | 🟢 |
| Pan and zoom | 🟢 | 🟢 | 🟨 ⁸ |
| Minimap | 🟢 | 🟢 | ❌ |
| View state in URL | 🟨 ⁹ | 🟢 | 🟢 |
| Preferences in localStorage | ❌ | ❌ | 🟢 |
| Keyboard navigation | ❌ | ❌ | 🟢 |
| In-drawing legend | ❌ | ❌ | 🟢 |
| In-view help | 🟨 ¹⁰ | ❌ | 🟢 |
| Link to traces | 🟢 | 🟢 | ❌ |

- ¹ The reference design had a whole-graph drawing and removed it because nobody read it; the whole graph is reachable only through search.
- ² Depth is fixed at one hop around each pin; going further is pinning the next node, which keeps the drawing one screen.
- ³ The DDG's density modes merge nodes that share a service or operation but do not fold a fan into a single labelled group.
- ⁴ The DAG node menu offers *Set focus*, which recentres the drawing on one node rather than adding it to a path.
- ⁵ The DDG side panel shows the detail of the configured decoration, not the node's own attributes.
- ⁶ Decorations can carry any one scalar per node from an external endpoint, which is a channel for one such signal, not a card of several.
- ⁷ The fold-per-side rule replaces density modes; with one hop per pin there is no entanglement to manage.
- ⁸ Scroll and drag-to-pan only; there is no zoom, because the drawing is designed to fit one screen.
- ⁹ The DAG keeps only the search term in the URL; focal service, depth and layout are component state.
- ¹⁰ Each DAG control has a help popover; there is no help for the drawing itself.

### 2.2 What the matrix says

The reference design wins on every row that concerns *finding* and *judging*, and loses on the rows that concern *coverage*: the whole graph, operations, density modes, zoom and a minimap. That split is not an accident. The reference design traded coverage for a one-screen drawing because its users are responders who already know roughly where to look and need to see whether the thing they are looking at is worse than it was. Jaeger's two views were built for a reader learning the system's shape.

The rows that decide the recommendation are *several focal services at once*, *fan folded into one group per side*, *comparison with an earlier window* and *search result is the pin toggle*. Together they make the drawing small enough to read and give it something to say. The rows Jaeger wins (layout choice, minimap, zoom, operations) matter less for the proposed mode because a drawing of under 30 boxes does not need them, and the existing views keep them for the readers who do.

### 2.3 What Jaeger's data can and cannot support

The reference design's overlays depend on a service mesh, a service catalog, a deploy system, an alerting system and an incident tool. Jaeger has none of those and should not grow adapters for them. What Jaeger does have:

| Signal | Source in Jaeger | Supports |
| --- | --- | --- |
| Caller → callee edges with call counts | Dependencies API, per lookback window | The drawing, fan folding, traffic ordering of the picker |
| Edges at two points in time | Dependencies API called for two windows | A call-count comparison with a baseline, and *new* / *gone* edge verdicts |
| Per-edge latency and error rate | Not in the Dependencies API today; derivable from spans by the dependency aggregation job, and available through the Service Performance Monitoring (SPM) metrics endpoints per service and operation | Edge health, once the API carries it (Milestone 4) |
| Per-service latency and error rate over time | SPM metrics endpoints | Node health against a baseline, where SPM is enabled |
| Traced paths with exemplar trace IDs | Deep dependency endpoint, trace search | *View traces* for a selected edge, bounded to the two services |
| Per-node external scalar | DDG decorations config | One third-party signal per node, kept as is |
| Service and operation names | Services API | Search by name; no owner, tier or layer without a catalog |

So the drawing, search by name, the picker, pin state, the details card over Jaeger's own facts, and a call-count baseline comparison transfer with the data Jaeger already serves. Edge latency and errors need one API addition. Owner, tier, layer, incidents, deploys and alerts do not transfer and are out of scope; the decoration mechanism remains the one extension point for an operator who has such a source.

---

## Part 3: Proposal

### 3.1 The exploration mode

Add an **Explore** mode to the System Architecture page beside the current DAG. The DAG stays as it is for the reader who wants the whole shape. Explore is laid out as three columns and a status line:

```
┌──────────────┬──────────────────────────────────────────┬────────────────┐
│ Search pane  │ Drawing                                   │ Details card   │
│              │   more callers (N)  shared  shared        │                │
│ [find…    /] │          │           │      │             │ selection, or  │
│ ☑ pin-a   ①  │        ┌─┴─┐       ┌─┴──┐   │             │ help when      │
│ ☑ pin-b   ②  │        │ ① │──────▶│ ②  │◀──┘             │ nothing is     │
│ ☐ svc-c      │        └─┬─┘       └─┬──┘                 │ selected       │
│ ☐ svc-d      │          │           │                    │                │
│ …            │   more callees (N)  shared                │                │
├──────────────┴──────────────────────────────────────────┴────────────────┤
│ Exploring from 2 pins · 7 services · 9 calls   [3 calls worse than 24h ago] │
└───────────────────────────────────────────────────────────────────────────┘
```

**Search pane.** A text input over service names with the DDG's existing `uiFind` semantics, a result list whose checkbox pins the service, and with no query the pins first and then services by total call count. `/` focuses the input. Facets wait for attributes Jaeger does not have and are not part of this RFC.

**Drawing.** The pinned-path rules of §1.2 applied to the Dependencies API edges: pins in call-direction rows, one folded group per side, shared neighbours of two or more pins drawn as nodes in the strips, a stub under each node with the undrawn in and out counts, mutual calls folded into one double-headed edge, disconnected pins side by side. Clicking an unpinned node pins it; the selected pin shows `+` handles that list its unpinned callers or callees; `×` unpins. A click on a group opens the picker in the details card, ordered by call count by default and by *worst now* once edge health exists.

**Details card.** For a selected service: its name, undrawn caller and callee counts, callers and callees ranked by call count with a pin button on each, and *View traces* for the service. For a selected edge: caller, callee, call count in the current window and in the baseline window, and *View traces* bounded to the pair. With nothing selected: the help legend. Decorations configured for the DDG render here as a section.

**Status line.** What is drawn and, once a baseline is available, the number of calls worse than the baseline and the number of new or vanished calls.

**Visual encoding.** Pins ringed and numbered, group boxes dashed, a solid arrow for a call, dash-dot for a mutual call, and a verdict colour on the edge once health exists. Role colours by degree statistics transfer directly, because degree is computable from the Dependencies API alone.

**State.** Pins and the selection in the URL; display switches in `localStorage` with the URL overriding when present, following the priority stack of RFC 0007 [\[4\]](./0007-layout-settings-priority-stack.md). `[` and `]` step through pins; `Esc` closes the card and then clears the pins. A cap of 12 pins bounds the drawing and any live fetches.

### 3.2 Layout engine

The reference design's layout is hand-written. Jaeger has three options.

Legend: 🟢 good · 🟨 partial · ❌ poor.

| Criterion | A: Graphviz `dot` via Plexus (current) | B: Port the reference pipeline | C: `@xyflow/react` + `elkjs` (RFC 0008 direction) |
| --- | --- | --- | --- |
| Layering by call direction with pins at natural depth | 🟢 | 🟢 | 🟢 |
| Group boxes as nodes above and below the path | 🟢 ¹ | 🟢 | 🟢 |
| Straight corridors for long edges | 🟨 ² | 🟢 | 🟨 ³ |
| Mutual-call fold and side-by-side components | 🟢 ⁴ | 🟢 | 🟢 ⁴ |
| Animated re-layout | ❌ ⁵ | 🟢 | 🟢 |
| New code the Jaeger team owns | 🟢 (none) | ❌ (~1,500 lines) | 🟨 (rewrite of the DAG consumer) |
| Independent of the RFC 0008 decision | 🟢 | 🟢 | ❌ |
| Layout latency for a 30-node drawing | 🟢 ⁶ | 🟢 | 🟢 |

- ¹ `rank=min` and `rank=max` subgraphs pin the two strips to the top and bottom rows.
- ² `dot` routes long edges through its own dummy nodes and `splines=polyline` keeps them as segments, but it does not enforce one straight vertical per edge.
- ³ ELK's layered algorithm supports straight long edges with `edgeRouting=ORTHOGONAL`, at the cost of wider drawings.
- ⁴ Both are graph transforms applied before layout, not layout features.
- ⁵ Plexus re-runs layout in a worker and replaces positions; animating between two layouts is possible by diffing positions in the consumer, and RFC 0008 lists the cost.
- ⁶ Under 50 ms in the worker for a graph of this size, which is far below what a user notices.

**Recommendation: A.** Lessons 1, 2 and 6 of §1.1 decide this. The proposed drawing never exceeds a few dozen boxes, and at that size `dot` gives the same layering, ordering and placement quality as the reference pipeline for everything except perfectly straight corridors. Porting 1,500 lines of layout code the team would then own fails the maintenance criterion that RFC 0008 already identified as Plexus's main cost. Option C is the right destination if RFC 0008 lands, and the Explore mode is written as a transform over the Dependencies data plus a renderer contract, so it moves with the DAG when the DAG moves. Animated re-layout is deferred until the renderer decision is made, because it is the one row where A loses and the win is visual, not functional.

### 3.3 API additions

Milestone 4 needs edge-level latency and error data. Two shapes were considered.

| Criterion | Extend the Dependencies API with per-edge p99 and error rate | Join SPM per-service metrics on the client |
| --- | --- | --- |
| Measures the *call*, not the callee's whole server | 🟢 | ❌ |
| Works without SPM enabled | 🟢 ¹ | ❌ |
| Backend change required | ❌ ² | 🟢 (none) |
| Baseline comparison | 🟢 (two windows, same shape) | 🟨 (two queries per service) |

- ¹ The dependency aggregation already walks every span pair; recording latency and error quantiles per pair is an extension of the same job.
- ² The change spans the storage schema for dependencies and the query API, and is a Jaeger backend RFC of its own.

**Recommendation: extend the Dependencies API**, in a separate backend RFC. Until it lands, Explore ships with call-count comparison only, which already gives *new*, *gone*, *much more* and *much less* verdicts, and uses the SPM per-service metrics for node health where SPM is enabled.

---

## Implementation Plan

Each milestone is a separately reviewable PR against the System Architecture page and ships behind the existing page's mode switch until Milestone 3 makes Explore the default for graphs above `dagMaxNumServices`.

- **M1 — Pinned-path transform and drawing.** A pure function from Dependencies edges plus a pin set to the drawn graph of §3.1 (groups, shared nodes, stubs, mutual-call fold), rendered through Plexus with `dot` and rank constraints for the strips. Click to pin, `×` to unpin, pins in the URL. Tests cover the transform exhaustively; the drawing is a snapshot test.
- **M2 — Search pane and details card.** Search over names, checkbox pins, the card for a service and for an edge with call counts and *View traces*, the help legend when nothing is selected, the `+` handles and the group picker ordered by call count, keyboard navigation.
- **M3 — Baseline comparison on call counts.** A second Dependencies fetch for the baseline window, edge verdicts by call-count ratio, new and gone edges marked, the status line chips, and *worst now* as the picker's default order. Explore becomes the default above `dagMaxNumServices`.
- **M4 — Edge health.** Consumes the extended Dependencies API once the backend RFC lands: p99 and error rate per edge against the baseline, verdict colours, the edge card's numbers. Node health from SPM where enabled.
- **M5 — Renderer follow-through.** When RFC 0008's DAG migration lands, Explore moves to the same renderer and gains animated re-layout.

---

## Validation Plan

- **M1 acceptance.** With the standard HotROD example data, pinning `driver` draws its single caller `frontend` and its single callee `redis` as nodes rather than boxes, and pinning `frontend` and `driver` draws the call between them with `customer` and `route` folded into one callees box under `frontend`. With a synthetic graph of 800 services and 10,000 edges, any set of 12 pins renders in under 200 ms end to end.
- **M3 acceptance.** With two dependency windows in which one edge's count doubles and one edge disappears, the drawing colours the first *much more* and marks the second *gone*, and the status line counts both.
- **Readability check before M3 makes Explore the default.** Five users unfamiliar with the drawing receive the help legend only and are asked to find which of a given service's callees is new since yesterday; the target is four of five succeeding within a minute.

---

## Open Questions

1. Whether the Dependencies API's lookback window should become a request parameter, so the baseline window can be chosen, or whether the UI issues two requests against the fixed window at two end times.
2. Whether the group picker's *most important* order has a Jaeger equivalent. Total call count is the fallback; a betweenness or PageRank score over the Dependencies edges is computable on the client for 800 nodes but needs a measurement before it is promised.
3. Whether the DDG's density modes and operation granularity should eventually reach Explore, or whether operation-level questions stay with the DDG.

---

## References

1. [RFC 0008: Graph Visualization Stack](./0008-graph-visualization-stack.md)
2. [Eades, Lin, Smyth. A fast and effective heuristic for the feedback arc set problem. Information Processing Letters 47 (1993)](https://doi.org/10.1016/0020-0190(93)90079-O)
3. [Gansner, Koutsofios, North, Vo. A Technique for Drawing Directed Graphs. IEEE TSE 19(3) (1993)](https://doi.org/10.1109/32.221135)
4. [RFC 0007: Layout Settings Priority Stack](./0007-layout-settings-priority-stack.md)
