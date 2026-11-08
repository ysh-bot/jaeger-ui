# RFC 0009: One Predicate for Span Search and Span Filtering in the Trace View

* **Status**: Draft
* **Created**: 2026-11-08
* **Last Updated**: 2026-11-08
* **Related**: [ADR-0009](../adr/0009-service-filter-trace-timeline.md) (service filter), [#4606](https://github.com/jaegertracing/jaeger-ui/issues/4606), [#4272](https://github.com/jaegertracing/jaeger-ui/issues/4272), [#4580](https://github.com/jaegertracing/jaeger-ui/issues/4580), [#3815](https://github.com/jaegertracing/jaeger-ui/issues/3815), [#651](https://github.com/jaegertracing/jaeger-ui/issues/651), [#4312](https://github.com/jaegertracing/jaeger-ui/pull/4312)

## TL;DR

The trace page has two ways to narrow a trace that do not know about each other. The search box in the page header highlights the spans whose text matches, and the funnel in the "Service & Operation" column header removes the spans of unselected services together with their subtrees. Each has its own matching code, its own URL parameter, and its own place on the screen, and every request to narrow the tree by something else (span status, span kind, GenAI span type, an attribute value) has to pick one of the two to extend.

This RFC proposes that both are the same operation: a predicate over spans, evaluated once, whose result is then *presented* in one of three ways. A **match** predicate (free text, status, duration, attribute) highlights the matching spans and can optionally hide the rest. A **visibility** predicate (service checkboxes, span type checkboxes) always hides the deselected spans. When spans are hidden, a **tree mode** decides what happens to their children: the current behaviour removes the whole subtree behind an "N spans pruned" row, and a second mode lifts the children up to the nearest visible ancestor, as if the hidden span had never been recorded. All of this is driven from one control cluster: the search input with the filter popover beside it, where the popover holds the structured criteria, the per-type counts, and the two presentation switches.

The span type facet, with the lift mode as its default, is the GenAI "logical view" that [jaegertracing/jaeger#8401](https://github.com/jaegertracing/jaeger/issues/8401) asks for. It replaces the service-level approximation in [#4312](https://github.com/jaegertracing/jaeger-ui/pull/4312), which cannot remove the HTTP and database spans that live inside the agent's own service.

## Context

### What the trace page does today

**In-page search.** `TracePageSearchBar` in the page header writes the `uiFind` URL parameter, debounced by 250 ms. `filterSpans` in `src/utils/filter-spans.ts` splits the text on whitespace, keeps `"quoted phrases"` together, lower-cases every token, and marks a span as a match when *any* token is a substring of *any* of: the span name, the service name, an attribute key, an attribute value, the `key=value` form of an attribute, an event attribute, a resource attribute, or (as an exact match) the span ID. A token starting with `-` is not a negation; it excludes attribute *keys* containing that text from the comparison. There is no field scoping, no AND, and no fuzzy matching. The help tooltip documents this correctly.

Matches are presented, never removed. `SpanBarRow` adds the `is-matching-filter` class, the header shows the match count, and the up/down buttons scroll from one visible match to the next, skipping matches inside collapsed subtrees. The locate button (and loading a page whose URL already carries `uiFind`) runs `calculateFocusedFindRowStates`, which collapses every span, re-expands the ancestors of each match, and opens a detail row for every match. That is already a "filter with a collapsing side effect"; it is just not called one. `uiFind` is also consumed by the Graph view (vertex emphasis) and the Statistics view (case-sensitive `indexOf` on the row name, a different algorithm), and the search results page links into a trace with `uiFind=<spanID>`.

**Service filter.** [ADR-0009](../adr/0009-service-filter-trace-timeline.md) added the funnel button to the "Service & Operation" column header of the timeline. Its popover lists every service with a checkbox, a colour dot, a span count and a `root` badge, with Select All, Select None, Save as Default and Apply. The result is `prunedServices` in `store.timeline.ts`, encoded in the URL as `svcFilter=<checksum>.<bitmask>` and optionally saved to `localStorage`. In `generateRowStates.ts`, `buildVisibleRows` skips a span of a pruned service *and its whole subtree*, even children from visible services, and credits the hidden span count and error count to the nearest visible parent. `insertPrunedPlaceholders` then adds a "N spans pruned, M errors" row after that parent's visible children. A single root service cannot be pruned; with several root services at least one must stay. Critical-path segments of pruned descendants are merged onto the parent bar.

**How the two combine.** Only in one direction: `filterPrunedSpanIDs` removes pruned spans from the search match set before the count is shown. The locate button does not know about pruning and can open detail rows for spans that are not on screen. The Graph view ignores pruning entirely.

**Collapsing.** The four collapse buttons (Expand +1, Collapse +1, Expand All, Collapse All) sit next to the funnel in the same column header. `childrenHiddenIDs` is ephemeral: not in the URL, reset on trace change, and independent of pruning.

**GenAI view.** `ETraceViewType.GenAITimelineViewer` renders the same `TraceTimelineViewer` with the same props as the plain timeline. The only differences are that it is auto-selected for a GenAI trace and that `spanDecorations.ts` draws a type icon from `span.genAIKind`, which `classifySpan` in `src/utils/genai/detect.ts` derives from `gen_ai.operation.name` into `LLM_CALL`, `TOOL_CALL`, `AGENT`, `RETRIEVAL` or `UNKNOWN_GENAI`. Nothing narrows the tree.

### What has been asked for

- [#4606](https://github.com/jaegertracing/jaeger-ui/issues/4606) asks for filtering by GenAI span type and for hiding non-GenAI spans, and names the alternative tree behaviour: "promote their still selected / visible children to the nearest visible ancestor, in their original sibling order".
- [#4272](https://github.com/jaegertracing/jaeger-ui/issues/4272) and [#3815](https://github.com/jaegertracing/jaeger-ui/issues/3815) ask for a GenAI "logical view" that folds the infrastructure spans away. [#4312](https://github.com/jaegertracing/jaeger-ui/pull/4312) implements it as a preset of the service filter: every service with no GenAI span at or below it is pruned. Review found two bugs in the subtree logic and a design problem, which is that the service filter panel and the timeline then disagree about what is hidden. The deeper limit is stated in the PR itself: a service that owns one GenAI span keeps *all* of its spans, including its HTTP and DB spans, because they share a `service.name`. In the fixture `TraceTimelineViewer/genaiTestTrace.json`, every `POST /v1/messages` under a `chat` span survives the preset.
- [#4580](https://github.com/jaegertracing/jaeger-ui/issues/4580) asks for `status=error`, `kind=server` and `scope=…` in the search box. [#651](https://github.com/jaegertracing/jaeger-ui/issues/651) asks to highlight every span carrying a given tag. Both are predicates; one wants highlighting and the other would be happy with either.
- [#255](https://github.com/jaegertracing/jaeger-ui/issues/255) asks to view only the subtree under a given span.

### Three ways to remove a span from a tree

A span that fails a predicate can leave the tree in three ways, and each answers a different question.

- **Remove the subtree.** The span and every descendant disappear, and a placeholder row marks the gap. This is what the service filter does today. It answers "show me the trace without this service", and the placeholder keeps the parent's duration honest.
- **Lift the children.** The span disappears and its visible children take its place under its parent, in their original order. It answers "show me the tree as if these spans had never been recorded", and it is the natural rule for plumbing spans such as HTTP clients and gateways between two agent steps. No placeholder is needed, because nothing of interest was removed.
- **Keep the ancestors.** A non-matching span stays when any descendant matches, so the path from the root to each match remains as context rows, and a match's non-matching descendants are removed. It answers "where does this match sit?". It is the rule a search wants, and it is the wrong rule for a type filter: a user who unticks a span type expects that type gone, not kept as context.

Two details follow from the third rule. Per-type and per-service counts belong to the whole trace rather than to the filtered result, so that a checkbox label says what ticking it would do. And a filter must not couple itself to expansion: if the ancestors of every match are forced open on every render, Collapse All and the row chevrons stop working while a filter is active. The ancestors of the matches are opened once, when the match set changes, and expansion is then the user's again.

### Goals

1. One predicate model that today's search, today's service filter, and the requested status, kind, attribute and GenAI type criteria all fit into, so that a new criterion is a new clause and not a new mechanism.
2. One place on the screen for narrowing the tree, so that a user who has hidden spans can see that and undo it from where they typed.
3. A tree behaviour for hidden spans that can remove a span without removing what it called, so that filtering by span type produces the agent's flow and not an empty tree.
4. The GenAI logical view as a consequence of 1 to 3, not as a separate feature.
5. The existing `uiFind` and `svcFilter` URLs, the highlight behaviour, the placeholder rows and the critical-path bubbling keep working unchanged by default.

### Non-goals

- A query language. Field-scoped syntax in the text box (`status=error`) is a possible later addition and is discussed under Alternatives, but this RFC adds structured controls, not grammar.
- Server-side filtering. Everything here runs over the spans already in the browser.
- Changing the Graph, Statistics, Spans Table or Flamegraph views beyond continuing to receive the text match set they receive today.

## Design

### The predicate

A **span filter** is a record of criteria. Each criterion is either a *match* criterion or a *visibility* criterion.

| Criterion | Kind | Semantics | Today |
| --- | --- | --- | --- |
| Text terms | match | Any term is a substring of any searched field; `"quoted phrase"` is one term; `-key` excludes attribute keys; exact span ID. Unchanged from `filterSpans`. | `uiFind` |
| Status | match | `error` (status code ERROR), `ok`, `unset`. | none |
| Duration | match | Inclusive min and max on the span's own duration. | none |
| Attribute | match | `key:value` substring over the flattened `key:value` text of each span and resource attribute; a bare key or a bare value also matches. | partly, via text terms |
| Service | visibility | Deselected services are hidden. | `svcFilter` |
| Span type | visibility | Deselected types are hidden. Types are `genAIKind` plus `Non-GenAI`; the span kind (`server`, `client`, …) is a second facet in the same group. | none |

A span **matches** when it satisfies every *match* criterion, where the text terms count as one criterion with OR inside it. A span is **visible** when it satisfies every *visibility* criterion. The two kinds differ in what a user means by them. A checkbox list with Select All and Select None is read as "show these"; deselecting one of ten services and having the other nine *highlighted* would be absurd. A text term or a status is read as "find these"; the user may want them highlighted in context or may want the rest gone, and that is a choice, so it is a switch rather than a fixed rule.

Evaluation is one pass over the trace's pre-order span list, returning `matchIDs: Set<spanID>` and `hiddenIDs: Set<spanID>`. It is memoized on the trace and the filter record, as `filterSpans` is today. Per-type and per-service counts are computed over the whole trace once per trace load, not over the filtered result, so the counts tell the user what a checkbox would do rather than what it has done. This is how facet counts behave in log and trace search products, and it is cheaper.

### Presentation

The predicate result is presented in three layers, and a user controls each one.

**Highlight** is always on for the match set. `is-matching-filter`, the match count, the previous and next buttons, and the locate button behave as today, with one fix: the match set and the locate computation both exclude hidden spans, so that a count, a highlight and a detail row never refer to a span that is not on screen.

**Hide non-matching** is a switch in the popover, off by default. When it is on, spans outside the match set are added to the hidden set. With the switch off, a status criterion of `error` is [#651](https://github.com/jaegertracing/jaeger-ui/issues/651): every error span is highlighted and the tree is intact. With the switch on it is [#4580](https://github.com/jaegertracing/jaeger-ui/issues/4580): only the error spans and whatever the tree mode keeps around them remain. The switch is in the URL with the rest of the filter, so a shared link reproduces the choice.

**Tree mode** decides what happens to the children of a hidden span. It is a two-way control in the popover.

- **Remove subtree** is today's behaviour from [ADR-0009](../adr/0009-service-filter-trace-timeline.md). The hidden span and every descendant disappear, the nearest visible parent gets a "N spans pruned, M errors" placeholder row, and the pruned descendants' critical-path segments merge onto the parent bar. It is the right mode when the hidden span is an *entry point*: hiding a service means "I do not care what this service did", and what it called is part of what it did.
- **Lift children** is the behaviour asked for in [#4606](https://github.com/jaegertracing/jaeger-ui/issues/4606). The hidden span disappears and its visible children take its place under its parent, in their original order, one indent level shallower. A hidden root's visible children become roots. It is the right mode when the hidden span is *plumbing*: hiding every `Non-GenAI` span in the fixture leaves `invoke_agent triage-agent` as a root with `chat`, `execute_tool` and `retrieval` children, and under `execute_tool summarize_findings` the nested `invoke_agent summarizer-agent` sits directly, with the `mcp-gateway` and `auth-service` spans between them gone.

Lift mode has no natural place for a per-parent placeholder row, because the hidden spans are not a subtree under one parent. Instead the toolbar shows "N of M spans shown" whenever anything is hidden, in both modes, and in lift mode the error count of hidden spans is shown there too. The cost is that a reader of a lifted tree cannot tell from a row that something used to sit between it and its parent; the indent guide can mark a lifted row with a dotted connector, which is a polish item in the plan below.

The default mode is **Remove subtree**, so that an existing `svcFilter` URL and an existing saved default render exactly as before. The GenAI view's default filter (below) sets lift mode explicitly. Two defaults that depend on which checkbox was touched were considered and rejected: a user who deselects a service and then a span type would get a tree whose behaviour switched under them, and the URL would need to carry a mode per facet.

Collapsing stays independent of filtering: when the match set changes and hide-non-matching is off, the ancestors of the new matches are expanded once, and expansion is then the user's again; Collapse All works while a filter is active. The locate button keeps its stronger "collapse everything else" behaviour as an explicit action.

### Row generation

`buildVisibleRows` already runs over the pre-order list with a depth stack. Replacing `prunedServices: Set<string>` with `hiddenIDs: Set<spanID>` and `treeMode` changes it as follows.

- In **remove-subtree** mode the loop is unchanged except that the membership test is `hiddenIDs.has(span.spanID)` instead of a service lookup. `insertPrunedPlaceholders` is unchanged.
- In **lift** mode a hidden span is skipped without skipping its subtree, and each row carries a `visibleDepth` equal to the number of *visible* ancestors, maintained by the same depth stack: push on entering a span, pop on leaving, count only the visible frames. `SpanTreeOffset`, the indent guides and `SpanBarRow` read `visibleDepth` and the visible-ancestor list rather than `span.depth`. Collapse state uses the visible parent: a lifted row is hidden by collapsing its nearest visible ancestor, not its original parent.

Both modes are one O(n) pass plus the existing placeholder pass, memoized as `generateRowStatesFromTrace` is today. `isSpanPruned` and `filterPrunedSpanIDs`, which today walk the ancestor chain per span to decide whether a search match is hidden, become set lookups because `hiddenIDs` already contains every hidden span in both modes. Critical-path bubbling in lift mode merges a hidden span's own segments onto its nearest visible ancestor, the same rule as for collapsed children.

The existing root protection is kept for visibility criteria in remove-subtree mode, where hiding every root empties the tree. In lift mode a hidden root is not a problem, so the `root` lock applies only when the mode is remove-subtree.

### Controls

The search input, its match count, the previous, next and locate buttons, and the filter funnel form one cluster. The funnel moves out of the column header and into the cluster, directly after the search input. Deciding where the cluster lives is the layout question this RFC was asked to settle; see [Layout](#layout) under Alternatives. The recommendation is to keep the cluster in the page header title row, where the search input already is, and render the funnel only when the current view can hide spans (the Timeline and GenAI views).

The popover holds, top to bottom:

1. Status select, duration min and max, attribute `key:value` input. These are the match criteria that have no text-box form.
2. The **Hide non-matching spans** switch and the **Hidden spans: Remove subtree | Lift children** control, each with one line of explanation.
3. **Span type** facet: a checkbox per `genAIKind` value present in the trace plus `Non-GenAI`, each with its icon and its count, and an "only" action on hover. Shown only for GenAI traces.
4. **Span kind** facet: `server`, `client`, `producer`, `consumer`, `internal`, with counts.
5. **Service** facet: today's list with colour dot, count and `root` badge, Select All and Select None.
6. **Clear all filters**, **Save as Default**, **Apply**.

Apply stays a button rather than a live update, because the service facet is applied that way today and because a facet click that reflows an 80k-span tree should not happen on every checkbox. The text input remains live and debounced, as today; typing changes only the match set.

Active criteria other than text appear as removable chips under the cluster, so that a shared link with a hidden service shows what is hidden without opening the popover. The funnel is filled when anything is hidden, as today.

### URL and persistence

The text stays in `uiFind` and the service facet stays in `svcFilter`, so every existing link keeps working. New parameters:

| Parameter | Value | Absent means |
| --- | --- | --- |
| `spanTypes` | comma-separated list of *hidden* `genAIKind` values and `none` for Non-GenAI | nothing hidden by type |
| `spanKinds` | comma-separated list of hidden kinds | nothing hidden by kind |
| `status` | `error`, `ok`, `unset` | no status criterion |
| `minDuration`, `maxDuration` | duration with unit, as on the search page | no bound |
| `attr` | `key:value` text | no attribute criterion |
| `hide` | `1` | hide-non-matching is off |
| `tree` | `lift` | remove-subtree mode |

Hidden values rather than visible values are encoded so that the empty set is the default, as [ADR-0009](../adr/0009-service-filter-trace-timeline.md) chose for `prunedServices`. The parameters are written with `replace: true` like `uiFind` and `svcFilter`, and they follow [RFC 0007](./0007-layout-settings-priority-stack.md): a value in the URL wins, a saved default applies when the URL is silent, and a saved default is written only by Save as Default.

The view's **default filter** is the third source. The plain timeline's default is empty. The GenAI view's default hides `Non-GenAI` in lift mode. When a user changes the filter in the GenAI view, the URL carries the full filter, and clearing it writes `spanTypes=` with an empty value so that the default does not come back on the next render, which is the loop that [#3815](https://github.com/jaegertracing/jaeger-ui/issues/3815) worried about.

### Worked example

The fixture `genaiTestTrace.json` has 59 spans in 10 services. `api-edge` and `graphql-gateway` sit above `coding-agent`, whose `invoke_agent` span runs three rounds of `chat`, two `execute_tool` calls through `mcp-gateway` into `auth-service`, `metrics-backend`, `trace-store` and `k8s-api`, and a `retrieval` into `vector-store`, followed by a tool call that reaches a nested `summarizer-agent`.

| Filter | Rows | What the reader sees |
| --- | --- | --- |
| none | 59 | The full waterfall. |
| `svcFilter` hiding the five leaf services, remove subtree ([#4312](https://github.com/jaegertracing/jaeger-ui/pull/4312)'s result) | 34 spans + 10 placeholders | Every GenAI span, every `POST /v1/messages`, every `mcp-gateway` span, and a "spans pruned" row under each `mcp-gateway` call and each `retrieval`. |
| `spanTypes=none`, lift | 18 | `invoke_agent triage-agent` as root; its `chat`, `execute_tool` and `retrieval` children; `invoke_agent summarizer` directly under `execute_tool summarize_findings`, with its own `chat`. No HTTP, no gateway, no auth. |
| `status=error`, hide on, lift | 1 | The failing `k8s-api GET /api/v1/pods` as the only row. |
| `status=error`, hide off | 59 | The same span highlighted in place; the locate button collapses the tree to its path. |

The row counts come from applying the rules by hand to the fixture's span list and are to be confirmed by the tests in milestone 2.

## Alternatives Considered

### Where the predicate lives

| Criterion | A. Two mechanisms (today) | B. Extend the search syntax | C. One predicate, structured controls (recommended) |
| --- | --- | --- | --- |
| Adding a criterion is a clause, not a mechanism | ❌ | 🟢 | 🟢 |
| Discoverable without reading a tooltip | 🟨¹ | ❌ | 🟢 |
| Shows what a choice would do (counts) | 🟨² | ❌ | 🟢 |
| Keeps `uiFind` links working | 🟢 | 🟨³ | 🟢 |
| Serves the GenAI logical view | ❌ | 🟨⁴ | 🟢 |
| Implementation cost | 🟢 | 🟨 | ❌ |

🟢 good, 🟨 partial, ❌ poor.

- ¹ The service facet is discoverable; the text search's exclusion syntax is not.
- ² Only the service list shows counts.
- ³ A scoped token such as `status=error` is a substring today and would change meaning.
- ⁴ `type=llm` in a text box finds the spans but cannot say what to do with the rest.

Option C wins on the rows that matter to the open issues: the GenAI view needs a visibility criterion with counts, and the status and attribute requests need to be found by someone who did not read the help text. Option B remains compatible with C: a scoped token in the text box can later be parsed into the same criterion record, which is why the attribute criterion is already written as `key:value`.

### Tree behaviour for hidden spans

| Criterion | Remove subtree only (today) | Lift children only | Keep ancestors of matches | Both remove and lift as a mode (recommended) |
| --- | --- | --- | --- | --- |
| Hiding a service hides what it did | 🟢 | ❌ | ❌ | 🟢 |
| Hiding a span type yields the agent flow | ❌¹ | 🟢 | 🟨² | 🟢 |
| Hidden errors stay accounted for | 🟢 | 🟨³ | 🟨³ | 🟢 |
| Existing links render unchanged | 🟢 | ❌ | ❌ | 🟢 |
| One rule to explain | 🟢 | 🟢 | 🟢 | 🟨 |

- ¹ Hiding `Non-GenAI` removes the root and with it the whole trace.
- ² Produces `api-edge → graphql-gateway → agent → …`, with the plumbing above the agent kept and the plumbing below removed; defensible but inconsistent.
- ³ Only through the toolbar count, not next to the parent.

"Keep ancestors of matches" is the natural rule for a search, and it answers a real question: where does this match sit? This RFC does not adopt it as a third mode, because the same question is already answered by leaving hide-non-matching off and pressing the locate button, which collapses the tree to the matches' paths while keeping every row. Two switches with a crisp meaning each (hide or not; subtree or lift) are easier to explain than a third mode whose meaning depends on which criterion produced the match. The cost is that a reader who wants both a reduced tree and the path to each match has to use lift mode and read the toolbar count, with no context rows.

### Layout

| Criterion | A. Search in page header, funnel in column header (today) | B. Both in the page header title row (recommended) | C. Both in a toolbar row above the timeline header |
| --- | --- | --- | --- |
| Search and filter read as one thing | ❌ | 🟢 | 🟢 |
| Filter sits next to the tree it changes | 🟢 | 🟨 | 🟢 |
| Collapse buttons stay next to the tree | 🟢 | 🟢 | 🟢 |
| No new vertical space | 🟢 | 🟢 | ❌ |
| Works for the Graph and Statistics views, which take the text match set | 🟢 | 🟢 | 🟨¹ |
| The column header stays readable at 25% width | 🟨² | 🟢 | 🟢 |

- ¹ The toolbar would appear only in timeline views, so the text input would move between views.
- ² The header already holds the title, the funnel and four collapse icons; a search input would not fit.

Option B is recommended. The filter is a property of the whole view, not of one column, and the page header is where the view's other properties (view type, settings, search) already live. The funnel renders only in views that can hide spans. The cost is that the filter is one row further from the tree than today; the active-criteria chips under the cluster and the "N of M spans shown" count make the state visible without looking for the funnel.

### Service-level GenAI preset ([#4312](https://github.com/jaegertracing/jaeger-ui/pull/4312))

Kept as a mental model, not as code. A span type facet in lift mode produces the tree the preset was trying to approximate, without the two subtree bugs the review found and without a second source of truth for "what is hidden". The PR's `getServicesWithoutGenAIDescendants` and its union-and-resanitize logic are not needed when the facet is span-level. The reviewer's suggestion on that PR, that the toggle should seed the one filter rather than sit beside it, is what the GenAI view's default filter does here.

## Limitations

- **Lift mode loses the placeholder.** A reader cannot tell from a lifted row that something used to sit above it. The toolbar count and a dotted connector on lifted rows are the mitigation; a per-row marker that lists what was removed is left for later.
- **The attribute criterion is a substring, not a comparison.** `http.status_code:20` matches `200` and `201`. Typed comparisons would need a per-attribute type, which the OTLP model has but the UI does not yet surface; this stays open until the search page's structured filters ([jaeger RFC 0005](https://github.com/jaegertracing/jaeger/blob/main/docs/rfcs/0005-structured-query-filters.md)) settle the vocabulary.
- **Counts ignore other facets.** The service count next to a checkbox is the trace total, not the count after the type facet is applied. Cross-facet counts are what Datadog does and cost a second pass per facet change; they are not in scope.
- **Other views do not hide spans.** The Graph, Statistics and Spans Table views continue to receive the text match set only. Applying the visibility criteria to the Graph view's vertices is a separate design question.
- **No span-level "only this subtree"** ([#255](https://github.com/jaegertracing/jaeger-ui/issues/255)). The predicate model can express it as a visibility criterion on ancestry, but it needs its own entry point on the row and is left for a follow-up.

## Implementation Plan

Each milestone is one pull request and leaves the UI working.

1. **Predicate module.** Add `TracePage/spanFilter.ts` with the criteria record, `evaluateSpanFilter(trace, filter) → { matchIDs, hiddenIDs }`, and the URL codec for the new parameters. `filterSpans` becomes the text-term clause; the service facet becomes the first visibility clause, producing the same `hiddenIDs` that `isSpanPruned` computes today. `TracePage/index.tsx` and the store consume the result in place of `prunedServices` and `filterPrunedSpanIDs`. No UI change.
2. **Lift mode in row generation.** `visibleDepth` and the visible-ancestor list in `RowState`; `buildVisibleRows` takes `hiddenIDs` and `treeMode`; `SpanTreeOffset`, indent guides, collapse and critical path read the visible parent. Tests on `genaiTestTrace.json` assert the row counts in the worked example. No UI change; the mode is reachable only through the URL.
3. **Popover and cluster.** Move the funnel next to the search input, add the status, duration and attribute inputs, the hide switch, the tree mode control, the span kind and span type facets with counts, and the active-criteria chips. Update [ADR-0009](../adr/0009-service-filter-trace-timeline.md)'s status to point at the ADR that will record the result.
4. **GenAI view default.** The GenAI view applies `spanTypes=none` in lift mode as its default filter, with the explicit-empty clearing rule. Close [#4272](https://github.com/jaegertracing/jaeger-ui/issues/4272), [#3815](https://github.com/jaegertracing/jaeger-ui/issues/3815) and [#4606](https://github.com/jaegertracing/jaeger-ui/issues/4606); supersede [#4312](https://github.com/jaegertracing/jaeger-ui/pull/4312).
5. **Polish.** Dotted connector on lifted rows, the locate button and `calculateFocusedFindRowStates` excluding hidden spans, expand-once-on-new-matches, and the Statistics view using `evaluateSpanFilter` instead of its own `indexOf`.

Once milestones 1 to 4 have landed, an ADR records the resulting design and this RFC is marked Implemented.

## References

1. [ADR-0009: Service Filter for Trace Timeline View](../adr/0009-service-filter-trace-timeline.md)
2. [RFC 0007: Layout Settings Priority Stack](./0007-layout-settings-priority-stack.md)
3. [jaeger-ui#4606: More flexible span filtering in the timeline](https://github.com/jaegertracing/jaeger-ui/issues/4606)
4. [jaeger-ui#4272: Large GenAI traces are hard to read](https://github.com/jaegertracing/jaeger-ui/issues/4272)
5. [jaeger-ui#4312: feat(genai): hide infrastructure services in logical view](https://github.com/jaegertracing/jaeger-ui/pull/4312)
6. [jaeger-ui#3815: add logical view filter to hide infrastructure spans](https://github.com/jaegertracing/jaeger-ui/issues/3815)
7. [jaeger-ui#4580: Add timeline span filtering by status code, instrumentation scope, and span kind](https://github.com/jaegertracing/jaeger-ui/issues/4580)
8. [jaeger-ui#651: Allow highlight spans base on tag in trace detail view](https://github.com/jaegertracing/jaeger-ui/issues/651)
9. [jaeger-ui#255: Allow visualizing only part of the trace](https://github.com/jaegertracing/jaeger-ui/issues/255)
10. [jaeger#8401: Jaeger for GenAI Observability: Specialized Trace Visualization](https://github.com/jaegertracing/jaeger/issues/8401)
11. [jaeger RFC 0005: Structured query filters](https://github.com/jaegertracing/jaeger/blob/main/docs/rfcs/0005-structured-query-filters.md)
12. [OpenTelemetry semantic conventions for generative AI](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
