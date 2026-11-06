<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# research how to make k-plex - [https://github.com/zsviczian/kplex](https://github.com/zsviczian/kplex), but for Hermes [https://hermes-agent.nousresearch.com/](https://hermes-agent.nousresearch.com/) - [https://github.com/NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent), ideally integrate kplex into if uncapable to integrate, recreate and make sure to include a mention in the license (as a plugin)

Yes—this looks feasible as a standalone Hermes plugin, and the best first approach is to reuse K-Plex’s portable semantic engine and selected React components rather than recreate everything.

My recommendation is an **integration-first** project: a Hermes-native knowledge navigator with K-Plex-derived graph semantics, a new Hermes data adapter, and a native Desktop pane. Keep it outside Hermes core. Both projects expose architectural seams that support this approach, but K-Plex is not currently a drop-in, host-independent application.[^1_1][^1_2][^1_3]

This is a research and implementation blueprint, not a claim that a working port has already been built or tested.

## 1. What the research establishes

### K-Plex has reusable foundations—not just an Obsidian UI

K-Plex is a React-based knowledge navigator with meaningful spatial relationships:


| Position | Meaning |
| :-- | :-- |
| North | Parents, origins |
| South | Children, consequences |
| West | Friends, supporting connections, previous |
| East | Challengers, transformative connections, next |
| Peripheral area | Siblings |

Its distinguishing feature is deterministic relationship placement, not a conventional force-directed graph. It also supports relationship evidence, configurable ontologies, lenses, history, pins, and multiple view modes.[^1_4]

The important finding is that its developer documentation explicitly identifies portable modules:


| K-Plex component | Documented location | Porting implication |
| :-- | :-- | :-- |
| Markdown/property parser | `src/core/parser/metadata.ts` | Candidate for reuse |
| Normalized graph identities and source facts | `src/core/graph/model.ts`, `source.ts`, `settings.ts` | Candidate for reuse |
| Evidence and relationship classification | `src/core/graph/evidence.ts`, `relations.ts`, `resolver.ts` | Particularly valuable to preserve |
| Full and incremental semantic compilation | `src/core/graph/compiler.ts`, `patch.ts` | Candidate for reuse |
| Predicates and lens evaluation | `src/core/plex/` | Candidate for reuse |
| Shared React components and migrated features | `src/ui/components/`, `src/ui/features/` | Reuse selectively |
| Obsidian data collection | `src/adapters/obsidian/*SourceCollector.ts` | Replace with Hermes adapters |
| Live index, search, caches | `src/index/GraphIndex.ts` | Still coupled; investigate before reuse |

K-Plex’s own documentation cautions that “a portable semantic engine does not mean the application and all UI are host-independent.” That is the central constraint: reuse the established boundaries, not the entire plugin entry point.[^1_1]

### Hermes already has the extension surfaces needed

Hermes has distinct plugin systems:

- Python plugins for agent tools, lifecycle hooks, skills, and CLI commands.
- Native Desktop plugins for panes, pages, navigation, and other UI contributions.
- A separate web-dashboard plugin system.

These are not interchangeable. A Desktop plugin does not automatically work in the browser dashboard.[^1_2][^1_3]

For this project, the strongest starting point is Hermes’s unified package layout: Python agent-side code plus `desktop/plugin.js` in one standalone plugin. The Desktop SDK supports plugin-scoped REST/socket access through `/api/plugins/<id>` and a backend `plugin_api.py`.[^1_2]

## 2. Recommended architecture

Proposed project name: `hermes-knowledge-plex`.

The following is a proposed design, not an existing upstream package:

```text
Hermes Desktop
└── Knowledge Plex pane
    ├── Compass visualization
    ├── Search, history, pins
    ├── Evidence inspector
    └── Relationship proposal/review UI
              │
              │ Plugin-scoped REST/socket API
              ▼
Hermes Python plugin
├── Agent tools
├── Source adapters
├── Incremental indexing coordinator
├── Provenance and proposal store
└── Explicitly approved source writes
              │
              ▼
K-Plex-derived TypeScript engine
├── Normalized source facts
├── Relationship compiler/resolver
├── Neighborhood projection
└── Lens evaluation
```


### Preserve the TypeScript engine

Do not immediately translate K-Plex’s semantics into Python.

Instead, test whether its portable core can run as a small, plugin-owned Node sidecar, communicating with the Python backend through a versioned JSON protocol. This preserves the original classifier and evidence precedence rather than creating two implementations that gradually disagree.

This is an architectural recommendation based on K-Plex’s documented portable-core boundaries—not a verified sidecar implementation. Its current development baseline is Node 22.22.2, and the repository includes a dedicated host-free core verification command.[^1_5][^1_1]

A browser-hosted core is another option for an initial visualization prototype, but I would prefer a backend-accessible engine for the full product so Hermes tools can query it even when Desktop is closed.

### Use the native Desktop SDK, not UI injection

Register a real Knowledge Plex pane or page. Avoid DOM scraping, synthetic clicks, or monkey-patching Hermes internals. Hermes documents those internals as outside its compatibility contract, and its plugin catalog rejects core overrides.[^1_3][^1_2]

One packaging detail matters: disk Desktop plugins load uncompiled ESM, and their allowed import specifiers are restricted to the SDK and React. Therefore, do not assume you can import a K-Plex npm package directly at runtime. Test a compiled, self-contained frontend bundle that leaves only the supported host imports external.[^1_2]

## 3. What “K-Plex for Hermes” should represent

I would make this a navigator over Hermes-related knowledge, not merely an Obsidian graph embedded beside a chat.

Hermes already maintains conversations, memories, and skills, and supports cross-session recall. Those provide the starting source categories.[^1_6]

### Proposed node types

| Node type | Initial source | Initial editing policy |
| :-- | :-- | :-- |
| Knowledge note | Explicitly configured Markdown directory | Editable with preview |
| Skill | Hermes skill resources | Read-only initially |
| Conversation | Supported session interfaces | Read-only |
| Memory entry/document | Supported memory interfaces or configured files | Read-only initially |
| Project | User-declared project record | Plugin-owned |
| External reference | Linked URL or document | Metadata only |
| Proposed concept | Agent-generated suggestion | Pending review |

Do not assume every memory provider exposes the same records or modification API. Provider integrations should be capability-based and added separately.

### Proposed relationship mapping

| Relationship | Compass role |
| :-- | :-- |
| `part_of`, `derived_from` | Parent |
| `contains`, inverse of `derived_from` | Child |
| `supports`, `related_to` | Friend |
| `contradicts`, `challenges` | Challenger |
| `previous`, `next` | Ordered lateral relationships |
| Shared parent | Sibling |

These mappings should be configurable. In particular, do not equate every revision or replacement with a contradiction.

### Keep evidence separate from interpretation

This is the feature most worth carrying over from K-Plex.

K-Plex records why an edge exists and can distinguish several source occurrences contributing to the same visible relationship. Its source documents remain authoritative rather than being replaced by an opaque graph database.[^1_4]

For Hermes, each relationship should retain:

```text
source identifier
source kind
source revision/hash
exact location where available
declared relationship
resolved compass role
origin: explicit / structural / agent-proposed
review status
```

My recommended precedence:

1. Explicit user-authored relationship.
2. Explicit structured source metadata.
3. Structural relationship.
4. Reviewed agent proposal.
5. Unreviewed agent proposal, shown separately.

An LLM-generated connection should never silently become a user-authored fact.

## 4. Integrate first; recreate only where necessary

### Stage 1: Bounded portability spike

Before committing to a rewrite:

1. Pin a specific K-Plex commit and Hermes release.
2. Run K-Plex’s actual core checks and build.
3. Feed synthetic Hermes-style source records through the portable compiler.
4. Render one Compass neighborhood inside a genuine Hermes Desktop pane.
5. Verify that the selected frontend dependency closure contains no Obsidian runtime imports.
6. Exercise evidence inspection and incremental source updates.

K-Plex already provides architecture checks, host-free core tests, and a real build contract. Use those as the baseline rather than substituting stub-only type checks.[^1_5][^1_1]

### Stage 2: Choose the reuse boundary

| Finding | Response |
| :-- | :-- |
| Core and selected UI work without Obsidian | Reuse both |
| Core works, UI remains heavily host-bound | Reuse core; implement Hermes-native UI |
| Index orchestration remains host-bound | Replace orchestration, not semantic classification |
| Specific modules cannot be separated reasonably | Reimplement those modules against documented behavior |
| Licensing provenance remains unresolved | Pause copied-code distribution; resolve permissions or choose an independently implemented route |

The likely sweet spot is core reuse plus a substantially new host adapter and some new UI. That is an inference from the documented boundaries, not a completed import audit.[^1_1]

## 5. Plugin package and agent interface

A proposed standalone layout:

```text
hermes-knowledge-plex/
├── plugin.yaml
├── __init__.py
├── tools.py
├── adapters/
├── service/
├── desktop/
│   ├── plugin.js
│   └── plugin_api.py
├── engine/
│   ├── src/
│   └── package.json
├── upstream/
│   └── kplex/
├── skills/
│   └── knowledge-navigation/
│       └── SKILL.md
├── tests/
├── LICENSE
├── NOTICE
├── THIRD_PARTY_NOTICES.md
└── UPSTREAM.md
```

Use Hermes-managed durable plugin storage rather than storing the database inside the replaceable installation directory. Hermes documents `plugin_data_dir()` and `plugin_db()` specifically for state that survives plugin updates/removal and follows the active profile.[^1_3]

### Proposed tools

Start with a small interface:


| Tool | Purpose |
| :-- | :-- |
| `plex_search` | Find nodes |
| `plex_neighborhood` | Return a bounded, typed neighborhood |
| `plex_explain_connection` | Return supporting evidence |
| `plex_propose_relation` | Create a reviewable proposal |
| `plex_index_status` | Report coverage, pending work, and failures |

Add source-editing tools later, after implementing previews, revision checks, and approval handling.

The actual registration mechanism is documented: Python plugins use `register(ctx)` with `ctx.register_tool(...)`; the manifest should list exactly the tools registered.[^1_3]

I would initially make this an additive tools/UI plugin, not a replacement memory provider. Hermes’s memory-provider interface is a distinct, single-select mechanism, so replacing it unnecessarily would make this plugin compete with the user’s chosen memory backend.[^1_7]

## 6. Licensing: there is a real inconsistency to resolve

The repository currently presents conflicting signals:

- GitHub identifies K-Plex as AGPL-3.0.
- Its root `LICENSE` contains the GNU Affero General Public License v3.
- Its `package.json` says `"license": "MIT"`.[^1_8][^1_4][^1_5]

Do not treat the package metadata as sufficient permission to redistribute copied K-Plex code under MIT.

For a port using upstream code, my conservative recommendation is:

- Resolve the inconsistency with the maintainer and audit the pinned revision’s notices.
- Plan for AGPLv3 compliance unless clear applicable permissions establish otherwise.
- Preserve upstream copyright and license notices.
- Identify modified files and modification dates.
- Provide corresponding source and build materials when required.
- Include the applicable source-access offer for remote interaction with a modified version.[^1_8]

Calling it a plugin does not, by itself, create a copyleft exception. The scope of any combined work requires review; a standalone repository or process boundary is an architectural choice, not an automatic legal exemption. The AGPL’s terms expressly address covered works, combined works, aggregation, and corresponding source.[^1_8]

Hermes itself is MIT-licensed, but that does not make copied K-Plex material MIT-licensed.[^1_6]

### How to include the acknowledgment you requested

Keep the standard AGPL license text unchanged. Put the project-specific acknowledgment in `NOTICE` or `THIRD_PARTY_NOTICES.md`, with a pointer from the project’s licensing documentation. The AGPL license document itself states that changing its text is not allowed.[^1_8]

For an actual code-reusing port, suggested notice:

```text
K-Plex attribution

This plugin incorporates and adapts portions of K-Plex (Knowledge Plex)
by Zsolt Viczian and its contributors.

Upstream project:
https://github.com/zsviczian/kplex

Upstream revision:
[exact commit SHA]

Adapted components:
[list actual incorporated files or modules]

Upstream copyright and license notices are preserved.
See LICENSE and THIRD_PARTY_NOTICES.md for applicable terms.

This is an independent Hermes integration and is not represented as
an official release of K-Plex or Nous Research.
```

If the fallback is independently implemented without incorporating upstream code or assets, use an accurate acknowledgment instead:

```text
Design acknowledgment

This plugin's ontology-driven Compass navigation was inspired by
K-Plex (Knowledge Plex), created by Zsolt Viczian:
https://github.com/zsviczian/kplex

This implementation is independent and does not incorporate K-Plex
source code or assets.
```

Use that second version only if it is true. Rewriting copied implementation details into another language should not be treated as proof of independence.

## 7. Delivery plan and acceptance criteria

### Milestone 1 — Read-only knowledge navigator

Deliver:

- Native Hermes Desktop pane.
- Configured Markdown source directory.
- Compass navigation.
- Search and connection evidence.
- Incremental updates and restart recovery.
- Accurate upstream attribution.

Acceptance: indexing and navigation work without loading Obsidian, and disabling the plugin removes its UI and listeners cleanly.

### Milestone 2 — Hermes knowledge integration

Deliver:

- Skill and conversation adapters through supported interfaces.
- Stable source identities.
- Provenance-aware relationships.
- Agent-queryable neighborhoods.
- Profile isolation.
- Explicit coverage reporting when some sources are unavailable.

Acceptance: changing profiles or connected gateways cannot leak or mix graph data.

### Milestone 3 — Reviewed editing

Deliver:

- Relationship proposals.
- Before/after source previews.
- Source revision conflict detection.
- Approved Markdown writes.
- Recovery or rollback records.
- Preservation of unrelated source content.

Acceptance: a stale edit cannot overwrite newer content, and an agent proposal cannot silently modify a source.

### Milestone 4 — Advanced views and distribution

Deliver:

- Lenses, pins, history, and optional additional layouts.
- Packaging and source-release process.
- Catalog validation.
- Optional separate web-dashboard frontend.

Hermes provides `hermes plugins doctor` and `hermes plugins validate` for discovery/registration and catalog checks; these should supplement, not replace, functional and integration tests.[^1_3]

## Final recommendation

Build a standalone Hermes plugin that reuses K-Plex’s semantic core first, replaces Obsidian-specific data access, and contributes a native Desktop visualization. Recreate only the portions that fail a bounded portability test.

The two decisions to settle before distribution are the exact reusable dependency boundary and K-Plex’s conflicting license declarations. Everything else can be organized into an incremental implementation, beginning with a read-only graph and preserving the attribution you requested from the first release.[^1_5][^1_1][^1_8]

<span style="display:none">[^1_10][^1_11][^1_12][^1_13][^1_14][^1_15][^1_16][^1_17][^1_18][^1_19][^1_20][^1_21][^1_9]</span>

<div align="center">⁂</div>

[^1_1]: https://github.com/zsviczian/kplex/blob/main/AGENTS.md

[^1_2]: https://hermes-agent.nousresearch.com/docs/developer-guide/desktop-plugin-sdk

[^1_3]: https://hermes-agent.nousresearch.com/docs/developer-guide/plugins

[^1_4]: https://github.com/zsviczian/kplex

[^1_5]: https://github.com/zsviczian/kplex/blob/main/package.json

[^1_6]: https://github.com/NousResearch/hermes-agent

[^1_7]: https://hermes-agent.nousresearch.com/docs/user-guide/features/plugins

[^1_8]: https://github.com/zsviczian/kplex/blob/main/LICENSE

[^1_9]: https://hermes-agent.nousresearch.com/

[^1_10]: https://hermes-agent.nousresearch.com/docs/developer-guide/memory-provider-plugin

[^1_11]: https://hermes-agent.nousresearch.com/docs/plugins/hermes-memory-wiki

[^1_12]: https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers

[^1_13]: https://hermes-agent.nousresearch.com/docs/developer-guide/image-gen-provider-plugin

[^1_14]: https://hermes-agent.nousresearch.com/docs/developer-guide/web-search-provider-plugin

[^1_15]: https://hermes-agent.nousresearch.com/docs/user-guide/features/built-in-plugins

[^1_16]: https://hermes-agent.nousresearch.com/docs/developer-guide/model-provider-plugin

[^1_17]: https://hermes-agent.nousresearch.com/docs/developer-guide/plugins/catalog-submission

[^1_18]: https://hermes-agent.nousresearch.com/docs/plugins/superlocalmemory

[^1_19]: https://hermes-agent.nousresearch.com/docs/plugins/neuralmind

[^1_20]: https://hermes-agent.nousresearch.com/docs/developer-guide/video-gen-provider-plugin

[^1_21]: https://hermes-agent.nousresearch.com/docs/plugins/memory-rewind


---

# research more, and also include potential for Hermes Desktop App integration, we want to re-use, not re-implement as much of the original as possible, ideally having a fork of kplex as a submodule in our new repository

Yes—the right direction is a new Hermes integration repository containing your own K-Plex fork as a pinned Git submodule, with narrowly scoped portability changes made inside that fork. The goal should be **reuse** of K-Plex’s implementation, not a Hermes-specific recreation of its behavior.

The deeper architectural review strengthens that recommendation, but changes one part of my earlier advice: do not commit to a Node sidecar before testing how much of K-Plex’s existing application and rendering code can be extracted intact. First establish a reusable K-Plex runtime and UI boundary; then decide where that runtime executes.

Hermes Desktop is a credible primary integration target: its SDK explicitly supports contributed full pages, dockable panes, sidebar navigation, commands, settings, and plugin-owned backend APIs.[^2_1]

## 1. What the deeper review uncovered

### K-Plex has more reusable infrastructure than the first answer captured

K-Plex’s architecture now documents reusable view policies, application sessions, action dispatch, localization, and presentation-environment contracts—not only parsing and graph semantics.[^2_2]


| Area | What already exists | Integration strategy |
| :-- | :-- | :-- |
| Parsing | Portable Markdown/property grammar with injected scheduling and cancellation | Reuse unchanged where possible |
| Graph semantics | Normalized source contracts, compiler, patch preparation, evidence, resolver | Reuse; adapters supply facts |
| Predicates and lenses | Host-free evaluation and evidence/property capabilities | Reuse |
| View policies | Host-free projection/layout definitions and static registry | Reuse |
| View-session coordination | `application/PlexViewSession` | Investigate and reuse |
| Action dispatch | `application/ActionManager` with narrow host ports | Adapt handlers rather than replace the dispatcher |
| Search UI | Portable `ui/features/SearchBox.tsx` receiving a narrow read contract | Reuse |
| Shared interactions | Components and migrated feature presentation | Reuse |
| Localization | Typed catalog and translator | Reuse |
| Device and keyboard presentation | Explicit environment and shortcut contracts | Implement Hermes environment adapter |

These are documented boundaries, not proof that the entire UI can already run outside Obsidian. K-Plex explicitly says Obsidian remains its only production host and that much of the legacy UI, indexing, and settings remains coupled.[^2_3][^2_2]

### Three existing views are especially relevant

K-Plex documents these production definitions:

- Compass, stored as `classic`.
- Mind Map, stored as `mindmap`.
- Facets, stored as `wuxing`.

Their projection/layout definitions live under `core/plex/views/`. The architecture says Classic’s ordered neighborhood/sibling policy and measured geometry have one implementation.[^2_2]

That changes the implementation target:

> Preserve the existing view policies, layout algorithms, renderer behavior, and action model. Extract or adapt their host dependencies rather than build another graph visualization.

Do not promise all three views immediately. Compass should establish the reusable integration, with Mind Map and Facets retained as explicit follow-on compatibility targets.

### The biggest remaining coupling is above the semantic engine

The architecture identifies specific constraints:

- `GraphIndex` still owns publication, search, caches, and host-bound coordination.
- Legacy `PlexGraph`, content presentation, and other consumers still depend on concrete graph/index structures.
- `KplexDocumentSession` retains native workspace/editor ownership.
- Compass and Facets use an attention-following native editor.
- Mind Map uses a shared capacity-one occurrence host.
- Native settings, commands, file operations, and modal shells remain Obsidian responsibilities.[^2_2]

Therefore, “import the portable core and write a new graph UI” would leave considerable upstream value unused. Your preferred fork gives us a better option: finish only the extractions required for Hermes, preserving the original implementation.

## 2. Recommended repository structure

Use two repositories with distinct responsibilities.

### Repository A: your K-Plex fork

Purpose: maintain an upstream-compatible K-Plex with a reusable embedding surface.

Changes belong here when they:

- Remove an unnecessary Obsidian dependency from existing logic.
- Introduce a narrow capability interface around an actual host operation.
- Make an existing component embeddable.
- Add a portable entry point or a second build target.
- Add portability/conformance tests.

Keep Hermes-specific SDK imports out of this repository.

### Repository B: the Hermes integration

Purpose: adapt the reusable K-Plex surface to Hermes Desktop and agent tools.

Proposed structure:

```text
hermes-kplex/
├── .gitmodules
├── vendor/
│   └── kplex/                  # Your K-Plex fork, pinned submodule
├── packages/
│   ├── kplex-hermes-adapter/   # Hermes-facing data/action adapters
│   ├── desktop-plugin/        # Native Hermes Desktop entry point
│   └── engine-service/        # Optional headless runtime
├── hermes_plugin/
│   ├── plugin.yaml
│   ├── __init__.py
│   ├── tools.py
│   └── desktop/
│       ├── plugin.js          # Generated installable bundle
│       └── plugin_api.py
├── tests/
│   ├── compatibility/
│   ├── desktop/
│   └── fixtures/
├── scripts/
│   ├── build.mjs
│   └── package.mjs
├── docs/
│   ├── architecture.md
│   ├── upstream-maintenance.md
│   └── compatibility-matrix.md
├── LICENSE
├── NOTICE
└── THIRD_PARTY_NOTICES.md
```

This is a proposed layout, not a requirement imposed by either upstream.

The installable directory follows Hermes’s documented unified-package pattern: agent code with a `desktop/plugin.js` frontend. Developer source organization can remain separate from that generated installation layout.[^2_1]

### Why a forked submodule fits this project

A submodule lets the integration record an exact K-Plex revision while preserving the fork’s independent history. Git normally checks out the commit recorded by the parent repository, rather than automatically following the latest branch head.[^2_4]

Recommended policy:

- Fork `zsviczian/kplex`.
- Maintain a small portability branch in your fork.
- Pin the parent repository to a reviewed fork commit.
- Update through explicit compatibility PRs.
- Preserve upstream history and attribution.
- Avoid copying K-Plex files into a parallel implementation.

Developer setup, once the fork exists:

```bash
git submodule add <YOUR_KPLEX_FORK_GIT_URL> vendor/kplex

git -C vendor/kplex remote add upstream \
  https://github.com/zsviczian/kplex.git

git submodule update --init --recursive
```

For release builds, use the recorded commit. Do not run `git submodule update --remote` in release CI: that option intentionally selects the remote-tracking revision instead of the parent’s recorded revision.[^2_4]

## 3. The reusable boundary to create in the fork

I would aim for an embedded K-Plex application surface—not merely a library of graph algorithms.

The following names are illustrative proposed APIs, not existing exports:

```ts
interface KplexHostCapabilities {
  sources: SourceAccess;
  navigation: NavigationAccess;
  documents: DocumentAccess;
  persistence: PersistenceAccess;
  commands: CommandAccess;
  presentation: PresentationAccess;
}
```

The interfaces should be derived from actual call sites. Do not introduce a speculative “universal host” abstraction before examining the import and effect graph.

K-Plex already follows this principle: define narrow capabilities in the consuming layer, implement effects in adapters, and keep `App`, plugin instances, native files, and mutable graph repositories out of portable code.[^2_3][^2_2]

### Extraction priorities

| Priority | Work in the fork | What it preserves |
| :-- | :-- | :-- |
| 1 | Public exports for existing portable modules | Semantic implementation |
| 2 | Embeddable view composition around existing policies/rendering | Graph appearance and interactions |
| 3 | Narrow document/navigation capabilities | Existing selection and navigation behavior |
| 4 | Host-independent ownership around necessary index/runtime operations | Existing scheduling and publication invariants |
| 5 | Native editor capability boundary | Existing editor-placement behavior |
| 6 | Settings/action presentation separation | Existing settings model and action semantics |

Avoid extracting the entire application at once.

K-Plex’s own refactor plan currently pauses broader structural work after C14. A Hermes effort should explicitly scope the additional changes rather than presume that all later planned extractions already exist.[^2_3]

### Do not use a fake Obsidian runtime as the production solution

A compatibility shim may help characterize behavior in tests, but I would reject a production architecture that pretends Hermes is an Obsidian vault, workspace, metadata cache, and editor.

Instead:

```text
Existing K-Plex implementation
             │
      narrow capability
        ┌────┴─────┐
        ▼          ▼
Obsidian adapter   Hermes adapter
```

This follows K-Plex’s documented dependency direction and makes upstreaming feasible. Its architecture specifically identifies “another PKM host” as a new adapter supplying normalized facts through narrow ports.[^2_2]

## 4. Hermes Desktop integration: concrete possibilities

### A. Full-page knowledge workspace

This should be the primary surface.

Register a route and sidebar item for a large K-Plex workspace. Hermes supports full pages through `ROUTES_AREA` and sidebar navigation through `SIDEBAR_NAV_AREA`.[^2_1]

Proposed experience:

- Existing K-Plex visualization fills the workspace.
- Existing search, lenses, history, and view selection are retained as they become portable.
- Selecting a source opens its document or source-specific detail view.
- “Discuss this” inserts a reference into the relevant Hermes composer.

This suits K-Plex better than forcing the entire graph into a narrow sidebar.

### B. Dockable companion pane

Use a second surface for a bounded neighborhood beside chat.

Hermes supports draggable panes with semantic placement and explicit docking relative to built-in panes such as the workspace, files, terminal, and review panel.[^2_1]

Proposed behavior:

- Follow the selected session or an explicitly pinned graph node.
- Show a smaller neighborhood.
- Offer “open in knowledge workspace.”
- Preserve independent view state where appropriate.
- Reuse the same underlying graph runtime rather than create another index.

One Desktop lifecycle detail matters: closing a plugin’s only contributed pane disables that plugin; closing one of multiple panes dismisses only that pane. Test the intended combination of route, pane, and commands against the selected Hermes version.[^2_1]

### C. Composer integration without automatic sending

Hermes exposes supported composer methods for reading drafts, inserting text, setting drafts, and submitting messages. They are session-addressed and fail closed when the target surface is unavailable.[^2_1]

Recommended first interaction:

> Select a K-Plex node → “Add to conversation” → insert a concise reference into the intended draft.

Do not submit automatically. Keep “insert context” and “send message” separate actions.

### D. Settings, commands, and indexing status

Hermes supports plugin settings pages, palette commands, user-rebindable keybindings, and status-bar contributions.[^2_1]

Use them for:

- Source-root configuration.
- Index status and errors.
- Open knowledge workspace.
- Focus graph search.
- Toggle follow-session behavior.
- Show upstream/version information.

Reuse K-Plex’s settings data model where possible, but do not attempt to reuse Obsidian’s native settings shell unchanged.

### E. Session annotations

Hermes provides session-row decoration slots and durable lineage-aware session identifiers.[^2_1]

A later integration could show whether a session has linked knowledge nodes or a curated summary. Use the durable identity supplied by the supported interface; do not derive identity by inspecting React internals or sidebar DOM.

### F. Features that need further SDK verification

Do not assume arbitrary message-selection menus, per-message graph buttons, or native pop-out controls are already supported.

Hermes has issue discussions requesting selection context, message-render extension points, project-scope state, and additional pop-out controls. Those discussions are signals to verify capabilities against the chosen release—not evidence that every requested API exists.[^2_5][^2_6][^2_7][^2_8]

These are optional enhancements, not blockers for the page-and-pane integration.

## 5. Build integration: a new target, not the Obsidian bundle

K-Plex’s current production build:

- Starts at `src/main.ts`.
- Produces CommonJS.
- Leaves `obsidian`, `electron`, CodeMirror modules, and Node built-ins external.
- Copies the Obsidian manifest and styles.
- Includes an Obsidian-specific React DOM hardening step.[^2_9]

That bundle cannot simply become Hermes’s `plugin.js`.

Hermes disk plugins require an ESM default export, and runtime imports are restricted to:

```text
@hermes/plugin-sdk
react
react/jsx-runtime
```

The “no build required” documentation describes authoring plain JavaScript plugins; it does not mean an existing TypeScript/React codebase needs no compilation.[^2_1]

### Recommended build approach

Add a separate embedding/Desktop build:

```text
K-Plex fork source + Hermes adapters
                  │
               bundler
                  ▼
       ESM desktop/plugin.js
       + packaged backend/runtime assets
```

Build rules:

1. Preserve K-Plex’s original Obsidian build.
2. Compile the Hermes entry point separately.
3. Externalize Hermes SDK and host React imports.
4. Bundle permitted implementation dependencies.
5. Fail if `obsidian`, Node built-ins, Electron, or app-internal imports survive in the frontend artifact.
6. Inspect `react-dom` usage explicitly.
7. Install styles through a documented, disposable mechanism.

K-Plex depends on React and React DOM, while its portable UI layer allows ReactDOM usage. Hermes’s disk import whitelist does not include `react-dom`, making portals and mounting behavior a specific compatibility probe. Do not blindly bundle a second React instance or assume a nested root is necessary.[^2_10][^2_2][^2_1]

### CSS and theme compatibility deserve their own adapter

K-Plex intentionally uses Obsidian semantic CSS classes and variables.[^2_3]

For maximum reuse:

- Keep upstream component structure and K-Plex-specific styles.
- Scope the integration under a dedicated root.
- Map the required Obsidian-style tokens to Hermes theme values.
- Replace only native-shell assumptions that cannot be mapped.
- Test portal styling, stacking, focus, and cleanup.

Avoid importing an entire Obsidian theme or applying compatibility styles globally.

## 6. Where the graph runtime should execute

There are two practical targets, and they need not be solved simultaneously.

### First: embedded runtime for the Desktop portability prototype

Run reusable K-Plex code in the Desktop renderer against a fixture/source adapter.

Purpose:

- Prove the actual view implementation can render.
- Identify remaining host dependencies.
- Preserve upstream interactions.
- Avoid introducing IPC before it is needed.

This is a development milestone, not my recommendation for the final authoritative data store.

### Then: gateway-accessible runtime for agent tools

For a complete plugin, the graph must remain queryable without an open Desktop window.

A proposed final arrangement:

```text
Hermes Desktop plugin
  └── reused K-Plex UI/projection code
                 │
        plugin-scoped API
                 ▼
Hermes Python plugin
  └── backend/API and lifecycle bridge
                 │
                 ▼
K-Plex TypeScript runtime
  └── reused parsing, compilation, evidence,
      indexing/query behavior where extracted
```

A Node process is a plausible way to retain the TypeScript implementation, but its process lifecycle, packaging, runtime availability, and cross-platform behavior remain engineering work. K-Plex currently declares Node `>=22.22.2 <23` for its development/build environment; that does not itself establish a tested runtime-support range for a new sidecar.[^2_10]

The key invariant should be one authoritative semantic runtime per configured scope—not separate Python and TypeScript classifiers.

### Remote Hermes connections affect the design

Desktop plugins are app-level and can operate while the window connects to different profiles, gateways, or remote machines. Hermes exposes backend access through scoped plugin APIs.[^2_1]

Therefore:

- Backend-owned source paths must be interpreted on the backend machine.
- Graph/query caches must be isolated by connection and profile.
- A connection switch must invalidate in-flight work and subscriptions.
- Desktop-local preferences must not accidentally become shared knowledge state.
- Remote files should not be opened through guessed local `file://` paths.

This is another reason not to make Desktop-local indexing the final authority for all installations.

## 7. Maximize reuse by starting with Markdown-backed sources

My previous proposal introduced many Hermes-specific node categories immediately. For your stated priority, I would narrow the first release.

### First source: a configured Markdown knowledge directory

This keeps the initial source model close to K-Plex’s existing domain:

- Markdown notes.
- Explicit ontology properties.
- Links.
- Tags.
- Folders.
- Attachments.
- Unresolved references.

K-Plex explicitly requires preserving these source kinds and their relationship semantics.[^2_3]

The missing piece is source acquisition and link resolution outside Obsidian—not a new relationship engine.

### Later: Hermes-specific sources

Add conversations, skills, and memory-provider content through adapters once the original model works.

Important constraints:

- Keep opaque node IDs opaque and case-sensitive.
- Preserve semantic paths separately from physical paths.
- Do not invent exact source locations when the source does not supply them.
- Preserve declaration multiplicity and provenance.
- Keep agent-proposed connections distinguishable from authored relationships.

These are closely aligned with K-Plex’s normalized source contract and compiler invariants.[^2_2][^2_3]

For read-only Hermes sources, disable unsupported mutation actions through capabilities rather than duplicating the view or pretending those sources are editable Markdown.

## 8. Submodule maintenance and licensing

### Keep the fork’s delta measurable

Track each fork change as one of:

- Portable export/build support.
- Host capability extraction.
- Reusable UI extraction.
- Test infrastructure.
- Necessary bug fix.

Keep Hermes business logic in the parent repository.

For every upstream update, record:

```text
upstream base commit
fork commit
parent repository commit
changed portability surfaces
actual test results
known regressions or unsupported features
```

Git submodules preserve the exact dependency commit, but the maintenance discipline is yours to establish.[^2_4]

### License correction remains a release gate

The previously identified inconsistency remains relevant: K-Plex’s `package.json` declares MIT, while the root license reviewed earlier contains AGPLv3. The package metadata alone should not be used to conclude that copied code is MIT-licensed.[^2_11][^2_10]

A submodule does not resolve that discrepancy or remove obligations associated with distributing modified code.

Recommended release position:

- Preserve the upstream license and notices.
- Resolve the conflicting declarations with the maintainer.
- Plan conservatively for AGPLv3 compliance until clarified.
- Include the complete corresponding source for the exact shipped fork and integration where required.
- Do not rely solely on an installer bundle or a reference to a moving branch.[^2_11]

Use the parent project’s `NOTICE` to state plainly:

```text
This project integrates K-Plex (Knowledge Plex), created by
Zsolt Viczian and contributors, through a maintained fork.

The K-Plex implementation is reused and adapted, not independently
recreated.

Upstream: https://github.com/zsviczian/kplex
Fork: [actual fork URL]
Shipped revision: [exact commit]
```

Also make this attribution visible in the plugin’s About/settings page. Keep the standard license text unchanged.

## 9. The implementation sequence I would choose

| Step | Deliverable | Acceptance criterion |
| :-- | :-- | :-- |
| 1 | Pinned baseline and dependency inventory | Exact commits recorded; portable, legacy, and native dependencies distinguished |
| 2 | Forked submodule wiring | Clean recursive checkout reproduces the dependency tree |
| 3 | Desktop fixture prototype | Actual K-Plex view policies and reusable rendering run in a Hermes page |
| 4 | Minimal fork extractions | No production Obsidian shim; no duplicate semantic implementation |
| 5 | Markdown source adapter | Existing semantics and evidence match characterized fixtures |
| 6 | Runtime/backend integration | Agent tools and Desktop query the same authoritative graph |
| 7 | Docked pane and composer context | Supported SDK calls only; correct session/profile targeting |
| 8 | Editing and additional views | Existing actions reused where portable; unsupported operations fail explicitly |
| 9 | Release packaging | Reproducible bundle, notices, source materials, and compatibility report |

K-Plex’s existing `verify`, `check:core`, architecture tests, source tests, browser tests, and real Obsidian build should remain part of fork validation. Its upstream documentation explicitly warns that passing architecture checks does not prove second-host portability.[^2_10][^2_2][^2_3]

## Recommended project definition

> A native Hermes Desktop and agent plugin that embeds K-Plex from a pinned forked submodule, preserves its graph semantics and visualization implementations, and supplies Hermes-specific source, navigation, editing, storage, and lifecycle adapters.

The unresolved work is not whether Hermes can host a graph page—it has the documented extension points. It is how much of K-Plex’s remaining application/rendering dependency chain can be separated with a small, maintainable fork delta. That should be the first implementation investigation, rather than starting a new renderer or translating the engine to Python.[^2_2][^2_1]

This research reviewed upstream architecture, build configuration, SDK documentation, and relevant extension discussions. It did not clone, compile, or run either project; the exact reusable import closure, React/portal compatibility, and end-to-end Desktop behavior remain to be validated.

<span style="display:none">[^2_12][^2_13][^2_14][^2_15][^2_16][^2_17][^2_18][^2_19][^2_20][^2_21][^2_22]</span>

<div align="center">⁂</div>

[^2_1]: https://hermes-agent.nousresearch.com/docs/developer-guide/desktop-plugin-sdk

[^2_2]: https://github.com/zsviczian/kplex/blob/main/docs/ARCHITECTURE.md

[^2_3]: https://github.com/zsviczian/kplex/blob/main/AGENTS.md

[^2_4]: https://git-scm.com/docs/git-submodule

[^2_5]: https://github.com/NousResearch/hermes-agent/issues/147146

[^2_6]: https://github.com/NousResearch/hermes-agent/issues/87768

[^2_7]: https://github.com/NousResearch/hermes-agent/issues/143514

[^2_8]: https://github.com/NousResearch/hermes-agent/issues/148224

[^2_9]: https://github.com/zsviczian/kplex/blob/main/esbuild.config.mjs

[^2_10]: https://github.com/zsviczian/kplex/blob/main/package.json

[^2_11]: https://github.com/zsviczian/kplex/blob/main/LICENSE

[^2_12]: https://github.com/NousResearch/hermes-agent/compare/v2026.6.19...v2026.7.1

[^2_13]: https://github.com/NousResearch/hermes-agent/blob/main/website/docs/getting-started/updating.md

[^2_14]: https://github.com/NousResearch/hermes-agent/issues/107312

[^2_15]: https://github.com/NousResearch/hermes-agent/issues/2736

[^2_16]: https://github.com/NousResearch/hermes-agent/issues/116305

[^2_17]: https://github.com/NousResearch/hermes-agent/issues/143515

[^2_18]: https://github.com/NousResearch/hermes-agent/issues/118798

[^2_19]: https://github.com/NousResearch/hermes-agent/issues/141816

[^2_20]: https://github.com/NousResearch/hermes-agent/blob/main/setup-hermes.sh

[^2_21]: https://github.com/NousResearch/hermes-agent/issues/107291

[^2_22]: https://github.com/NousResearch/hermes-agent/tree/main/skills/note-taking/obsidian


---

# continue

The next step is to turn the architecture proposal into a source-level extraction plan. I inspected K-Plex’s actual source tree through GitHub, and found an important opportunity: it already has a versioned automation API with graph, evidence, coverage, configuration, and canvas contracts.

That suggests a stronger design: reuse K-Plex’s existing **contracts** for Hermes agent access, while extracting the existing visual application for Desktop. Do not invent a parallel graph API or renderer unless a specific incompatibility forces it.

There is still no verified drop-in embedding entry point. The documentation explicitly warns that portable graph modules do not make the remaining UI and indexing runtime host-independent.[^3_1]

## 1. New finding: an existing public API can shape the Hermes integration

The GitHub source inspection found:

```text
src/application/
├── PublicApi.ts
├── PublicApiConfiguration.ts
├── PublicApiOntologyConfiguration.ts
├── PublicCanvasRegistry.ts
├── publicApiContracts.ts
└── publicApiMarkdown.ts

src/publicApi/
├── createPublicApi.ts
└── ontologyCompletion.ts
```

In `publicApiContracts.ts`, the current contract declares API version 1 and separates graph operations, canvas operations, revisions, coverage, and configuration.

The particularly useful interfaces are:


| Existing contract | What it exposes | Proposed Hermes use |
| :-- | :-- | :-- |
| `PublicGraphPort` | Node lookup, search, neighbors, evidence, coverage | Agent tools and backend queries |
| `PublicCanvasPort` | List canvases, snapshots, Markdown export, navigation, view/preset selection | Desktop actions and optional agent-assisted navigation |
| `PublicApiRevision` | Semantic, source, configuration, optional canvas revisions | Reject stale requests |
| `PublicCoverage` | Scope, completeness, pending/discovery state | Make incomplete results explicit |
| `PublicIndexStatus` | Index phase, counts, readiness, hydration, search completeness | Status UI and diagnostics |
| `PublicApiResult<T>` | Typed success/failure envelope | Preserve upstream error semantics |

These details came from direct source reads. They are not proof that the production Obsidian API factory can run unchanged in Hermes. Its adapters and construction dependencies still need inspection.

### Why this matters

Previously, I proposed tools such as `plex_search` and `plex_neighborhood` as a fresh integration API. A better implementation is to make those tools thin wrappers around upstream operations:

```text
Hermes tool
    │
    ▼
Transport and argument validation
    │
    ▼
Existing K-Plex public API/use cases
    │
    ▼
Existing semantic and evidence owners
```

That preserves behavior, error handling, limits, and provenance.

K-Plex’s architecture already insists that adapters should not create another graph store, history engine, mutation scheduler, or relationship classifier. This approach follows that constraint.[^3_1]

### Preserve the existing limits

The inspected contract includes bounded page sizes, evidence limits, query budgets, and output limits. Reuse those limits rather than exposing an unrestricted “dump the whole graph” endpoint.

Likewise, do not flatten results into a bare array. A response that omits `coverage`, `truncated`, or revision information can make partial results appear authoritative.

## 2. A concrete rendering obstacle: even individual nodes retain host coupling

I also inspected `src/ui/ThoughtNode.tsx`.

It is valuable existing implementation—not something to replace. It already handles:

- Node and gate presentation.
- Selection and keyboard identity.
- Branch and section disclosure.
- Connection-drag states.
- Badges and coverage hints.
- Embedded content.
- Accessible labels.
- Image-consent checks.
- Context and corner actions.

However, it imports legacy settings/types, styling and layout helpers, a gate-count hook, and `ObsidianIcon`.

That gives us a concrete first extraction target.

### Proposed fork change: extract node presentation without changing behavior

The aim is not to rewrite `ThoughtNode`. It is to make its necessary inputs explicit.


| Current dependency | Proposed treatment |
| :-- | :-- |
| `ObsidianIcon` | Inject a narrow icon-rendering capability |
| Full `KplexSettings` | Supply the subset actually consumed by the component |
| Legacy positioned-node structure | Introduce a narrow presentation contract only where required |
| Host-sensitive count lookup | Supply the existing count presentation through an adapter or portable hook |
| Styling/layout helpers | Move existing pure implementations into admitted portable owners where needed |
| Embedded content | Retain the existing caller-supplied React content slot |

Do this through small changes that keep the Obsidian version working after every commit.

The important distinction is between replacing an implementation and changing its dependency boundary. K-Plex’s architecture already uses the latter strategy for its portable search component and graph contracts.[^3_1]

### Do not strip away subtle behavior to make the prototype compile

For example, preserve:

- Image permission checks before assigning `src`.
- Separation of canonical node identity and occurrence identity.
- Distinct section and branch controls.
- Lower-bound versus complete relationship counts.
- Input isolation for embedded document content.

A visually similar replacement can silently lose these guarantees. The upstream architecture treats coverage, occurrence identity, editor ownership, and evidence authority as separate concerns.[^3_1]

## 3. Revised integration architecture

I would now divide the work into three reusable surfaces.

```text
Your K-Plex fork — pinned Git submodule
│
├── Existing semantic engine
│   ├── parser
│   ├── compiler / incremental patching
│   ├── resolver / evidence
│   └── predicates / lenses
│
├── Existing application/public API
│   ├── graph reads
│   ├── revisions / coverage
│   ├── canvas registry
│   └── configuration use cases
│
└── Existing visual implementation
    ├── projection / layout policies
    ├── scene preparation
    ├── node / connection presentation
    └── interaction and session coordination

Your Hermes integration repository
│
├── Source and host adapters
├── Backend transport / process ownership
├── Agent-tool wrappers
├── Desktop SDK composition
└── Packaging / compatibility tests
```

The existing source organization supports this direction, but the boundaries are uneven: the host-free view definitions are further along than the legacy graph UI and index orchestration.[^3_1]

### Avoid a premature npm-package extraction

You do not need to publish a separate K-Plex npm package to prove this.

Initially:

1. Import fork source through the parent build.
2. Establish a small explicit export surface in the fork.
3. Keep the submodule pinned.
4. Compile the needed dependency closure into the Hermes artifact.
5. Publish a reusable package only if its boundary becomes stable.

This reduces packaging work while preserving the option to upstream a proper embedding API.

## 4. Desktop integration should preserve the K-Plex experience

The first Desktop milestone should be a full workspace using the original implementation—not a miniature new graph assembled from API results.

Hermes provides full-page routes, sidebar navigation, dockable panes, settings, commands, and plugin-scoped backend access.[^3_2]

### Primary surface: full-page K-Plex workspace

Proposed responsibilities:

```text
Hermes shell
├── Route and sidebar navigation
├── Connection/profile selection
├── Backend status
└── Native plugin settings

Embedded K-Plex
├── Graph navigation
├── Existing view policies
├── Search and filters
├── Selection and history
├── Evidence presentation
└── Document-content host
```

Keep the Hermes shell thin. Avoid redesigning K-Plex’s toolbar and navigation merely to make every control look like a new Hermes component.

Use Hermes UI components for integration-specific settings, connection errors, and installation diagnostics; preserve K-Plex components where they embody the original experience.

### Secondary surface: companion pane

After the full workspace works, add a docked pane that shares the runtime but has its own view session.

Do not share all UI state indiscriminately:


| State | Recommended scope |
| :-- | :-- |
| Semantic graph and source revisions | Shared runtime |
| Indexing status | Shared runtime |
| Canvas camera | Per canvas |
| Selection | Per canvas |
| View mode and restoration | Per view session, consistent with upstream behavior |
| Composer target | Explicit Hermes session |
| Desktop layout placement | Hermes-owned |

K-Plex already separates per-document view-session coordination from native workspace/editor ownership. Preserve that distinction rather than replacing it with a global selected-node singleton.[^3_1]

### Capture the intended composer target before asynchronous work

Hermes supports inserting text into an explicitly addressed session composer and fails closed when no matching mounted surface exists.[^3_2]

For “Discuss this node”:

1. Capture the intended session address.
2. Capture the node identity and relevant revision.
3. Fetch the bounded context.
4. Verify the request still belongs to the same connection/profile.
5. Insert into that captured session.
6. Report failure instead of falling back to a different active chat.

This avoids a subtle cross-session mistake when the user changes tabs while context loads.

## 5. The document editor is a separate portability track

The graph can be reused without immediately reproducing Obsidian’s editor integration.

K-Plex’s existing architecture deliberately preserves native editor ownership: Compass and Facets use an attention-following editor, while Mind Map uses a capacity-one occurrence host.[^3_1]

### First release: support document viewing

Recommended starting capabilities:

```text
read document
render document
navigate to source
copy reference
insert reference into Hermes composer
```

Keep unsupported edit actions disabled explicitly.

### Later: implement a document host, not an Obsidian editor clone

A proposed document-host capability should answer:

- Which source is mounted?
- Can it be edited?
- Who owns focus?
- Can it navigate to a heading/block?
- How are unsaved changes handled?
- What happens when the view switches?
- What is disposed when the document or canvas disappears?

The exact API should emerge from existing editor call sites.

Do not implement a new editor just to satisfy a fake `WorkspaceLeaf` abstraction. Conversely, do not claim “full K-Plex support” if the integrated document editing and occurrence behavior remain absent.

## 6. Runtime placement: preserve one semantic authority, not necessarily one execution location

There is a useful refinement to the earlier “single runtime” recommendation.

The invariant should be:

> One authoritative semantic state per configured source scope, with explicitly versioned projections and queries.

That does not mean every operation must run in the same process.

### A practical staged design

| Stage | Runtime arrangement | Purpose |
| :-- | :-- | :-- |
| Portability prototype | Renderer-local fixture runtime | Prove original UI reuse |
| Local functional prototype | Reused TypeScript runtime with local source adapter | Prove source semantics |
| Integrated product | Gateway-owned authoritative runtime, Desktop client | Support agent tools and remote gateways |
| Optional optimization | Renderer-side pure layout/projection | Keep interaction responsive without duplicating classification |

K-Plex’s pure projection/layout policies can remain frontend work while the backend owns semantic publication. The architecture explicitly distinguishes view projection from source acquisition and semantic classification.[^3_1]

### Do not serialize everything blindly

Some inspected contracts contain functions, such as cancellation callbacks. Those are valid in-process capabilities, not JSON transport fields.

For a backend boundary:

- Preserve serializable result shapes.
- Translate request cancellation into transport cancellation or request IDs.
- Keep bounded errors.
- Represent revisions explicitly.
- Never send callbacks, host objects, or mutable runtime instances over the wire.

The adapter should translate the transport, not redefine the underlying semantics.

## 7. Compatibility tests should measure reuse, not just appearance

I would add an explicit reuse ledger to the repository.

```text
Feature:
Upstream implementation owner:
Integration adapter:
Fork modifications:
Characterization fixtures:
Known unsupported behavior:
```


### Example acceptance matrix

| Feature | Required proof |
| :-- | :-- |
| Relationship classification | Same normalized inputs produce equivalent upstream decisions |
| Evidence inspection | Declaration multiplicity and suppression reasons survive |
| Compass layout | Existing policy/geometry executes, not a substitute algorithm |
| Search | Existing ranking implementation retained where extracted |
| View changes | Do not rebuild graph semantics unnecessarily |
| Source update | Stale work cannot overwrite a newer publication |
| Multiple canvases | Independent view state with shared semantic revision |
| Plugin disable | Listeners, timers, UI and subscriptions are released |
| Remote/profile switch | Old results cannot enter the new scope |
| Unsupported writes | Fail explicitly; do not fall back to unsafe source edits |

The upstream architecture warns that architecture/type checks do not establish runtime portability or native behavior. Keep the original production build and tests, then add actual Hermes Desktop acceptance tests.[^3_1]

### Useful negative tests

These often reveal integration bugs sooner than happy-path demos:

- Disable the plugin while a source read is pending.
- Switch gateways during an evidence request.
- Delete or rename a centered document.
- Open the full page and companion pane simultaneously.
- Switch view modes while an editor is mounted.
- Receive incomplete index coverage.
- Try to discuss a node after its target composer unmounts.
- Reload the plugin repeatedly and check for duplicated subscriptions.
- Deny remote images and verify no request is initiated.


## 8. The first fork PRs I would prepare

Keep these independently reviewable.

### PR 1 — Reproducible embedding investigation

Add:

- A fixture-driven embedding harness.
- A dependency/import report.
- A separate experimental build entry.
- No changes to graph behavior.
- No claim of production embedding support.


### PR 2 — Minimal visual presentation extraction

Start with a small dependency closure around existing node presentation and required helpers.

Success criterion: the original component renders in the harness without importing Obsidian, while the Obsidian build remains functional.

### PR 3 — Graph API/runtime composition

Investigate and expose the existing public API against portable runtime capabilities.

Success criterion: bounded search, neighbors, and evidence calls work against fixtures without constructing an Obsidian plugin instance.

### PR 4 — Full Compass scene integration

Reuse existing scene preparation, projection, geometry, and interaction code as far as the dependency audit permits.

Success criterion: a native Hermes page runs the original Compass implementation with documented limitations.

### PR 5 — Real Markdown source adapter

Add source acquisition, identity, link-resolution rules, and incremental invalidation.

Success criterion: equivalent fixtures produce equivalent semantic outputs; adapter-specific link resolution is documented rather than silently assumed.

The ordering can change if the import audit reveals an easier existing composition seam. The point is to keep each extraction justified by an executable consumer.

## 9. What I would not do yet

- Fork Hermes itself.
- Rewrite K-Plex in Python.
- Adopt a replacement graph visualization library.
- Build a broad fake Obsidian API.
- Introduce embeddings or agent-generated relationship inference.
- Promise every K-Plex settings panel is portable.
- Treat source metadata as universally editable.
- Create a new ontology or evidence schema when upstream already has one.
- Claim a Node sidecar is validated before testing packaging and lifecycle.

Those choices either reduce reuse or distract from the actual blocker: the remaining host-bound composition.

## Current research conclusion

The strongest next milestone is:

> Run the original K-Plex Compass implementation inside a real Hermes Desktop page, from a pinned forked submodule, using fixture data and minimal host-capability extraction.

In parallel, validate whether the existing version-one public API can serve Hermes tools without an Obsidian instance. If both probes succeed, the project becomes an adapter-and-extraction effort rather than a reimplementation.

No repository or fork has been created, and no build has been run. The new findings above come from source inspection; runtime compatibility, the exact import closure, and the size of the required fork delta remain unverified.

<div align="center">⁂</div>

[^3_1]: https://github.com/zsviczian/kplex/blob/main/docs/ARCHITECTURE.md

[^3_2]: https://hermes-agent.nousresearch.com/docs/developer-guide/desktop-plugin-sdk


---

# do we need to fork Obsidian, as we'd ideally want to launch just the Hermes app, and have Obsidian (view) integrated somewhere in there, but it's the original Obsidian, so maybe a vendor fork of Obsidian as well?

No—we should not plan on a vendor fork of Obsidian. Unlike K-Plex, the original Obsidian application is proprietary, and its standard terms do not grant permission to fork, modify, or redistribute it. Being free to use—including commercially—does not make it open-source.[^4_1][^4_2]

Your clarification exposes an important distinction: embedding the original Obsidian application is a different project from embedding K-Plex’s existing view inside Hermes. My earlier proposal addressed the latter.

The decision is about **embedding** versus integration:


| Desired experience | Practical route | Original Obsidian running? |
| :-- | :-- | :-- |
| Launch Hermes; use K-Plex inside its window | Port the existing K-Plex implementation through host adapters | No |
| Launch Hermes; automatically open original Obsidian with K-Plex | Companion-app integration | Yes, separately |
| Launch Hermes; see the original Obsidian workspace inside a Hermes pane | Requires an embedding arrangement that we have not established technically or contractually | Yes |
| Vendor and modify Obsidian alongside Hermes | Requires explicit rights from Obsidian; not covered by its standard license | Would be a modified application |

Obsidian documents cross-app automation through its URI protocol, but that opens or controls Obsidian—it does not embed its workspace into another application.[^4_3]

## 1. Why a vendor fork is not the right assumption

Obsidian’s published terms reserve ownership of its application code and expressly restrict modification, derivative works, and redistribution. They contain a limited reverse-engineering exception for developing third-party plugins for non-commercial use; that is not a general right to extract and ship an embeddable Obsidian runtime.[^4_2]

Consequently, we should not:

- Put Obsidian’s application code or packaged binaries into a public vendor repository without permission.
- Extract its editor/workspace implementation and ship it as part of Hermes.
- Assume adding attribution or a license notice creates the necessary rights.
- Assume a private repository automatically resolves those restrictions.[^4_2]

For a distributable product that includes the original app, we would need a written agreement covering the actual embedding, redistribution, and modification activities. That is a licensing gate to review with Obsidian and appropriate counsel, not merely another Git submodule.

## 2. What “the original Obsidian view” could mean

There are two materially different interpretations.

### A. The original K-Plex view, currently hosted by Obsidian

This remains the achievable reuse-first target:

```text
Hermes Desktop
└── K-Plex workspace
    ├── Original K-Plex graph semantics
    ├── Original K-Plex layout/view policies
    ├── Original K-Plex components, where extracted
    └── Hermes document/navigation adapters
```

It does not need the original Obsidian app. However, the Obsidian-native parts—editor mounting, workspace leaves, menus, file operations, and some settings—need host adapters or capability extractions. K-Plex’s own architecture identifies those as remaining host-bound areas.[^4_4]

This is not a promise of zero adaptation. It is a way to preserve K-Plex rather than reproduce it.

### B. The complete original Obsidian workspace

That means retaining Obsidian’s own editor, tabs, workspace behavior, plugins, and application runtime.

My previous portability proposal does not deliver that. If this is the requirement, we should change the architecture rather than describe a K-Plex port as an embedded Obsidian.

## 3. Best route if preserving original Obsidian is the priority

Use a companion architecture first:

```text
User launches Hermes
        │
        ├── Hermes Desktop
        │   └── Knowledge integration pane
        │
        └── Original, user-installed Obsidian
            ├── Original K-Plex plugin
            └── Optional Hermes bridge plugin
```

Obsidian officially supports URIs for opening a vault, opening a note, navigating to headings/blocks, searching, and creating notes. Opening a vault can also focus its existing window. That gives us a documented foundation for coordinated launching and navigation.[^4_3]

### What we gain

As a proposed design, this would preserve the original K-Plex host environment rather than require us to replace its Obsidian-native dependencies.

We could build a bridge for:

- Requesting graph/search/evidence operations where the existing API permits.
- Opening a selected source in Obsidian.
- Linking a Hermes conversation to a note.
- Sending selected, bounded context into Hermes.
- Reporting whether Obsidian and the bridge are available.

The custom bridge and its transport would still need implementation and security review. The built-in URI protocol alone does not provide all those capabilities.

### What we do not gain

This produces one launch workflow, not one application window.

Original Obsidian would remain a separate application. We should not promise that a normal Hermes plugin can take its window and turn it into a stable, cross-platform docked pane.

## 4. Can we put original Obsidian inside a Hermes pane anyway?

There are two different ideas here.

### Officially licensed embedding

This is the clean route if exact original-app-in-Hermes behavior is mandatory.

Ask Obsidian whether it offers or would authorize:

1. An embeddable workspace/editor runtime.
2. Running the original runtime within another desktop shell.
3. Redistribution with a third-party application.
4. A supported integration boundary for plugins such as K-Plex.
5. Branding, updates, security, and support arrangements.

I have not established that such an SDK or agreement exists. We should treat it as a vendor conversation, not an available dependency.

### Hosting or streaming a separate app window

That would be a separate technical investigation, not a K-Plex integration shortcut. It would also need contractual review rather than an assumption that leaving the executable unmodified makes every embedding method permissible.

I would not make it the product’s foundation before proving the target-platform behavior and securing the necessary permissions.

Also, an iframe or Electron webview is not, by itself, a mechanism for embedding an arbitrary desktop application. The documented Obsidian URI interface is an automation interface, not a web-hostable application surface.[^4_3]

## 5. Recommended project structure now

Keep the K-Plex forked submodule, but do not add an Obsidian vendor directory.

```text
hermes-kplex/
├── vendor/
│   └── kplex/                 # Forked, pinned submodule
├── integrations/
│   ├── hermes-desktop/        # Native Hermes UI
│   ├── obsidian-bridge/       # Plugin for original Obsidian
│   └── shared-protocol/       # Bounded messages and identities
├── adapters/
│   ├── embedded-kplex/       # Longer-term standalone hosting
│   └── obsidian-companion/    # Original-app integration
└── docs/
    ├── licensing.md
    └── compatibility.md
```

These are proposed project boundaries, not existing packages.

This leaves two implementation paths open:


| Path | Main advantage | Main limitation |
| :-- | :-- | :-- |
| Original Obsidian + bridge | Maximum preservation of current K-Plex behavior | Separate application/window |
| K-Plex embedded in Hermes | Genuine one-window experience | Requires extraction of remaining Obsidian dependencies |

Hermes already supports the native page/pane side of the second path; the unresolved work is K-Plex’s host boundary, not whether Hermes can display a plugin workspace.[^4_5][^4_4]

## 6. My recommendation for your requirement

Since you explicitly want the original Obsidian, I would revise the plan as follows:

1. Start with original Obsidian as a user-installed companion, keeping K-Plex in its native environment.
2. Build a thin Hermes–Obsidian bridge, using documented navigation capabilities and investigating K-Plex’s existing public API.
3. Make Hermes the entry point, with explicit user configuration for launching/opening the selected Obsidian vault.
4. In parallel, ask Obsidian about licensed embedding if one-window original-app hosting is non-negotiable.
5. Retain the K-Plex submodule portability track as the alternative that can deliver a native Hermes workspace without shipping Obsidian.

The decisive question is: must the original Obsidian editor and workspace appear inside the same Hermes window, or is launching Hermes and automatically opening a coordinated original Obsidian window acceptable?

<span style="display:none">[^4_10][^4_11][^4_12][^4_13][^4_14][^4_15][^4_16][^4_17][^4_18][^4_6][^4_7][^4_8][^4_9]</span>

<div align="center">⁂</div>

[^4_1]: https://obsidian.md/license

[^4_2]: https://obsidian.md/terms

[^4_3]: https://help.obsidian.md/Extending+Obsidian/Obsidian+URI

[^4_4]: https://github.com/zsviczian/kplex/blob/main/docs/ARCHITECTURE.md

[^4_5]: https://hermes-agent.nousresearch.com/docs/developer-guide/desktop-plugin-sdk

[^4_6]: https://www.obsidian.net/terms-of-use

[^4_7]: https://github.com/NousResearch/Obsidian/blob/main/LICENSE

[^4_8]: https://github.com/open-source-labs/obsidian/blob/master/LICENSE

[^4_9]: https://obsidianstudio.design/pages/terms-conditions

[^4_10]: https://www.obsidian-cheats.com/tos

[^4_11]: https://obsidianri.com/terms

[^4_12]: https://web.obsidianscheduler.com/download/

[^4_13]: https://github.com/Dewey-Wang/obsidian-acadamic/blob/main/.obsidian/plugins/sheet-plus/LICENSE

[^4_14]: https://github.com/obsmd-projects/obsidian-projects/blob/main/LICENSE

[^4_15]: https://github.com/obsidian-nvim/obsidian.nvim/blob/main/LICENSE

[^4_16]: https://github.com/nadezhdkov/Obsidian/blob/main/LICENSE.md

[^4_17]: https://github.com/ObsidianMC/Obsidian/blob/1.21.x/LICENSE

[^4_18]: https://github.com/Everblush/Obsidian/blob/main/LICENSE


---

# break down the architecture and nesting and research more

The architecture has four different kinds of nesting, and separating them resolves most of the confusion:

1. Repository nesting: K-Plex can live in your repository as a forked submodule.
2. Runtime nesting: K-Plex normally runs as a plugin inside Obsidian.
3. Visual nesting: a view can appear inside a Hermes pane without the original Obsidian application running there.
4. Application embedding: placing the actual Obsidian workspace inside Hermes is a separate, unverified integration—not something a submodule or webview automatically enables.

The central distinction is **ownership**: who owns the window, the editor, the graph runtime, and the files?

My recommendation is to design two explicit backends—original Obsidian as a companion, and portable K-Plex as a native Hermes integration—rather than pretend they are the same implementation.

## 1. The original architecture: what actually lives inside what

Obsidian’s official API documentation identifies these application-level owners:

- `App`: the application object.
- `Vault`: files and folders.
- `Workspace`: panes and workspace views.
- `MetadataCache`: headings, links, embeds, tags, and block metadata.
- Plugins: register views, commands, settings, and lifecycle-managed events.[^5_1]

Conceptually, the original system is:

```text
Operating system
└── Original Obsidian application
    ├── Application/runtime
    ├── App
    │   ├── Vault
    │   ├── MetadataCache
    │   └── Workspace
    │       └── Workspace pane / registered view
    │           └── K-Plex view
    │               ├── Graph rendering
    │               ├── Navigation and interaction
    │               └── Integration with native document/editor hosts
    │
    └── K-Plex plugin lifecycle
        ├── Index/runtime coordination
        ├── Settings and commands
        ├── Obsidian source adapters
        └── Portable semantic modules
```

K-Plex’s architecture confirms that it already has portable semantic and view-policy modules, while parts of indexing, the legacy graph UI, native editors, and workspace ownership remain coupled to Obsidian.[^5_2]

The important consequence:

> K-Plex does not contain Obsidian. Obsidian hosts K-Plex and supplies capabilities it expects.

Copying K-Plex into Hermes therefore does not bring Obsidian’s editor, metadata cache, workspace, or plugin ecosystem with it.

### The public Obsidian repository is not the application runtime

The official `obsidianmd/obsidian-api` repository contains API type definitions and documentation. It is not an embeddable implementation of the Obsidian application. Its documentation describes importing `obsidian` from a plugin running within the host.[^5_1]

So this nesting would not work as imagined:

```text
Hermes
└── npm install obsidian
    └── Original Obsidian workspace   # Not supplied by the API definitions
```


## 2. Hermes has a different ownership tree

Hermes Desktop provides a contribution registry for pages, panes, navigation, settings, and commands. A plugin contributes React UI into that registry and can communicate with its own backend namespace.[^5_3]

The corresponding architecture is:

```text
Operating system
└── Hermes Desktop application
    ├── Desktop shell
    │   ├── Routes and sidebar
    │   ├── Layout panes
    │   ├── Chat and composers
    │   └── Plugin registry
    │       └── Our Desktop plugin
    │           ├── Knowledge page
    │           ├── Companion pane
    │           └── Integration settings
    │
    └── Connection to Hermes gateway
        └── Our Python/backend plugin
            ├── Agent tools
            ├── Plugin API
            └── Knowledge backend adapter
```

Hermes’s Desktop and Python plugin APIs are distinct. A unified plugin package can ship both, but the frontend and backend remain separate execution contexts. Desktop plugins can also connect to different profiles, gateways, and remote machines.[^5_3]

That gives us a supported place to put a knowledge interface. It does not, by itself, supply an Obsidian host.

## 3. Three viable product architectures

### Architecture A — Original Obsidian companion

This preserves the most original behavior.

```text
User launches Hermes
        │
        ▼
Hermes Desktop
├── Chat
├── Knowledge control pane
│   ├── Connection status
│   ├── Search/evidence results
│   └── Open selected source
└── Hermes backend plugin
          │
          │ Authenticated bridge — proposed
          ▼
Original Obsidian application
├── Obsidian workspace and editor
├── Original K-Plex implementation
└── Our bridge plugin
          │
          ▼
User's vault
```

Visual result:

```text
┌───────────────────────┐   ┌─────────────────────────┐
│ Hermes                │   │ Original Obsidian       │
│                       │   │                         │
│ Chat + integration    │   │ K-Plex + native editor  │
└───────────────────────┘   └─────────────────────────┘
```


#### What is reused?

- Original Obsidian editor and workspace.
- K-Plex’s current host integration.
- Existing native behavior and plugin environment.
- K-Plex’s semantic implementation.


#### What must be built?

- Obsidian bridge plugin.
- Hermes tool/backend adapter.
- Hermes integration UI.
- Pairing, capability checks, and lifecycle handling.

Obsidian’s URI protocol already supports opening vaults and notes, search, and navigation to headings or blocks. It also supports choosing a tab, split, or separate window for note opening. Those are useful launch/navigation primitives, not a complete graph-query bridge.[^5_4]

#### Main limitation

It is one coordinated workflow, but two applications and ordinarily two windows.

This is the lowest-adaptation route if “original Obsidian behavior” is the highest priority.

______________________________________________________________________

### Architecture B — K-Plex embedded natively in Hermes

This delivers the clearest one-window product without requiring Obsidian embedding.

```text
Hermes Desktop
├── Chat
└── Knowledge workspace
    └── Reused K-Plex visual implementation
        ├── Original projection/layout policies
        ├── Existing graph components where portable
        └── Hermes host capabilities
                    │
                    ▼
Hermes backend plugin
└── Reused K-Plex runtime
    ├── Parser
    ├── Compiler and evidence
    ├── Public API/application use cases
    └── New source adapter
                    │
                    ▼
Markdown knowledge directory / supported sources
```

Visual result:

```text
┌────────────────────────────────────────────────────┐
│ Hermes                                             │
├──────────────────────┬─────────────────────────────┤
│ Chat                 │ K-Plex workspace            │
│                      │                             │
│                      │ Reused K-Plex visualization │
├──────────────────────┴─────────────────────────────┤
│ Optional document/source panel                     │
└────────────────────────────────────────────────────┘
```


#### What is reused?

K-Plex’s existing portable modules and as much application/rendering implementation as can be extracted. Its architecture explicitly supports new host adapters supplying normalized source facts.[^5_2]

#### What is not present?

The original Obsidian application.

A Hermes document panel may render or edit Markdown, but it must not be presented as the original Obsidian editor.

#### Main limitation

We need to finish selected host-capability extractions. This is real engineering work, though it need not become a rewrite.

______________________________________________________________________

### Architecture C — Original Obsidian workspace inside Hermes

This is the exact interpretation of your proposed nested application:

```text
Hermes Desktop
├── Hermes chat
└── Embedded original Obsidian workspace
    ├── Original editor
    ├── Original workspace
    └── Original K-Plex
```

It is a product objective, not an established implementation.

It requires answering two independent questions:

1. Does Obsidian authorize the required hosting/distribution/modification arrangement?
2. Is there a supported technical way to host its runtime in Hermes?

The standard Obsidian terms restrict redistribution and derivative modifications, and the public API repository does not supply the application implementation. A vendor agreement would need to cover the actual arrangement; attribution alone is insufficient.[^5_5][^5_1]

I would not make this architecture a dependency of the initial project.

## 4. Why an Electron webview does not solve the nesting

The deeper Electron documentation makes this distinction clear.

`WebContentsView` displays an Electron `WebContents`. It can create a new web-content instance or adopt an existing `WebContents` supplied to it. It is a main-process API.[^5_6]

That supports:

```text
Hermes main process
└── WebContentsView
    └── Web page/application content owned by that runtime
```

It does not document a mechanism for:

```text
Hermes main process
└── WebContentsView
    └── Attach arbitrary running Obsidian desktop application
```

The fact that two applications use web technologies does not make their application runtimes interchangeable.

### There is also a Hermes plugin boundary

Hermes disk plugins import only the SDK and React modules permitted by the host. They do not get arbitrary access to Electron main-process APIs.[^5_3]

Therefore, a design requiring us to create native `WebContentsView` instances is not automatically achievable as an ordinary disk plugin. It would need an appropriate host capability or app-level development.

That is separate from the native React K-Plex page, which fits the documented plugin surface.

### Security must not be weakened to force compatibility

Electron recommends context isolation, renderer sandboxing, restricted navigation, validated IPC senders, and no Node integration for remote content. It explicitly warns against exposing raw Electron APIs to untrusted content.[^5_7]

A solution that works only after disabling these protections is not an acceptable integration baseline.

## 5. Repository nesting is independent of all of the above

Your desired submodule strategy remains sensible:

```text
our-hermes-knowledge-repository/
├── vendor/
│   └── kplex/                   # Pinned fork
├── packages/
│   ├── hermes-desktop/
│   ├── hermes-backend/
│   ├── obsidian-bridge/
│   └── protocol/
├── adapters/
│   ├── obsidian-companion/
│   └── standalone-kplex/
└── docs/
```

But this only describes source ownership.

```text
Git submodule ≠ process
Git submodule ≠ plugin installation
Git submodule ≠ embedded application
Git submodule ≠ redistribution permission
```

For Architecture A, the build produces an Obsidian plugin/bridge installed into the user’s original Obsidian environment.

For Architecture B, the build produces a Hermes plugin bundle containing reused K-Plex implementation.

The same fork can support both, provided the fork preserves the original Obsidian build and keeps Hermes-specific dependencies out of reusable modules.

## 6. Recommended shared architecture: two backends, explicit capabilities

To avoid throwing work away, define a small integration-facing boundary.

These names are proposed, not existing upstream APIs:

```ts
interface KnowledgeBackend {
  capabilities(): Promise<KnowledgeCapabilities>;
  status(): Promise<KnowledgeStatus>;
  search(request: SearchRequest): Promise<SearchResult>;
  neighbors(request: NeighborsRequest): Promise<NeighborsResult>;
  evidence(request: EvidenceRequest): Promise<EvidenceResult>;
}
```

Then implement:

```text
Hermes tools and integration controls
                │
                ▼
        KnowledgeBackend
           ┌────┴─────┐
           ▼          ▼
Original Obsidian     Standalone K-Plex
bridge adapter       runtime adapter
```

Do not invent a new semantic model behind this interface. Reuse upstream result contracts where practical and add transport/connection identity around them.

### Advertise capabilities rather than imitate missing features

For example:

```text
canQueryGraph
canExplainEvidence
canNavigateOriginalObsidian
canNavigateEmbeddedCanvas
canReadSource
canEditSource
canInspectCanvas
```

These are different authorities.

A connection that can search a graph should not implicitly gain note-write permissions or arbitrary command execution.

### Do not promise identical visual behavior across the backends

The companion backend can control or inspect an original K-Plex instance if the bridge supports it.

The native backend can render the extracted implementation directly.

A public canvas snapshot is not automatically a lossless representation of the complete interactive application. We should not assume that transmitting snapshots recreates every editor, gesture, menu, or lifecycle behavior.

## 7. File ownership: avoid two competing writers

A particularly important nesting question is not visual: where do the authoritative notes live?

### Companion mode

Recommended ownership:

```text
Obsidian owns vault-aware operations
    │
    ├── Reads/writes through supported plugin capabilities
    ├── Metadata and workspace updates
    └── K-Plex reacts to source changes

Hermes requests bounded operations through the bridge
```

Obsidian exposes `Vault`, `MetadataCache`, and `Workspace` as separate host services. Use those documented boundaries rather than manipulate its internal objects or infer state from the UI.[^5_1]

### Standalone mode

Recommended ownership:

```text
Standalone source adapter owns configured sources
    │
    ├── Explicit source roots
    ├── Revision checks
    ├── Validated source edits
    └── Incremental graph invalidation
```

If users also open those files in Obsidian, edits must be treated as concurrent external changes—not assumed to be coordinated.

### Avoid this design

```text
Hermes directly edits files
        +
Obsidian editor has unsaved state
        +
Bridge separately edits the same document
```

A shared directory is not a concurrency protocol. Pick an operational write authority per mode and use revision/conflict checks.

## 8. Local versus remote execution changes the bridge

Hermes Desktop’s plugin installation is app-level, while its backend connection can target different gateways or machines.[^5_3]

So this local diagram:

```text
Hermes gateway ── localhost ── Obsidian
```

is valid only when both endpoints actually run on the same machine.

For a remote gateway:

```text
Local machine                         Remote machine
┌────────────────────┐                ┌─────────────────────┐
│ Hermes Desktop     │◄──────────────►│ Hermes gateway      │
│                    │                │ Agent tools         │
│ Original Obsidian  │                └─────────────────────┘
└────────────────────┘
```

We need an explicit additional design to make local Obsidian available to remote agent tools. A renderer-only connection is not sufficient for headless backend access.

My recommended rollout:

1. Support a local gateway and local Obsidian first.
2. Bind the bridge narrowly and require pairing/authentication.
3. Keep remote access disabled by default.
4. Later add an explicitly authorized relay or remote-access design.
5. Scope every request to connection, vault, capability set, and revision.

Do not expose a vault bridge on a public interface as a shortcut.

## 9. Launch nesting is not shutdown ownership

“One launcher” need not mean “one process.”

A proposed companion startup sequence:

```text
Open Hermes
    ↓
Load integration
    ↓
Check whether bridge is reachable
    ├── Yes → attach to existing session
    └── No  → offer to open configured Obsidian vault
                    ↓
             Wait for bridge handshake
                    ↓
             Verify versions/capabilities
                    ↓
             Enable knowledge actions
```

Opening a configured vault through an Obsidian URI is documented; opening an already-open vault focuses its window.[^5_4]

Recommended shutdown rules:

- Disabling our integration disconnects its bridge session.
- Closing a Hermes pane does not close Obsidian.
- Closing Hermes does not automatically terminate a user’s pre-existing Obsidian process.
- A bridge failure leaves Obsidian and its vault usable.
- Backend-dependent controls fail explicitly rather than silently switch modes.

Hermes already provides lifecycle-tracked registrations, sockets, events, timers, and disposal hooks for the plugin side.[^5_3]

## 10. Decision matrix

| Requirement | Original Obsidian companion | Native K-Plex in Hermes | Licensed original-app embedding |
| :-- | :-- | :-- | :-- |
| Original Obsidian editor | Yes | No | Intended, subject to agreement and implementation |
| Original K-Plex host behavior | Preserved most directly | Requires portability work | Potentially, but unverified |
| Single Hermes window | No | Yes | Intended, unverified |
| Fork Obsidian required | No | No | Not necessarily; vendor arrangement determines approach |
| K-Plex fork useful | Optional for bridge changes | Yes | Depends on integration |
| Ordinary Hermes plugin route | Yes for integration UI/backend | Yes, within SDK capabilities | Not established |
| Headless knowledge queries | Depend on Obsidian availability | Can be designed into runtime | Not established |
| Main blocker | Bridge and lifecycle | Host extraction | Vendor rights plus technical embedding |

## Recommended direction

I would build the project around two named modes:

- Companion mode: original Obsidian + original K-Plex, coordinated from Hermes.
- Native mode: reused K-Plex implementation embedded in Hermes, without claiming to be Obsidian.

Start with companion mode if preserving the original editor and behavior is essential. Start with native mode if one-window operation is essential. Keep the forked K-Plex submodule shared between the relevant build targets, and treat original-app embedding as a separate vendor-backed investigation.

The further research confirms the key boundary: Obsidian’s public API is a plugin contract, Electron’s web-content view is not an arbitrary desktop-app container, and Hermes’s plugin SDK is a supported UI/backend contribution surface—not unrestricted access to its desktop main process.[^5_6][^5_1][^5_3]

<div align="center">⁂</div>

[^5_1]: https://github.com/obsidianmd/obsidian-api

[^5_2]: https://github.com/zsviczian/kplex/blob/main/docs/ARCHITECTURE.md

[^5_3]: https://hermes-agent.nousresearch.com/docs/developer-guide/desktop-plugin-sdk

[^5_4]: https://help.obsidian.md/Extending+Obsidian/Obsidian+URI

[^5_5]: https://obsidian.md/terms

[^5_6]: https://www.electronjs.org/docs/latest/api/web-contents-view

[^5_7]: https://www.electronjs.org/docs/latest/tutorial/security


---

# [https://github.com/HermesKPlex/HermesKPlex](https://github.com/HermesKPlex/HermesKPlex), continue@GitHub

Setting up the initial repository plan

---

# @GitHub continue, we forked [https://github.com/HermesKPlex/kplex-Vendor](https://github.com/HermesKPlex/kplex-Vendor), turns out Obsidian is not open-source, so we'll be using the original app

Confirming the Obsidian companion architecture choice

---

# @GitHub continue, but plan more, don't make direct changes from Perplexity to GitHub, just propose architectural and code changes, Hermes style and easily pluggable, research more original source code etc. etc. etc. as most extensive and comprehensive as possible, layouts, compilers, packaging, GitHub actions

Understood. I will use GitHub read-only: no commits, branches, issues, pull requests, fork modifications, or other repository changes. The earlier write proposals are superseded.

The deeper source review changes the implementation recommendation substantially: start with a Hermes plugin that calls K-Plex’s existing public API through the official Obsidian CLI. Do not build a network bridge first.

That is the most **reuse-first** architecture: the original Obsidian application continues to own the editor, workspace, indexing, semantic compiler, layouts, and K-Plex UI. Hermes adds bounded tools and a native companion interface—not a replacement implementation.

Below is a proposed engineering specification covering runtime architecture, source-level integration, layouts, compilers, packaging, security, testing, and GitHub Actions.

## 1. Executive findings: what changed after inspecting the source

I read the following directly through the GitHub connector:


| Repository | Files inspected |
| :-- | :-- |
| `HermesKPlex/kplex-Vendor` | `package.json`, `esbuild.config.mjs` |
| Vendor fork | `src/publicApi/createPublicApi.ts` |
| Vendor fork | `src/core/graph/compiler.ts` |
| Vendor fork | `tests/public-api-window.test.mjs` |
| Vendor fork | `docs/PUBLIC_API.md`, `docs/CLI.md` |
| Vendor fork | `.github/workflows/verify.yml`, `release.yml` |
| Hermes | Desktop SDK documentation source |
| Hermes | Public Python plugin API documentation |

These were source reads, not executed tests. The GitHub connector supplied file blob hashes, not a verified repository-wide commit baseline. We must still select exact commit revisions before implementation.

### Finding A — K-Plex already documents the integration route we need

The vendor’s `docs/CLI.md` explicitly describes calling its public API through:

```text
Obsidian CLI eval
    → window.kplexApi.v1
    → existing K-Plex application owners
```

The public API documentation establishes:

- A public root at `window.kplexApi`.
- Supported API versions.
- Capability discovery.
- Graph search, nodes, neighbors, evidence, and coverage.
- Canvas discovery and bounded scene snapshots.
- Explicit canvas navigation and view/preset selection.
- Mind Map Markdown export.
- Lifetime invalidation after plugin reload.
- No built-in HTTP or MCP listener.

This means the proposed custom Obsidian bridge is optional, not foundational.

The official Obsidian CLI independently confirms that it supports JavaScript evaluation inside the running application, explicit vault targeting, plugin inspection, and workspace commands.[^8_1]

### Finding B — We should not extract or run the graph compiler separately

The original app is now the chosen host. Therefore:

```text
Do not build:
Hermes → second compiler → second graph → second renderer
```

Instead:

```text
Build:
Hermes → bounded adapter → existing K-Plex API
```

This avoids duplicating the exact areas that make K-Plex valuable.

### Finding C — The fork’s current package metadata no longer says MIT

The vendor `package.json` inspected in this turn declares:

```json
"license": "AGPL-3.0-only"
```

That corrects my earlier statement about the inspected package metadata declaring MIT. Do not carry that stale discrepancy forward as if it were still present in this file.

A complete license/provenance review is still needed for the selected revision and distributed artifacts, but the current package metadata is now explicit.

### Finding D — Hermes is well suited to a standalone companion package

Hermes supports a unified plugin containing Python agent/backend code and `desktop/plugin.js`. Its Desktop SDK supplies native UI components, contribution-based pages and panes, scoped backend calls, lifecycle disposal, and session-addressed composer insertion.[^8_2]

We do not need to fork Hermes for this baseline.

______________________________________________________________________

## 2. Recommended product architecture

### 2.1 Primary architecture: official CLI adapter

```text
USER'S MACHINE
│
├── Hermes Desktop
│   └── HermesKPlex Desktop plugin
│       ├── Connection and diagnostics
│       ├── Knowledge search
│       ├── Relationship/evidence inspector
│       ├── Canvas selector
│       ├── Context preview
│       └── Insert context into chosen chat
│
├── Hermes CLI / gateway
│   └── HermesKPlex Python plugin
│       ├── Agent tools
│       ├── Scoped backend API
│       ├── Request validation
│       ├── Capability policy
│       ├── CLI process adapter
│       └── Bounded output decoder
│                    │
│                    │ argument-array subprocess
│                    ▼
└── Original Obsidian application
    ├── Official Obsidian CLI connection
    ├── Selected vault
    ├── Native workspace and editors
    └── K-Plex plugin
        ├── window.kplexApi.v1
        ├── Published graph and evidence
        ├── Source/index runtime
        ├── Original layouts
        └── Original React visualization
```

Obsidian’s CLI requires the application runtime; its documentation says that the first command launches Obsidian if it is not already running. It is not a standalone headless K-Plex engine.[^8_1]

### 2.2 Optional later architecture: persistent bridge transport

```text
Hermes plugin
    │
    └── KnowledgeTransport
        ├── ObsidianCliTransport       ← first implementation
        └── ObsidianBridgeTransport    ← optional, measured need only
```

Reasons to consider the bridge later:

- Measured CLI startup overhead becomes material.
- We need event subscriptions instead of polling.
- Several simultaneous clients require coordinated access.
- CLI output framing remains unreliable on a supported platform.
- A more restrictive dedicated RPC endpoint is desirable.

A bridge should still call the existing public API. It must not become another compiler or graph owner.

### 2.3 No new Node sidecar in the baseline

Earlier proposals considered a TypeScript sidecar. With original Obsidian hosting K-Plex, that is unnecessary for the first product.

Runtime dependencies become:


| Component | Runtime |
| :-- | :-- |
| Hermes tools/backend | Hermes-managed Python environment |
| Desktop companion UI | Hermes’s existing frontend runtime |
| K-Plex | Original Obsidian plugin runtime |
| Vendor build tooling | Node, during development/release |
| Independent graph service | Not required |

This is simpler to install and more faithful to the original.

______________________________________________________________________

## 3. Ownership and nesting, precisely

### 3.1 Source ownership

```text
HermesKPlex/HermesKPlex
├── Our integration source
└── vendor/kplex
    └── Git submodule → HermesKPlex/kplex-Vendor
```


### 3.2 Installation ownership

```text
Hermes installation
└── HermesKPlex plugin package

Obsidian vault configuration directory
└── K-Plex installation
```

The submodule is not the installed plugin. Build and installation are separate operations.

### 3.3 Runtime ownership

| State or behavior | Owner |
| :-- | :-- |
| Note contents | Obsidian/vault and user |
| Native unsaved buffers | Obsidian editors |
| Metadata and link resolution | Existing Obsidian/K-Plex owners |
| Relationship semantics | K-Plex |
| Evidence precedence | K-Plex |
| Graph publication | K-Plex |
| Compass/Mind Map/Facets geometry | K-Plex |
| Camera and canvas selection | Specific K-Plex canvas |
| Tool policy and request limits | HermesKPlex |
| Hermes layout placement | Hermes Desktop |
| Composer draft | Hermes |
| Process transport | HermesKPlex CLI adapter |

K-Plex’s documented architecture deliberately separates semantic compilation, view projection, application coordination, and native editor ownership. We should preserve those boundaries.[^8_3]

### 3.4 Visual ownership

The initial product should say:

> “K-Plex companion for Hermes.”

It should not say:

> “Obsidian embedded in Hermes.”

The original graph and editor stay in the original application. Hermes shows controls, structured results, evidence, and selected context.

______________________________________________________________________

## 4. Source-level integration: use the actual public API

The vendor’s `docs/PUBLIC_API.md` is the implementation contract to characterize first.

### 4.1 API acquisition

The documented entry point is:

```javascript
const root = window.kplexApi;
const api = root?.v1;
```

The same API root is published into supported Obsidian windows. The inspected window-publication tests explicitly cover:

- Main window.
- Existing popouts.
- Newly opened popouts.
- Removal of aliases when a popout closes.
- Disposal on unload.
- Fresh publication after reload.
- Startup before the workspace layout is ready.

This is substantially better than reaching into private plugin internals.

### 4.2 Capability discovery before optional operations

Proposed adapter sequence:

```text
Acquire API root
    ↓
Check supported root version
    ↓
Call capabilities()
    ↓
Verify required operation exists
    ↓
Validate request
    ↓
Execute bounded operation
    ↓
Validate result envelope
```

Do not assume every additive API-v1 capability is present just because the root says version 1.

### 4.3 Preserve the upstream envelope

The documented success shape is:

```ts
{
  ok: true;
  value: unknown;
  revision: {
    semantic: number;
    source: number;
    configuration: number;
    canvas?: number;
  };
  coverage: unknown;
  truncated: boolean;
}
```

Failure:

```ts
{
  ok: false;
  error: {
    code: string;
    message: string;
  };
  revision: unknown;
}
```

Do not convert this into:

```json
{"items": []}
```

That would discard whether the result is incomplete, stale, unavailable, or truncated.

### 4.4 Existing read operations to expose

| Upstream method | Proposed use |
| :-- | :-- |
| `capabilities()` | Feature detection |
| `status()` | Setup and diagnostics |
| `coverage()` | Explain completeness |
| `ontology()` | Exact configured relationship vocabulary |
| `ontologyGuide()` | Bounded agent guidance |
| `node()` | Exact node lookup |
| `search()` | Ranked knowledge search |
| `neighbors()` | Canonical relationships |
| `evidence()` | Explain one connection |
| `canvases()` | Explicit canvas selection |
| `canvas()` | Current materialized scene |
| `context()` | Scene-derived Markdown context |
| `exportMarkdown()` | Existing Mind Map exporter |

### 4.5 Keep actions separate

These are not passive reads:

```text
navigate
selectView
selectPreset
configuration.apply
applyOntologyConfiguration
```

For the initial implementation:

- Allow passive reads.
- Make navigation/view changes explicit user actions or separately approved tool actions.
- Disable configuration mutations.
- Do not expose generic JavaScript execution.

The public API offers more authority than the first Hermes integration should grant.

______________________________________________________________________

## 5. CLI transport: how to make it safe and robust

The official CLI supports:

```shell
obsidian vault=<name-or-id> eval code=<javascript>
```

Vault selection must precede the command. If omitted, Obsidian may choose a vault based on working directory or active vault, which is unsuitable for a multi-vault integration.[^8_1]

### 5.1 Always use argument arrays

Proposed Python execution:

```python
argv = [
    executable,
    f"vault={configured_vault}",
    "eval",
    f"code={generated_code}",
]
```

Never:

```python
command = f"obsidian vault={vault} eval code='{code}'"
```

Never invoke a shell for tool-supplied values.

### 5.2 Never accept an `eval` tool parameter

Bad:

```json
{
  "tool": "kplex_eval",
  "code": "..."
}
```

Recommended:

```json
{
  "tool": "kplex_search",
  "query": "project architecture",
  "limit": 20
}
```

The adapter chooses a fixed operation. User input is serialized as data.

### 5.3 Fixed dispatcher

Illustrative proposed JavaScript, not an existing exported upstream helper:

```javascript
const reads = {
  status: (api, args) => api.status(args),
  search: (api, args) => api.search(args),
  node: (api, args) => api.node(args),
  neighbors: (api, args) => api.neighbors(args),
  evidence: (api, args) => api.evidence(args),
  canvases: (api) => api.canvases(),
  canvas: (api, args) => api.canvas(args),
  context: (api, args) => api.context(args),
};

async function dispatch(request) {
  const root = window.kplexApi;

  if (!root?.v1) {
    return {
      ok: false,
      error: {
        code: "API_UNAVAILABLE",
        message: "K-Plex API v1 is unavailable",
      },
    };
  }

  if (!Object.hasOwn(reads, request.operation)) {
    throw new Error("Unsupported operation");
  }

  return await reads[request.operation](root.v1, request.arguments);
}
```

Production code must also validate root version, advertised capabilities, argument schemas, and output bounds.

### 5.4 Robust serialization

The process builder should:

1. Validate operation-specific arguments.
2. Serialize the request with a real JSON encoder.
3. Embed it as a data literal—not executable user text.
4. Keep the generated program deterministic.
5. Include a request identifier for framing.
6. Bound request size before spawning.

A request-specific output marker can help distinguish our result from CLI banners or incidental logs. However, framing must be characterized against actual Windows/macOS/Linux CLI behavior; do not assume a proposed parser is already compatible.

### 5.5 Output parsing needs more than exit status

The vendor’s CLI documentation warns that CLI output can contain an `Error:` even when the process exits with zero status. Its sample also handles a leading `=>`.

Therefore check:

- Spawn failure.
- Timeout.
- Process exit status.
- Output-size limit.
- Framing/JSON validity.
- Bridge envelope.
- Upstream `ok`.
- GraphQL field errors if GraphQL is ever exposed.

Prefer direct API methods initially instead of arbitrary GraphQL.

### 5.6 Bound subprocess resources

Proposed policy:


| Resource | Policy |
| :-- | :-- |
| Concurrent CLI evaluations | Small per-vault queue; start with one |
| Process deadline | Finite and configurable within bounds |
| Stdout/stderr | Bounded while reading, not only afterward |
| Input size | Validate before spawning |
| Repeated status polling | Visible UI only; backoff when unavailable |
| Retries | Limited passive reads only |
| Actions | No blind replay after uncertain completion |

A process timeout does not establish that an action never happened.

### 5.7 Platform handling

The official docs distinguish:

- Windows: `Obsidian.com` console redirector.
- macOS: registered CLI symlink.
- Linux: registered CLI binary.[^8_1]

Use a configured executable path or documented discovery routine. Do not assume Hermes’s background process inherits the same PATH as an interactive terminal.

### 5.8 First-run launch behavior must be explicit

Because a CLI command can launch Obsidian, even a “status check” can produce a visible application side effect.[^8_1]

Proposed setting:

```text
Allow integration requests to launch Obsidian: off by default
```

The exact non-launching detection mechanism needs platform investigation. Until that exists, label the initial connection test accurately rather than silently claiming it is passive.

______________________________________________________________________

## 6. Compiler architecture: retain the original pipeline

There are two meanings of “compiler” here.

### 6.1 Semantic compiler

The inspected `src/core/graph/compiler.ts` implements the source-to-evidence compiler.

Its actual classes include:

- `NormalizedGraphCompiler`.
- `GraphCompilerSourceRead`.
- `PortableGraphCompilation`.

It uses:

- Normalized source batches.
- Captured source boundaries.
- Reference policy selection.
- Private evidence accumulation.
- Cooperative yielding.
- Current-generation checks.
- Evidence ordering and suppression.
- Resolver finalization.

Conceptually:

```text
Obsidian sources and caches
          ↓
Existing source acquisition / normalization
          ↓
Normalized source facts
          ↓
Reference selection under captured settings
          ↓
Private compilation
          ↓
Evidence ordering / suppression / resolution
          ↓
Published semantic state
          ↓
Public API reads
```


### 6.2 Important invariants to preserve

From the compiler inspection:

- Node IDs are primary identities; paths remain separate facets.
- Private compilation state must not leak as a partially published graph.
- Rejected/stale work is discarded.
- Open source reads must finish before finalization.
- Evidence append authority is sealed before finalization escapes.
- Node-vocabulary projection is not proof of complete relationship coverage.

Hermes should consume the published results, not bypass these publication guards.

### 6.3 “Refresh” should not secretly mean “reindex”

The public API documentation says passive graph reads do not trigger source acquisition, Markdown reads, hydration, or semantic rebuilding.

Therefore the Hermes UI should distinguish:

```text
Refresh displayed result
```

from any future:

```text
Request indexing work
```

Do not add a “refresh” button that reaches into private index methods.

### 6.4 Coverage is part of the product, not debug metadata

The upstream API distinguishes:

- Configuration coverage.
- Global graph coverage.
- Local/neighborhood coverage.
- Cached coverage.
- Unknown coverage.
- Search readiness.
- Source readiness.

A partial empty result must be displayed as:

> No matches in the currently available index scope.

Not:

> This vault contains no matching notes.

______________________________________________________________________

## 7. Layout architecture: preserve three different scopes

This is particularly important for agent reasoning.

### Scope 1 — Canonical graph

```text
Nodes + semantic relationships + evidence
```

Queried through `node`, `neighbors`, and `evidence`.

### Scope 2 — Materialized canvas scene

```text
One view's admitted occurrences,
filters, lenses, folds, sections, groups, and connections
```

Queried through `canvas`.

### Scope 3 — Visible viewport

```text
The subset currently inside the camera/scroll viewport
```

A node can exist in the canonical graph without being admitted to a scene, and an admitted occurrence can be offscreen.

The public API documents these scopes separately. Do not collapse them into a single “visible graph.”

### 7.1 Retain the original view implementations

The public API uses these view IDs:


| Product view | ID |
| :-- | :-- |
| Compass | `classic` |
| Mind Map | `mindmap` |
| Facets | `wuxing` |

The original app continues rendering them. Hermes can offer explicit view-selection controls without reproducing the layouts.

### 7.2 Do not treat decorative guides as relationships

Canvas connections distinguish semantic, structural, and decorative kinds.

A Facets guide can explain placement without being relationship evidence. Agent summaries must preserve that distinction.

### 7.3 Canvas selection must be explicit

The API deliberately returns `AMBIGUOUS_CANVAS` when multiple canvases exist and no unique target is supplied.

Proposed Hermes behavior:

```text
No canvas:
    Explain that no canvas is currently available.

One canvas:
    Offer it explicitly.

Multiple canvases:
    Require selection; do not choose the last focused one silently.
```

Persisting a canvas choice across reloads is unsafe because canvas IDs are session-lifetime identities.

### 7.4 Action completion is not visual completion

The API distinguishes:

```text
accepted
completed
scenePending
motionPending
```

A returned action receipt does not prove pixels have finished painting.

For a “switch view and summarize it” workflow:

1. Submit the explicit view-selection action.
2. Read the same canvas again.
3. Verify desired view/root/attention.
4. Check scene revisions.
5. Wait with bounded polling if necessary.
6. Report uncertainty if the scene never settles.

______________________________________________________________________

## 8. Proposed Hermes Desktop layouts

Hermes supplies native routes, panes, sidebar rows, settings, status items, commands, and composer extensions. Use those supported surfaces and its own UI kit.[^8_2]

### 8.1 Full-page companion workspace

```text
┌──────────────────────────────────────────────────────────────┐
│ Knowledge                       Vault: Work  [Connected]      │
├──────────────────────────────────────────────────────────────┤
│ Search notes…                                   [Search]     │
├─────────────────────┬────────────────────────────────────────┤
│ Results             │ Selected node                          │
│                     │                                        │
│ Architecture        │ Name / source / identity               │
│ Runtime             │ Coverage / revision                    │
│ Packaging           │                                        │
│                     │ Relationships                          │
│                     │ Parents / children / lateral           │
│                     │                                        │
│                     │ Evidence                               │
├─────────────────────┴────────────────────────────────────────┤
│ [Open in Obsidian] [Preview context] [Insert into chosen chat] │
└──────────────────────────────────────────────────────────────┘
```

This is a search/evidence interface, not a substitute graph renderer.

### 8.2 Docked companion pane

Suggested default: right side, roughly 320–400 px, user-resizable.

```text
┌───────────────────────────────┬──────────────────────┐
│ Hermes conversation           │ K-Plex               │
│                               │                      │
│                               │ Vault / status       │
│                               │ Search               │
│                               │ Selected node        │
│                               │ Evidence summary     │
│                               │ Open original app    │
└───────────────────────────────┴──────────────────────┘
```

Hermes supports semantic placement and explicit docking relative to a workspace pane. It also has a specific close behavior: closing a plugin’s only pane disables that Desktop plugin. Design and test around that rather than assuming a close button merely hides a panel.[^8_2]

### 8.3 Context preview

Before insertion:

```text
Source: K-Plex
Vault: Work
Scope: selected canvas / selected node
Coverage: local, incomplete
Truncated: no

Included:
- selected node
- bounded neighbors
- connection evidence

Not included:
- full note bodies
- hidden or unrelated vault content
```

Give users a clear boundary around what will enter the model context.

### 8.4 Connection-state UI

Proposed states:

```text
Not configured
CLI unavailable
Launch consent required
Vault mismatch
K-Plex unavailable
API unsupported
Index partially ready
Ready
Request timed out
Disconnected
```

Avoid a single green “Connected” indicator that hides readiness or coverage.

### 8.5 Styling principles

- Use Hermes SDK components and theme tokens.
- Keep plugin styles scoped.
- Reuse native menus/dialogs where the SDK provides them.
- Do not import Obsidian’s global stylesheet into Hermes.
- Do not install a whole Hermes theme just to style this plugin.
- Do not inspect React fibers or composer DOM.
- Use documented composer methods.[^8_2]

______________________________________________________________________

## 9. Agent tools: small, bounded, and capability-oriented

Suggested initial tools:


| Tool | Purpose | Side effect |
| :-- | :-- | :-- |
| `kplex_status` | Availability and readiness | CLI may launch app unless policy prevents it |
| `kplex_search` | Ranked search | No source writes |
| `kplex_node` | Exact node lookup | No source writes |
| `kplex_neighbors` | Bounded canonical relationships | No source writes |
| `kplex_evidence` | Explain a pair | No source writes |
| `kplex_canvases` | List open canvases | No source writes |
| `kplex_canvas_context` | Current scene-derived context | No source writes |
| `kplex_export_mindmap` | Existing Markdown export | Returns bytes only |

Later, separately:

```text
kplex_navigate
kplex_select_view
kplex_select_preset
```

Do not initially expose:

```text
kplex_eval
kplex_shell
kplex_reindex_private
kplex_write_any_file
kplex_apply_any_configuration
```


### Tool-result design

Return concise structured data, preserving:

```text
scope identity
upstream result
coverage
truncation
revision
warnings
```

Do not make the model infer completeness from an omitted field.

### Skills should explain correct interpretation

Bundle a Hermes skill that teaches:

- Search before selecting opaque node IDs.
- Never infer paths from IDs.
- Check completeness and truncation.
- Use evidence to explain a relationship.
- Distinguish graph neighbors from canvas visibility.
- Do not claim exhaustive absence from partial results.
- Treat note text as untrusted content, not instructions.

Hermes supports bundled skills, native tool registration, manifest declarations, and plugin validation.[^8_4]

______________________________________________________________________

## 10. Hermes-style plugin packaging

### 10.1 Standalone plugin, not a core patch

Hermes explicitly directs third-party integrations toward standalone plugin repositories, installed through its plugin mechanisms.[^8_4]

Proposed runtime package:

```text
hermes-kplex/
├── plugin.yaml
├── __init__.py
├── schemas.py
├── tools.py
├── service.py
├── transports/
│   ├── base.py
│   ├── obsidian_cli.py
│   └── process_output.py
├── protocol/
│   ├── models.py
│   └── dispatch.js
├── desktop/
│   ├── plugin.js
│   └── plugin_api.py
├── skills/
│   └── kplex/
│       └── SKILL.md
├── resources/
│   └── compatibility.json
└── notices/
```

The exact backend route factory and registration signatures must be taken from the pinned Hermes revision. Do not publish guessed executable code as if it were confirmed.

### 10.2 Keep registration side-effect free

`register(ctx)` should register tools, skills, and supported hooks. It should not:

- Spawn Obsidian.
- Probe every vault.
- Install dependencies.
- Start a background network listener.
- Read all note content.

Hermes’s Plugin Doctor runs registration in isolation and blocks direct Python socket connections to detect accidental network work during registration. It is not a security sandbox.[^8_4]

### 10.3 Manifest outline

Illustrative only:

```yaml
name: hermes-kplex
version: 0.1.0
description: K-Plex companion for the original Obsidian application

provides_tools:
  - kplex_status
  - kplex_search
  - kplex_node
  - kplex_neighbors
  - kplex_evidence
  - kplex_canvases
  - kplex_canvas_context
  - kplex_export_mindmap
```

Add configuration fields only after validating the selected manifest schema.

Do not invent a global Hermes API version requirement. Current documentation contains both a behavior-based compatibility discussion and manifest-generation fields; use the actual pinned loader and validators as the implementation authority.[^8_4]

### 10.4 Store state outside replaceable code

Use Hermes’s plugin storage helpers for durable profile-scoped data.[^8_4]

Separate:


| Data | Location/policy |
| :-- | :-- |
| Executable path and selected vault | User configuration |
| CLI discovery cache | Disposable cache |
| Canvas IDs | Ephemeral |
| Graph results | Short-lived, revision-scoped |
| Diagnostics | Redacted |
| Pairing secrets for a future bridge | Secret storage, not Desktop localStorage |
| Vendor source | Development repository |


______________________________________________________________________

## 11. Repository layout and submodule policy

Proposed development repository:

```text
HermesKPlex/
├── vendor/
│   └── kplex/                     # Actual pinned submodule
├── plugin.yaml
├── __init__.py
├── schemas.py
├── tools.py
├── service.py
├── transports/
├── protocol/
├── desktop/
│   └── plugin_api.py
├── frontend/
│   ├── src/
│   ├── tsconfig.json
│   └── build.mjs
├── skills/
├── tests/
│   ├── python/
│   ├── protocol/
│   ├── desktop/
│   ├── native/
│   └── fixtures/
├── scripts/
│   ├── verify-vendor.mjs
│   ├── package-release.mjs
│   └── verify-archive.mjs
├── docs/
│   ├── architecture/
│   ├── setup/
│   ├── compatibility/
│   └── validation/
├── .github/
│   └── workflows/
├── package.json
├── package-lock.json
├── pyproject.toml
├── NOTICE
└── THIRD_PARTY_NOTICES.md
```


### Submodule rules

- Track `HermesKPlex/kplex-Vendor`.
- Pin an exact reviewed commit.
- Never resolve “latest” during release.
- Keep upstream history.
- Record the upstream base and fork delta.
- Make vendor updates separate reviewable changes.
- Preserve the upstream plugin ID unless intentionally creating an independently installable variant.

A `.gitmodules` file alone is not a submodule. The repository must include the Git link entry.

### Avoid accidental duplicate installations

The inspected vendor manifest uses plugin ID `k-plex`.

Therefore, a fork build with the same ID should be treated as a replacement installation of that plugin, not a second plugin that can safely coexist with the original in the same vault.

Installation tooling must:

- Detect existing installation.
- Display the target path.
- Preserve user data.
- Require confirmation before replacement.
- Avoid silently renaming IDs.

______________________________________________________________________

## 12. Build systems and compilation targets

### 12.1 K-Plex build: retain upstream behavior

The inspected build uses:

- TypeScript checking.
- esbuild.
- Entry point `src/main.ts`.
- CommonJS output.
- ES2021 target.
- External Obsidian/Electron/CodeMirror/Node modules.
- `dist/main.js`.
- Copied `manifest.json` and `styles.css`.
- Production React script-element hardening.

The vendor’s package declares Node:

```text
>=22.22.2 <23
```

Its verification workflow uses Node `22.22.2`.

Use the vendor’s own build rather than recreating that configuration in the parent repository.

### 12.2 Semantic compilation versus source-code compilation

Document these separately:

```text
Build-time:
TypeScript → JavaScript plugin artifact

Runtime:
Normalized source facts → semantic graph/evidence
```

Neither is something the Hermes Python wrapper should reimplement.

### 12.3 Hermes frontend build

If we author TypeScript/TSX:

```text
frontend/src/plugin.tsx
           ↓
TypeScript checking + esbuild
           ↓
desktop/plugin.js
```

Requirements:

- ESM default export.
- Bundle internal implementation dependencies.
- Externalize only permitted host imports.
- Do not bundle another React instance.
- No raw JSX in the final disk plugin.
- No app-internal imports.

Hermes disk plugins load uncompiled JavaScript and restrict imports to the SDK, React, and JSX runtime. “No build required” means plain-JavaScript authors can skip compilation—not that our TSX source will execute directly.[^8_2]

### 12.4 Python packaging

Prefer a small dependency set.

Potentially:

- Standard library for subprocess, JSON, paths, and async orchestration.
- An existing validated HTTP framework only if required by Hermes’s backend surface.
- No graph library.
- No vector database.
- No independent Markdown compiler.
- No Node runtime required merely to call the CLI.

Hermes supports dependency declarations through plugin metadata or an adjacent `pyproject.toml`; dependency admission is managed by the host.[^8_4]

______________________________________________________________________

## 13. Versioning and compatibility

Track separate version axes:


| Axis | Example |
| :-- | :-- |
| HermesKPlex release | Integration’s own version |
| K-Plex plugin version | Manifest value |
| K-Plex fork commit | Exact source dependency |
| K-Plex public API | Version plus advertised operations |
| Hermes release/commit | Tested host |
| Obsidian app version | Tested runtime |
| Obsidian installer/CLI version | CLI support |
| Protocol schema | Our transport envelope |

Do not equate them.

The official CLI requires an appropriate installer version; merely updating application code may not establish that the CLI is registered and usable.[^8_1]

### Capability-based compatibility

An older provider with fewer operations should still support the operations it advertises.

For example:

```text
Search available, Markdown export absent:
    Search works.
    Export control is disabled with explanation.
```

Do not make a missing optional export capability disable the whole integration.

### Schema generation

Proposed approach:

1. Keep our wire schemas in one canonical definition.
2. Generate JSON Schema and Python/TypeScript models where useful.
3. Check generated outputs in CI.
4. Verify fixtures against both languages.
5. Keep upstream API DTOs distinct from our transport metadata.

Do not generate a second implementation of semantic rules.

______________________________________________________________________

## 14. Security model

### 14.1 This is trusted local automation

Obsidian CLI `eval` has broad application privileges. Our wrapper’s allowlist reduces accidental exposure; it does not sandbox the underlying process.

Likewise, Hermes plugin capabilities are host consent gates, not a general sandbox.[^8_4]

### 14.2 Threats to address

| Threat | Mitigation |
| :-- | :-- |
| Model supplies arbitrary JS | No code parameter; fixed dispatcher |
| Shell injection | Argument arrays; no shell |
| Wrong vault | Explicit configured target and verified identity |
| Malicious executable path | User-controlled configuration; resolve and display path |
| Huge output | Bounded streaming capture |
| UI switches scope during request | Request-generation and scope checks |
| Plugin reload | Reacquire API; invalidate handles/canvas IDs |
| Sensitive logs | Redaction; no note bodies or eval payloads by default |
| Prompt injection in notes | Treat retrieved material as data |
| Destructive retries | No blind replay of actions |
| Remote endpoint confusion | Local-only baseline; no implicit relay |

### 14.3 Scope identity

Proposed request identity:

```text
connection generation
profile identity
configured vault identity
transport instance
request ID
upstream revision when applicable
```

Upstream numeric revisions are session-local observations. Do not treat them as durable global sequence numbers.

### 14.4 Context minimization

Default:

- Metadata and evidence.
- Bounded neighborhoods.
- Scene-derived context.
- No whole-vault export.
- No note bodies unless a later explicit capability is added.

The public API’s scene context and Mind Map export already provide a useful route that does not require returning note bodies.

______________________________________________________________________

## 15. Testing strategy

### Layer 1 — Pure unit tests

Test:

- Argument validation.
- Fixed-operation dispatch.
- JS serialization.
- Vault targeting.
- Output framing.
- Error mapping.
- Limits.
- Stale-generation rejection.


### Layer 2 — Cross-language contract tests

Use fixtures for:

```text
success
upstream failure
transport failure
partial coverage
truncated results
missing optional capability
stale revision
ambiguous canvas
unavailable API
```


### Layer 3 — Subprocess adapter tests

Use a fake CLI executable to simulate:

- Zero exit with error text.
- Nonzero exit.
- Delayed output.
- Oversized output.
- Malformed JSON.
- Prefix/banner variations.
- Spawn failure.
- Cancellation.

These are adapter tests, not native Obsidian acceptance.

### Layer 4 — Vendor verification

The inspected vendor package’s `verify` includes:

```text
check:release
check:architecture
check:core
lint:obsidian
test
build
```

Its `npm test` includes browser-dependent suites and source-repository tests. Its verification workflow supplies a Chrome path.

Preserve these checks when modifying the fork. Do not claim a thin mocked host test substitutes for them.

### Layer 5 — Real Obsidian acceptance

In a disposable synthetic vault:

1. Confirm app/installer/CLI versions.
2. Confirm exact vault.
3. Confirm K-Plex version and API capabilities.
4. Query known fixture relationships.
5. Inspect evidence.
6. Open multiple canvases.
7. Test explicit canvas selection.
8. Reload K-Plex.
9. Verify old handles/canvas references are rejected.
10. Test source rename/delete and pending indexing.
11. Confirm no file changes from read-only operations.

The vendor contains native public-API and API-lifetime runners worth studying and reusing where their assumptions fit.

### Layer 6 — Real Hermes acceptance

Test:

- Plugin discovery.
- Manifest/tool agreement.
- Enable/disable.
- Desktop hot reload.
- Profile/gateway switch.
- Explicit composer targeting.
- No accidental chat submission.
- UI disposal.
- CLI unavailable states.

Hermes provides `hermes plugins doctor` and `hermes plugins validate`; use both, plus functional tests.[^8_4]

______________________________________________________________________

## 16. GitHub Actions: proposed pipeline

Do not simply copy the vendor workflows into the parent repository.

The vendor verification workflow already runs on Ubuntu with Node 22.22.2, npm cache, Chrome, and `npm run verify`. Its release workflow builds, validates release metadata, attests assets, and publishes the three Obsidian plugin files.

Our parent repository needs different jobs.

### 16.1 Pull-request pipeline

```text
validate
├── repository / submodule policy
├── Python lint / types / tests
├── TypeScript checks / frontend tests
├── protocol generation / fixtures
├── artifact build / inspection
├── Hermes doctor / validation
└── vendor checks when relevant
```

Recommended jobs:


| Job | Purpose |
| :-- | :-- |
| `policy` | Check submodule URL, exact pin, no committed secrets/binaries |
| `python` | Lint, type checks, unit tests |
| `frontend` | Type checks, UI tests, ESM build |
| `contracts` | JSON Schema and cross-language fixtures |
| `package` | Build and inspect release archive |
| `hermes-compat` | Isolated plugin validation against pinned host |
| `vendor-verify` | Run vendor verification when pin or fork-sensitive code changes |
| `security` | Workflow analysis and dependency checks |

### 16.2 Keep parent and vendor Git histories separate

The vendor workflow uses `VALIDATION_BASE`.

When running it from a parent repository, do not pass the parent PR’s base SHA into the submodule’s history. Those commits belong to different repositories.

Compute the vendor’s own prior/new revisions or invoke its checks with a valid vendor-specific baseline.

This is easy to get wrong and can invalidate change-sensitive checks.

### 16.3 Cross-platform matrix

Run adapter tests on:

```text
Linux
Windows
macOS
```

Separate them from native app tests.

A test using a fake CLI on Windows is not proof that `Obsidian.com` works with the actual installed app.

### 16.4 Native application acceptance workflow

Use a manually triggered or tightly controlled workflow for real-app tests.

Requirements:

- Disposable vault.
- No personal notes.
- Documented installation/acquisition permissions.
- Exact build hashes.
- Redacted logs.
- Explicit tested platform.
- No public PR execution on a developer’s personal machine.

GitHub warns that self-hosted runners can be persistently compromised by untrusted workflow code, especially in public repositories. Avoid exposing an interactive Obsidian test machine to arbitrary fork PRs.[^8_5]

### 16.5 Release pipeline

```text
Protected release trigger
    ↓
Validate tag/version
    ↓
Checkout exact parent and submodule commits
    ↓
Install locked build dependencies
    ↓
Run required tests
    ↓
Build packages
    ↓
Inspect archive contents
    ↓
Generate checksums / source manifest / notices
    ↓
Attest artifacts
    ↓
Publish reviewed release
```

Do not rebuild a different artifact after approval without revalidating it.

### 16.6 Permissions and supply-chain controls

Recommendations:

- Default workflow token: `contents: read`.
- Grant release permissions only to the release job.
- Use protected environments for publishing.
- Avoid privileged PR triggers with untrusted checkout.
- Pin/restrict workflow dependencies.
- Review workflow changes through CODEOWNERS.
- Never expose release secrets to PR builds.
- Treat downloaded artifacts as untrusted until verified.

GitHub’s guidance covers least privilege, untrusted checkout risks, action dependency locking or full commit pinning, and environment approvals.[^8_5]

______________________________________________________________________

## 17. Release artifacts and installation

### 17.1 Separate artifacts

Recommended:


| Artifact | Contents |
| :-- | :-- |
| Hermes plugin package | Python/backend, Desktop ESM, skill, resources |
| K-Plex vendor build | Optional separately identified Obsidian plugin assets |
| Source bundle | Integration source and exact vendor source |
| Checksums/provenance | Artifact hashes and build identity |
| Notices/SBOM | Applicable licenses and dependency inventory |

Do not include original Obsidian binaries.

### 17.2 Avoid unbuilt source installation

A Git install of the development repository may not contain the generated `desktop/plugin.js`.

Pick and document one distribution strategy:

- A release-ready branch containing generated installable assets.
- A supported package distribution path.
- An explicit build/install procedure.

Do not assume Hermes’s installer will compile our frontend automatically.

### 17.3 Installation should not modify a vault silently

Proposed sequence:

1. Install Hermes plugin disabled.
2. User enables it.
3. User configures CLI path/vault.
4. Run explicit diagnostics.
5. Confirm compatible K-Plex installation.
6. Offer instructions or an explicitly approved vendor installation.
7. Verify capabilities.
8. Enable read tools.

Do not edit the vault’s enabled-plugin configuration behind the user’s back.

### 17.4 Source archive completeness

The source release should contain the exact vendor source required to reproduce shipped artifacts, rather than relying only on a submodule URL and a branch name.

Record:

```text
parent commit
vendor commit
upstream base
Node version
Python version
lockfile hashes
artifact hashes
build commands
test results
```


______________________________________________________________________

## 18. Licensing and attribution plan

The current vendor package declares `AGPL-3.0-only`.

Proposed conservative project policy:

- Preserve upstream notices.
- Keep the upstream license text intact.
- Attribute Zsolt Viczian and contributors in project documentation and About/settings.
- Record modified fork files and revisions.
- Include corresponding source/build materials as required.
- Do not claim that “plugin” or “separate process” automatically removes obligations.

The exact licensing of original integration code should be selected deliberately after reviewing how it incorporates, links to, or merely automates upstream code. This plan is not a legal determination.

Suggested notice:

```text
HermesKPlex integrates K-Plex (Knowledge Plex),
created by Zsolt Viczian and contributors.

K-Plex is used through its original Obsidian runtime and public API.
Where vendor code is distributed, its applicable license and copyright
notices are preserved.

Vendor source and exact revision are identified in the release manifest.
```

Do not retain the earlier “MIT versus AGPL” warning as a current confirmed conflict without rechecking the exact pinned revision.

______________________________________________________________________

## 19. Proposed implementation stages

### Stage 0 — Baseline and architecture records

Deliver:

- Exact vendor/Hermes commits.
- Current license inventory.
- Compatibility matrix template.
- Read-only scope definition.
- No repository or runtime claims beyond measured evidence.


### Stage 1 — CLI proof of concept

Deliver:

- Explicit vault target.
- Capabilities/status.
- Search.
- Neighbors.
- Evidence.
- Bounded output/error parsing.

Acceptance:

> A synthetic fixture vault can be queried through the original K-Plex API without modifying K-Plex or installing a new bridge.

### Stage 2 — Hermes native tools

Deliver:

- Side-effect-free registration.
- Tool schemas.
- Operation allowlist.
- Profile-scoped configuration.
- Bundled usage skill.
- Doctor/validator checks.


### Stage 3 — Desktop companion

Deliver:

- Native status/settings.
- Search/evidence page.
- Optional docked pane.
- Canvas selector.
- Context preview.
- Explicit composer insertion.


### Stage 4 — Canvas workflows

Deliver:

- Scene context.
- Mind Map Markdown export.
- Explicit navigation/view/preset actions.
- Accepted-versus-settled handling.


### Stage 5 — Hardening and release

Deliver:

- Cross-platform adapter tests.
- Real-app acceptance evidence.
- Packaging validation.
- Notices/source bundle.
- Provenance.
- Installation/upgrade/rollback documentation.


### Stage 6 — Optional persistent bridge

Only after measured need.

Do not treat it as mandatory just because it appeared in the earlier proposal.

______________________________________________________________________

## 20. Architecture decisions I would adopt now

| Decision | Proposed outcome |
| :-- | :-- |
| Original host | User-installed original Obsidian |
| K-Plex implementation | Existing vendor fork, minimal delta |
| Initial transport | Official Obsidian CLI |
| Graph/compiler duplication | None |
| Hermes integration | Standalone unified plugin |
| Desktop experience | Native companion controls, not fake embedded Obsidian |
| Initial permissions | Read-only knowledge operations |
| Navigation | Separate explicit action |
| Note writes | Deferred |
| Network listener | Not required initially |
| Remote gateway | Explicitly unsupported until designed |
| Submodule | Exact pinned fork commit |
| CI | Locked, scoped, reproducible, separated from native acceptance |
| Distribution | Separate plugin artifacts and complete source materials |
| Repository writes from this conversation | None |

The most important source-level discovery is that K-Plex already supplies a documented, versioned API and an official-CLI usage guide. That removes the need to extract its compiler, rebuild its renderer, or create a network server before the first useful Hermes integration. The first engineering milestone should therefore be a bounded CLI adapter validated against the actual original application—not another architecture layer.

<div align="center">⁂</div>

[^8_1]: https://help.obsidian.md/cli

[^8_2]: https://hermes-agent.nousresearch.com/docs/developer-guide/desktop-plugin-sdk

[^8_3]: https://github.com/zsviczian/kplex/blob/main/docs/ARCHITECTURE.md

[^8_4]: https://hermes-agent.nousresearch.com/docs/developer-guide/plugins

[^8_5]: https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions


---

# @GitHub research more

The additional source review strengthens the CLI-first approach, but exposes several implementation traps that should change the plan before coding: upstream validation is revision-specific, Windows has special transport behavior, ordinary graph reads do not provide universal pagination, and Hermes’s application-declaration machinery is not automatically wired to arbitrary plugin manifests.

I used GitHub read-only. No repository changes were made.

The most useful next step is a small **compatibility** layer around the existing K-Plex public API—not another graph engine, and not yet a new Obsidian bridge.

## 1. Newly inspected source

This round focused on implementation details and recorded validation rather than repeating the architecture overview.


| Source | Why it matters |
| :-- | :-- |
| Vendor `scripts/testing/obsidian/processTools.mjs` | Actual CLI discovery, shell-free execution, Windows long-command handling |
| Vendor `docs/validation/public-api-cli-240-2026-10-30.md` | Historical native-test evidence and its limits |
| Vendor `docs/validation/public-api-cli-240-refresh-2026-10-31.md` | Later focused validation; explicitly not a full exact-head suite |
| Vendor `skill.md` | Existing agent instructions we should adapt rather than reinvent |
| Vendor `src/adapters/obsidian/publicApiWindow.ts` | Actual public API publication and cleanup |
| Hermes `plugins/application-declarations` documentation | Application discovery and availability gates |
| Official Obsidian CLI documentation | Current host prerequisites, targeting, and command surface |
| Hermes Desktop SDK documentation | Supported UI, persistence, lifecycle, and session targeting |

The GitHub connector returned source text and file blob hashes, but not citation IDs for those individual reads. I identify those findings by exact path below; linked citations refer to the separately retrieved public documentation.

## 2. Upstream has strong evidence—but it is not our acceptance result

The vendor’s validation documents are unusually useful because they distinguish different test checkpoints.

### Historical full verification

The October 30 validation record reports a full run on:

```text
Windows
Node 22.22.2
Obsidian 1.14.5
Installer 1.14.5
```

It records:


| Lane | Reported passing tests |
| :-- | --: |
| Architecture/numeric controls | 9 |
| Core | 130 |
| Main regressions | 1,749 |
| Browser | 425 |
| Sources | 357 |
| IndexedDB/browser persistence | 855 |

It also reports native tests covering multiple canvases, popouts, API lifetime, bounded reads, configuration operations, and Mind Map export.

These are upstream-authored records, not tests I ran.

### Later refresh validation was deliberately narrower

The October 31 refresh record says:

- The API branch was refreshed against another upstream checkpoint.
- Focused tests, production build, native public-API tests, and lifetime tests passed.
- Expensive aggregate indexing suites were intentionally omitted.
- The earlier full verification is historical evidence, not an exact-head full-suite claim.

That distinction needs to survive into our compatibility documentation.

### Proposed evidence model

For every release, record four separate things:

```text
1. Upstream historical evidence
2. Vendor revision verification
3. HermesKPlex adapter verification
4. End-to-end native acceptance
```

Do not label the integration “tested on Obsidian 1.13+” merely because the plugin manifest has that minimum. The native evidence I inspected specifically leaves minimum-host acceptance and other platforms as separate work.

## 3. Windows transport is more complicated than a normal subprocess call

The actual `processTools.mjs` contains a useful implementation lesson.

### Short commands

For short, single-line Windows evaluations, upstream passes the original argument array directly to the CLI.

### Long or multiline evaluations

The test helper calculates a conservative command-length estimate. When the Windows evaluation is long or multiline, it:

1. Creates an owned temporary directory.
2. Writes a local JavaScript file.
3. Passes a short evaluation that reads that file.
4. Removes the temporary directory in `finally`.

It uses no shell.

This is explicitly test tooling, not a production transport component.

### What we should reuse—and what we should not

Reuse the lessons:

- Argument arrays.
- Explicit vault selection.
- Console launcher on Windows.
- Bounded command length.
- Owned temporary resources.
- Guaranteed cleanup.
- No PowerShell/CMD string interpolation.

Do not copy the entire test helper into production without analysis.

The temporary-script path introduces:

- Filesystem permissions and privacy considerations.
- Cleanup after abrupt termination.
- Lifetime races if the app still accesses the script after a timeout.
- Dependence on the execution environment exposing Node file access.
- More privileged machinery than a short read-only request actually needs.


### Recommended production baseline

Keep generated evaluations short and single-line.

For the initial operations—status, search, neighbors, evidence, and canvas reads—requests should usually fit a conservative command budget without temporary JavaScript files.

If a request exceeds that budget:

```text
Reject with REQUEST_TOO_LARGE
```

Do not silently switch to an arbitrary-script execution pathway.

The official CLI confirms that Windows requires the `Obsidian.com` redirector for proper terminal I/O; macOS and Linux have different CLI registration mechanisms.[^9_1]

## 4. Executable discovery needs stricter production rules

The upstream test helper discovers:

- An explicit environment override.
- Common Windows installation locations.
- macOS app-contained executables.
- A PATH fallback.

That is reasonable for developer test tooling, but our user-facing plugin should distinguish:

```text
App is installed
CLI executable exists
CLI is registered
CLI can reach the intended vault
K-Plex API is available
Required capabilities exist
```

These are not equivalent.

### Proposed discovery sequence

```text
Configured executable
    ↓
Validate path and report identity
    ↓
Otherwise inspect supported installation locations
    ↓
Otherwise inspect PATH
    ↓
Ask user to confirm the selected executable
    ↓
Run explicit connection test
```

Avoid an automatic fallback from a missing CLI binary to a GUI executable unless that exact platform/version combination has passed acceptance.

### Installer version and app version are separate

The official documentation requires CLI-capable installer versions and instructs users to enable and register the command-line interface. An application update alone is not the complete prerequisite.[^9_1]

Proposed diagnostics:

```text
Application version: observed / unknown
Installer/CLI support: verified / unverified
Executable: resolved path
Target vault: explicit
K-Plex: available / missing
API version: supported / unsupported
Capabilities: observed list
```

Never present guessed values as observed facts.

## 5. Hermes already has application discovery—but not a magic manifest switch

The newly inspected application-declarations documentation is directly relevant, with an important limitation.

Hermes has reusable machinery for locating desktop applications through:

- Executable paths.
- PATH commands.
- Windows uninstall registry entries.
- macOS application bundles.
- Flatpak exports.
- Snap locations.
- Version inspection where supported.[^9_2]

However, the documentation explicitly says:

> Loader integration is separate work; core does not read plugin YAML automatically.

It also says the declaration registry is process-wide, not profile-scoped, and its existing tool gating is described for MCP servers.[^9_2]

### Implication for HermesKPlex

We should not add this and assume it works:

```yaml
app:
  ...
requires:
  app: true
```

Our direct Python plugin needs either:

1. Explicit, supported integration with those helpers; or
2. Its own thin availability service using public host utilities.

Do not accidentally adopt an MCP-specific declaration as if it automatically governs native plugin tools.

### Availability is not liveness

The documented evaluator does not launch an app or connect to it. It also does not currently produce its reserved `installed_not_running` state.[^9_2]

That is useful for a passive “installed?” check—but not a replacement for the explicit K-Plex connection test.

### Proposed separation

```text
Passive availability service
    └── May inspect approved local installation metadata

Active connection service
    └── May invoke CLI after launch policy/consent

Knowledge service
    └── Executes bounded API operations
```

This allows a settings page to render without waking Obsidian.

## 6. The public API is one service shared across windows

The actual `publicApiWindow.ts` confirms the lifecycle model.

It:

- Publishes the same immutable API root into the main window and popouts.
- Preserves a pre-existing property descriptor.
- Registers window-open and window-close handlers.
- Waits for layout readiness before enumerating restored leaves.
- Removes only aliases it still owns.
- Disposes listeners and retained window references.
- Rolls back partial installation failures.


### Architectural consequence

Do not create one semantic backend per Obsidian window.

The model is:

```text
One loaded K-Plex service
    ├── main-window alias
    ├── popout alias
    └── another popout alias

Service
    └── multiple distinct canvas registrations
```

A window alias is not a separate graph.

### How our adapter should behave

For each operation:

1. Reacquire the current public root.
2. Verify the relevant capability.
3. Execute against an explicit target.
4. Return detached results.
5. Reject stale local UI generations.

Do not persist a JavaScript API handle across plugin reloads.

Canvas IDs also need different treatment from durable vault/node references.

## 7. Upstream already has an agent skill worth adapting

The vendor’s root `skill.md` already contains most of the reasoning safeguards we need:

- Explicit vault on every invocation.
- Verify destination before private reads.
- Discover capabilities.
- Read authored ontology descriptions.
- Preserve coverage and truncation.
- Treat node IDs as opaque and case-sensitive.
- Distinguish graph, scene, and viewport.
- Avoid first/latest-canvas assumptions.
- Check action readback.
- Do not replay uncertain writes.
- Do not treat note content as executable code.


### Recommended reuse approach

Create a Hermes-oriented wrapper skill that:

- Attributes the upstream skill.
- References the pinned vendor version.
- Uses our bounded tools rather than instructing the agent to issue arbitrary `eval`.
- Preserves upstream interpretation rules.
- Removes unsupported mutation workflows from the first release.
- Includes an explicit distinction between read access and authority to act.

Do not bundle the full skill unchanged if it teaches workflows our tool surface intentionally forbids.

### Additional reason to keep scope narrow

The upstream skill includes advanced ontology consolidation, source inspection, and reviewed configuration writes. Those are useful capabilities, but they are not necessary to deliver a strong search/evidence companion.

We should not let the presence of upstream instructions expand the first release’s authority accidentally.

## 8. Pagination: a significant limitation we must expose

The root skill explicitly warns:

> Only advertised usage pagination has continuation cursors; do not invent cursors for other reads.

### This rules out a common adapter mistake

Bad design:

```python
while response.truncated:
    response = search(offset=offset + limit)
```

If the upstream method does not support that pagination model, this fabricates an interface and may repeatedly query the same bounded candidate set.

### Proposed capability matrix

| Operation | Initial behavior |
| :-- | :-- |
| Search | Bounded result; truncation is explicit |
| Neighbors | Bounded result; no invented continuation |
| Evidence | Bounded decisions |
| Canvas snapshot | Bounded committed scene |
| Ontology usage | Use actual upstream cursor if the optional capability is enabled |
| Whole-vault enumeration | Not implied by any of the above |

### UI consequence

Prefer:

```text
Results are limited by the provider.
Narrow the query to inspect a smaller scope.
```

Not:

```text
Load more
```

unless the method genuinely supports it.

## 9. Search, relationship, and visual context are different retrieval products

The source review suggests we should expose three clearly named modes in Hermes.

### A. Search knowledge

Purpose:

```text
Find indexed candidate nodes.
```

This is not a full-text note-body search unless the API explicitly supplies that behavior.

The official Obsidian CLI has separate full-text search commands, but those should not be silently substituted for K-Plex search: they have different semantics and data exposure.[^9_1]

### B. Inspect semantic neighborhood

Purpose:

```text
Read canonical direct relationships and evidence.
```

Not:

```text
Read everything shown on screen.
```


### C. Use current map context

Purpose:

```text
Capture the selected canvas's admitted scene.
```

Not:

```text
Export the whole graph or note bodies.
```


### Proposed Desktop tabs

```text
Knowledge
├── Search
├── Relationships
└── Open maps
```

This is more accurate than a single ambiguous “Graph” tab.

## 10. Export behavior needs stricter targeting

The upstream skill documents:

- Mind Map Markdown export uses the committed scene.
- A subtree is selected by occurrence ID, not file path.
- Repeated occurrences remain distinct.
- Filters and folds constrain output.
- Offscreen constructed rows may still export.
- The API returns bytes without clipboard or file-save effects.
- Native PNG support does not mean API PNG support.


### Proposed export workflow

```text
Select canvas
    ↓
Read snapshot
    ↓
Check view/capability/pending state
    ↓
Select whole map or exact occurrence
    ↓
Export using expected revision
    ↓
Show warnings and scope
    ↓
Explicitly insert or save
```

Do not automatically switch a Compass canvas to Mind Map merely because the user clicked “Export.” That would be an additional action and can change the user’s workspace.

### Native-only image export

For the baseline, show:

```text
Image export is available in the original app
```

when that is the supported route.

Do not introduce screenshots or DOM capture as an undocumented fallback.

## 11. A refined internal module design

The previous large plan can now be simplified.

```text
Hermes tool handlers / Desktop backend routes
                    │
                    ▼
             KnowledgeService
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
      Policy     Scope     Capability
      checks     checks    checks
                    │
                    ▼
             CliTransport
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
      Program    Process    Output
      builder    runner     decoder
                    │
                    ▼
        Existing K-Plex public API
```


### Proposed responsibilities

| Module | Owns | Must not own |
| :-- | :-- | :-- |
| Availability | Local install discovery | Graph queries |
| Scope resolver | Configured vault/connection identity | Guessing active vault |
| Capability reader | Supported operations and limits | Semantic classification |
| Program builder | Fixed safe evaluation templates | Arbitrary agent code |
| Process runner | Deadlines/output caps/cancellation | UI state |
| Output decoder | Framing and schemas | Guessing missing results |
| Knowledge service | Policy and orchestration | Second graph database |
| Context formatter | Bounded agent-readable rendering | Inventing relationships |
| Desktop client | View state and presentation | Subprocess execution |

The Desktop should reach the backend through Hermes’s scoped namespace, not acquire local process privileges itself. The SDK supports that split directly.[^9_3]

## 12. Cache design: cache observations, not a parallel graph

A tempting optimization is to persist every returned node and edge in SQLite. I would avoid that initially.

### Safe initial caching

Cache:

- Resolved executable.
- Passive discovery result.
- Recent capability snapshot with short expiry.
- Last result for a mounted view.
- Request deduplication while work is in flight.

Invalidate on:

- Profile/connection generation change.
- Vault change.
- API unavailability/reload indication.
- Relevant revision change.
- Plugin disable.
- Explicit refresh.


### Do not initially persist

- A whole second graph.
- Canvas IDs as durable references.
- Session-local revision numbers as globally meaningful.
- All note/evidence text by default.

A small transport cache can reduce repeated subprocess work without becoming another source of semantic truth.

## 13. Request scheduling and UI lifecycle

The CLI path should be treated as a limited resource.

### Proposed scheduling policy

- Serialize per configured vault initially.
- Coalesce identical status requests.
- Debounce search.
- Drop obsolete search results.
- Stop polling hidden/unmounted views.
- Back off when the provider is unavailable.
- Give explicit user actions priority over background status refresh.
- Never let a polling loop hold the queue indefinitely.

The upstream test helper’s full verification timeout is long because its serial test suite contains expensive cases. That timeout is not an appropriate default for interactive requests.

### Cancellation semantics

Distinguish:

```text
Cancel waiting in our queue
Cancel our subprocess
Discard obsolete response
Cancel work inside Obsidian
```

These are different operations.

Killing the CLI process does not necessarily undo work already accepted by the running application.

For passive reads, discarding a stale response is generally sufficient for UI correctness. For actions, readback is required before retrying.

## 14. Better diagnostics without exposing private data

Proposed diagnostic record:

```json
{
  "requestId": "...",
  "operation": "evidence",
  "scopeGeneration": 7,
  "transport": "obsidian-cli",
  "durationMs": 123,
  "processExit": 0,
  "result": "success",
  "coverageScope": "local",
  "truncated": false
}
```

Do not log by default:

- Query text.
- Note titles.
- File paths.
- Evidence raw values.
- Generated evaluation source.
- Full stdout.
- Full vault path.

Support an explicit diagnostic export with a preview and redaction.

In the UI, explain where a failure occurred:

```text
Discovery failed
Process failed
Output could not be decoded
K-Plex API unavailable
Operation unsupported
Index incomplete
Canvas expired
```

“Connection failed” is too broad to troubleshoot.

## 15. Testing changes prompted by this research

Add these cases to the proposed test plan.

### CLI and encoding

- Windows redirector path.
- Spaces and non-ASCII executable/vault names.
- Quote, backslash, newline, Unicode, and control-character inputs.
- Conservative command-length rejection.
- Zero exit with error payload.
- Output cap reached mid-stream.
- Late process output after cancellation.
- API failure serialized as valid JSON.


### Scope and lifetime

- Two vaults with the same display name if the host permits that configuration.
- Explicit vault ID versus display name.
- K-Plex unload/reload between discovery and execution.
- Popout migration.
- Canvas closes between preview and export.
- Another Hermes profile becomes active during a request.


### Completeness and reasoning

- Partial empty search.
- Truncated neighbors with no continuation.
- Node exists but has incomplete relationship coverage.
- Decorative connection is not returned as evidence.
- Offscreen occurrence remains in constructed-map export.
- Repeated node occurrences are not deduplicated incorrectly.


### Application availability

- Installed app but CLI unregistered.
- Registered CLI but incompatible installer.
- Original app closed and launch disallowed.
- Missing optional API method.
- No open canvas.
- Multiple eligible canvases.


## 16. CI should distinguish evidence classes

The upstream records demonstrate why a single green badge is insufficient.

Recommended published matrix:


| Evidence class | What it establishes |
| :-- | :-- |
| Unit tests | Our validation and formatting logic |
| Fake-process tests | Transport handling |
| Vendor portable/browser tests | Upstream implementation behavior |
| Real Obsidian CLI tests | Actual local integration |
| Real Hermes Desktop tests | UI and lifecycle integration |
| Cross-platform native acceptance | Specific OS/app/installer combinations |

A macOS fake-process test must not turn the macOS native-support column green.

### Additional workflow checks

Proposed new checks:

- Reject accidental arbitrary-code tool schemas.
- Verify read-only dispatcher allowlist.
- Verify final Desktop bundle imports.
- Detect drift between advertised tools and registered handlers.
- Confirm package excludes vault fixtures containing personal material.
- Confirm vendor commit recorded in release metadata.
- Separate “build succeeded” from “native acceptance completed.”

Hermes’s application-declaration docs also show that not all discovery features automatically integrate with plugin YAML, so CI should test actual registration behavior rather than validate a plausible-looking manifest only.[^9_2]

## 17. The fork may need fewer changes than expected

This is the strongest architectural result from the latest review.

For the first milestone, the ideal vendor diff is:

```text
zero runtime changes
```

The fork still serves useful purposes:

- Exact dependency tracking.
- Reproducible builds.
- Compatibility testing.
- Source provenance.
- A place for narrowly justified fixes.
- Controlled future upstream updates.

But we should not modify it merely to justify having a fork.

Potential later changes must each answer:

```text
Which missing public capability blocks a real user workflow?
Why can the integration not use an existing operation?
Can the change remain additive and lifecycle-owned?
What tests prove no extra source/index work?
Can it be upstreamed?
```


## 18. Updated recommended milestone

Before a polished UI or bridge, define a source-backed acceptance probe:

```text
1. Confirm explicit target vault.
2. Discover K-Plex public API capabilities.
3. Obtain status and authored ontology guidance.
4. Search for a known fixture node.
5. Read its canonical neighbors.
6. Explain one known pair with evidence.
7. Enumerate two canvases without choosing silently.
8. Capture one explicit canvas's context.
9. Export a committed Mind Map with warnings.
10. Reload K-Plex and verify stale references are not reused.
```

The official CLI supplies the host automation surface; the existing K-Plex API supplies the semantic operations. Hermes’s SDK supplies the companion UI and backend routing. None of those requires reproducing Obsidian or K-Plex’s renderer.[^9_3][^9_1]

The outstanding uncertainty is now narrower: not “can these projects be integrated in principle?” but “does the exact pinned vendor build behave reliably through the supported CLI on each target platform, and does our wrapper preserve its scope, lifecycle, and completeness guarantees?”

No builds, native tests, or repository writes were performed in this round.

<div align="center">⁂</div>

[^9_1]: https://help.obsidian.md/cli

[^9_2]: https://hermes-agent.nousresearch.com/docs/developer-guide/plugins/application-declarations

[^9_3]: https://hermes-agent.nousresearch.com/docs/developer-guide/desktop-plugin-sdk


---

# @GitHub deeper integration, research how plugin tool hook modification works, as recently Hermes, implemented a pre_tool_call modification hook, which we can use for an even deeper integration into this

Yes. Hermes’s current `pre_tool_call` hook can return an argument-modification directive, and we can use it for a substantially deeper HermesKPlex integration without patching Hermes core.

However, the source review reveals an even better division of responsibilities:

- `tool_request` middleware for normalizing arguments before policy checks.
- `pre_tool_call` for blocking, requesting approval, and narrowly scoped argument patches.
- `tool_execution` middleware for wrapping execution when genuinely necessary.
- `post_tool_call` for observing the actual outcome.
- `pre_llm_call` for optional, bounded knowledge context.

The key is **composition**: use these together rather than turn one pre-tool hook into a hidden retrieval engine, router, approval system, and graph writer. Hermes documents these as distinct extension surfaces with different ordering and failure behavior.[^10_1][^10_2]

I used GitHub read-only. No repository changes were made.

## 1. What I verified in the current source

I inspected or searched:

```text
hermes_cli/plugins.py
hermes_cli/plugins_dispatch.py
model_tools.py
tests/hermes_cli/test_plugins.py
tools/connectors/dispatch.py
plugins/disk-cleanup/
plugins/security-guidance/
website/docs/developer-guide/middleware.md
website/docs/developer-guide/observer-hooks.md
website/docs/user-guide/features/hooks.md
```

GitHub code search returned matches tied to commit:

```text
9242ce464b4a84fb8b7a81df18400099d2a1a790
```

That is useful as a research checkpoint, but I have not run this revision or established it as the release we should support.

### Confirmed modification contract

A Python plugin registers:

```python
def register(ctx):
    ctx.register_hook("pre_tool_call", on_pre_tool_call)
```

It can return:

```python
return {
    "action": "modify",
    "args": {
        "some_argument": "replacement"
    },
}
```

The returned dictionary is shallow-merged over the tool arguments before execution. Multiple modifications accumulate; if two directives modify the same key, the later one wins.[^10_1]

### Other control directives

The hook can also return a block directive, and the documented precedence is:

```text
Any valid block wins
    ↓
Otherwise the first valid approve
    ↓
Argument modifications are accumulated
```

An `approve` directive requests the human approval flow; it does not mean that the plugin has already authorized execution.[^10_1]

### An important limitation

The documented `modify` shape changes arguments. It does not provide a documented tool-name replacement field.

Therefore:

```text
read_file → kplex_node
```

is not something we should assume `pre_tool_call` can do.

Keep tool routing, argument normalization, context enrichment, and execution wrapping conceptually separate.

## 2. The actual extension pipeline

Hermes documents this tool sequence:

```text
Model emits tool call
        ↓
Parse and coerce arguments
        ↓
tool_request middleware
        ↓
Normal pre-execution path
    ├── availability checks
    ├── pre-tool directives
    ├── guardrails
    └── approvals
        ↓
tool_execution middleware
        ↓
Tool handler
        ↓
post_tool_call
        ↓
transform_tool_result
        ↓
Model receives result
```

The important distinction is that `tool_request` runs before hooks and approval checks. Execution middleware runs later.[^10_2]

### Recommended HermesKPlex mapping

| Integration behavior | Best initial surface |
| :-- | :-- |
| Resolve a selected connection for our tools | `tool_request` plus handler validation |
| Add safe defaults or bounded limits | `tool_request` |
| Reject missing/unauthorized vault scope | Handler and `pre_tool_call` |
| Request approval for an explicit action | `pre_tool_call` |
| Query K-Plex | Our normal tool handler |
| Measure actual outcome | `post_tool_call` |
| Supply optional turn-level graph context | `pre_llm_call` |
| Wrap an existing compatible tool | Optional `tool_execution`, after dedicated testing |
| Rewrite model-bound result text | Usually avoid; use our handler’s formatter |

This gives us deeper integration while keeping the effect of each feature understandable.

## 3. Why middleware is often preferable to pre-tool modification

The two return shapes differ.

### Pre-tool hook: patch

```python
return {
    "action": "modify",
    "args": {"limit": 20},
}
```

This is a shallow argument patch.[^10_1]

### Tool-request middleware: complete replacement

```python
return {
    "args": complete_effective_arguments,
    "source": "hermes-kplex",
    "reason": "resolved explicitly selected connection",
}
```

Middleware should return a complete replacement argument payload. Hermes carries optional source/reason entries forward as `middleware_trace`.[^10_2]

### Proposed policy

For HermesKPlex:

> Perform routine normalization in request middleware. Reserve the pre-tool hook for explicit policy decisions and small compatibility patches.

Benefits:

- Downstream hooks see normalized arguments.
- Approval logic evaluates the intended request.
- Transformations can carry an audit reason.
- We avoid unnecessary late argument changes.
- The normalizer can be idempotent and independently tested.

This is not a reason to avoid the new hook. It is a reason to use it at the correct layer.

## 4. A deeper architecture for HermesKPlex

```text
Hermes Desktop / CLI / gateway
│
├── Session knowledge binding
│   ├── Connection identifier
│   ├── Selected vault
│   ├── Optional selected canvas
│   ├── Allowed capabilities
│   └── Binding generation
│
├── pre_llm_call
│   └── Optional bounded, authorized context
│
├── tool_request middleware
│   └── Normalize our tool requests
│
├── pre_tool_call
│   ├── Scope checks
│   ├── Explicit action approval
│   └── Narrow opt-in compatibility modifications
│
├── HermesKPlex tool handler
│   ├── Validate effective arguments again
│   ├── Resolve authoritative scope
│   ├── Enforce capability and request budgets
│   └── Call original K-Plex through Obsidian CLI
│
└── post_tool_call
    ├── Update ephemeral activity state
    ├── Record redacted outcome
    └── Invalidate relevant UI observations

Original Obsidian
└── Existing K-Plex runtime and public API
    ├── Compiler and evidence
    ├── Published graph
    ├── Canvas/session owners
    └── Original visualization
```

The added depth belongs in Hermes’s orchestration, not in a second K-Plex semantic engine.

## 5. High-value integration: bind each conversation to knowledge scope

This is the first feature I would build around the hook system.

### User experience

In Hermes Desktop:

```text
Knowledge scope
Vault: Work
Connection: Local Obsidian
Canvas: Architecture map, or none
Mode: Read-only
```

Then a model can call:

```json
{
  "query": "packaging decisions",
  "limit": 20
}
```

without repeatedly guessing a vault name.

### Important design constraint

A session binding is trusted plugin state, not an assertion supplied by the model.

Proposed binding:

```text
profile scope
session identity
connection identity
vault identity
binding generation
optional canvas reference
capability policy
```

The tool can accept a connection identifier, but the handler must resolve and validate it against the current binding.

### Why both middleware and handler validation?

Middleware is documented as fail-open. If a normalization callback fails, execution can continue through the normal path.[^10_2]

Therefore:

```text
Middleware improves the request.
Handler enforces its validity.
```

Do not make missing middleware equivalent to unrestricted access.

## 6. Useful pre-tool modifications

### A. Apply conservative limits to our tools

Example:

```text
kplex_neighbors(limit=200)
    ↓
Configured interactive cap: 40
    ↓
Effective limit: 40
```

If the plugin changes a user/model request, report that adjustment in the result or trace. Do not silently claim that an exhaustive request was fully satisfied.

### B. Add strictness for an explicitly strict workflow

For a user-selected “complete results only” mode, the plugin can add the equivalent of upstream `requireComplete`.

But that must mean:

> Fail explicitly if coverage is insufficient.

It must not mean:

> Trigger unbounded acquisition until the request appears complete.

### C. Resolve explicit aliases

A user-configured alias such as:

```text
“research-vault” → configured connection ID
```

can be normalized.

Do not infer aliases from arbitrary note text, tool output, or an LLM guess.

### D. Attach a selected canvas only when unambiguous and authorized

A user-selected session canvas can be used as the default for our canvas tools.

Do not choose the first or most recently focused canvas when multiple canvases exist.

### E. Narrow delegation context

For explicitly opted-in delegation workflows, we could add a bounded knowledge brief to a child’s task instructions.

This is more sensitive because it modifies another tool’s prompt and can expose private knowledge. It should be a later feature, with:

- Tool-specific schema verification.
- Explicit user enablement.
- Clear source/scope labeling.
- Token budget.
- No unrelated vault retrieval.
- Parent/child scope checks.


## 7. What pre-tool modification cannot accomplish by itself

### It does not change a decision the model already made

If the model has already emitted a tool call, adding text to that tool’s arguments does not retroactively improve the reasoning that produced it.

For example:

```text
Model chooses an unsuitable operation
    ↓
Hook adds unrelated graph context to its arguments
```

does not reliably cause the model to reconsider.

Use turn-level context or an explicit retrieval tool when the model needs knowledge before choosing an action.

### It is not a result injection interface

Returning:

```python
{"action": "modify", "context": "..."}
```

is not the documented way to append graph knowledge to model context.

The documented hook modifies `args`; `pre_llm_call` supplies turn context; tool handlers return results.[^10_3][^10_1]

### It is not a universal router

Do not add a fictitious `tool_name` field and expect the runtime to dispatch another tool.

### It is not a security sandbox

The plugin still executes inside a trusted host. A guard on known tool calls does not constrain every possible subprocess, direct API access, or other enabled extension.

## 8. Critical finding: approval ordering can be unsafe across plugins

This is the most consequential issue uncovered in the research.

The withHuman catalog documentation explicitly warns:

> Hooks running after its approval hook can change arguments, so the approved call may execute with different arguments.[^10_4]

The source-search results also include a test named:

```text
test_first_approve_wins_among_approves_and_keeps_later_modify
```

That corroborates the need to test accumulated modification and approval behavior together.

### Consequence for HermesKPlex

Do not assume:

```text
A pre-tool callback checked these arguments
    ⇒
Those exact arguments will execute
```

Another callback may contribute a later modification.

### Recommended mitigation

For our own tools:

1. Normalize early through request middleware.
2. Request approval for explicit actions.
3. Revalidate effective arguments inside the handler.
4. Resolve connection/vault scope from trusted state.
5. Reject stale or scope-changing effective arguments.
6. Avoid execution-time target rewriting.

For future sensitive writes, the handler should validate an immutable reviewed plan or equivalent approval binding. That is a proposed design requirement, not a claim that the current hook automatically provides it.

### Core enhancement worth considering

If exact approval binding is not available through the public API, propose a generic Hermes feature:

```text
Approve the final normalized execution plan
    ↓
Bind approval to its exact arguments and target
    ↓
Reject changes after approval
```

That should be a general host capability, not a HermesKPlex-specific patch.

## 9. Keep policy hooks fast

The inspected `plugins_dispatch.py` marks `pre_tool_call` as fail-closed.

It contains behavior for:

- Callback timeouts.
- Still-running callbacks.
- Suppression after a timeout.
- A cap on abandoned workers.
- Exception-to-block conversion for policy hooks.

This is stricter than generic observer-language summaries elsewhere in the documentation.

The event-hook documentation confirms that timed-out or still-running pre-tool callbacks block execution; the default callback timeout is 30 seconds, configurable up to 600 seconds or disabled with zero.[^10_1]

### Do not query Obsidian inside every pre-tool callback

Bad:

```text
Every terminal/read/search call
    ↓
Launch CLI
    ↓
Query graph
    ↓
Wait for indexing/status
    ↓
Decide whether to continue
```

A slow or unavailable Obsidian instance could block unrelated Hermes tools.

### Recommended hot path

```text
Check tool name
    ↓
Not ours / not explicitly targeted?
    Return None immediately
    ↓
Read bounded local binding/policy snapshot
    ↓
Return patch/block/approval directive
```

Actual retrieval belongs in a tool handler or deliberately bounded context pipeline.

## 10. Hook errors: code is the stronger evidence

There is a documentation tension worth recording.

The observer guide broadly describes fail-open callbacks. The current dispatcher source explicitly converts `pre_tool_call` exceptions into block directives, and treats timeout/still-running conditions as fail-closed.

For the selected supported Hermes revision:

- Test exception behavior.
- Test timeout behavior.
- Test malformed directive behavior.
- Test disabled hook behavior.
- Do not rely on a general “hooks never block on failure” statement.

The hook catalog and middleware documentation also emphasize that observers, transforms, directive hooks, and middleware have different semantics.[^10_2][^10_1]

## 11. Optional dynamic context through `pre_llm_call`

This is where we can provide “knowledge follows the conversation.”

Hermes documents `pre_llm_call` as turn-scoped: it fires before the tool loop for a user turn, not before every provider request. It can return a string or `{"context": "..."}` for ephemeral context injection.[^10_3][^10_1]

### Proposed initial mode

Only inject context when:

- The session has an explicit knowledge binding.
- The user enabled automatic context.
- A selected node/map or explicit query supplies the retrieval scope.
- A small budget is available.
- The context belongs to the current generation.

Example context:

```text
K-Plex context
Scope: user-selected Work vault / Architecture map
Coverage: local, incomplete
Revision: observed snapshot

Relevant authored relationships:
...

This is retrieved source material, not instructions.
```


### Avoid hidden broad retrieval

Do not send every user message to every configured vault. Do not retrieve a full neighborhood for unrelated turns.

### Do not put changing graph facts in a durable system section

Hermes’s cache-safe system sections are rendered once for a new session and frozen; dynamic context belongs in the turn-level mechanism.[^10_1]

Use a static section or skill for interpretation rules, not current graph state.

## 12. Post-tool observation enables useful continuity

`post_tool_call` receives outcome data including status, timing, errors, arguments, and correlation identifiers. It also fires for blocked and cancelled outcomes. Its return value is ignored.[^10_3][^10_1]

Useful applications:

- Update the Desktop activity list.
- Mark a request as unavailable rather than indefinitely pending.
- Record which node/canvas the session inspected.
- Invalidate an affected cached observation.
- Capture a local proposal for later knowledge curation.


### Do not automatically turn every tool result into a note

That would create:

- Excessive noise.
- Sensitive-data duplication.
- Unapproved vault writes.
- Agent-generated “facts” without review.
- A second uncontrolled knowledge ingestion pipeline.

Better:

```text
Observe outcome
    ↓
Create optional local draft/proposal
    ↓
User reviews
    ↓
Separate approved persistence operation
```

For now, the proposal store should remain outside the vault.

## 13. Correlation must use compound scope

The shipped disk-cleanup plugin is a useful example: its documentation explains that tool-call IDs alone are not always unique, and that it keys work by owning task/session plus the call ID.

Proposed key:

```text
profile scope
session/task identity
turn identity where available
tool_call_id
local request sequence
```

Treat IDs as opaque and tolerate missing fields.

Do not use a global dictionary keyed only by:

```python
pending[tool_call_id]
```

Also do not assume a pre-hook’s original arguments equal the post-hook’s effective arguments. Another plugin can modify them.

## 14. Illustrative plugin code

This is an architectural skeleton, not tested drop-in code. The helper classes are proposed.

```python
READ_TOOLS = {
    "kplex_search",
    "kplex_node",
    "kplex_neighbors",
    "kplex_evidence",
    "kplex_canvases",
    "kplex_canvas_context",
}

ACTION_TOOLS = {
    "kplex_navigate",
    "kplex_select_view",
    "kplex_select_preset",
}


def register(ctx):
    ctx.register_middleware("tool_request", normalize_request)
    ctx.register_hook("pre_tool_call", guard_request)
    ctx.register_hook("post_tool_call", observe_result)


def normalize_request(tool_name="", args=None, **context):
    if tool_name not in READ_TOOLS | ACTION_TOOLS:
        return None

    effective = dict(args or {})
    binding = bindings.for_context(context)

    if binding is None:
        return None

    if "connection_id" not in effective:
        effective["connection_id"] = binding.connection_id

    return {
        "args": effective,
        "source": "hermes-kplex",
        "reason": "resolved explicit session knowledge binding",
    }


def guard_request(tool_name="", args=None, **context):
    if tool_name not in READ_TOOLS | ACTION_TOOLS:
        return None

    decision = policy.check(
        tool_name=tool_name,
        args=args or {},
        context=context,
    )

    if decision.blocked:
        return {
            "action": "block",
            "message": decision.message,
        }

    return None


def observe_result(tool_name="", status="", **context):
    if tool_name not in READ_TOOLS | ACTION_TOOLS:
        return

    activity.record_metadata(
        tool_name=tool_name,
        status=status,
        context=context,
    )
```

The action-approval return is intentionally omitted here rather than inventing its full field schema. Implement it from the exact pinned host contract and test the final effective-argument behavior.

The handler must still validate `connection_id`, capabilities, scope generation, and limits before accessing Obsidian.

## 15. Recommended integration levels

| Level | Behavior | Recommendation |
| :-- | :-- | :-- |
| 0 | Explicit K-Plex tools only | Baseline |
| 1 | Session binding and argument defaults | Enable after tests |
| 2 | Readiness/scope guards and action approval | High value |
| 3 | Optional selected-map context per turn | Opt-in |
| 4 | Tool outcome continuity and curated proposals | Opt-in, no automatic writes |
| 5 | Modification of selected external tool prompts | Advanced, tool-specific |
| 6 | Replacing unrelated tool execution with K-Plex operations | Avoid initially |
| 7 | Autonomous vault mutation driven by arbitrary tool activity | Out of scope |

This lets users choose depth without making the entire agent depend on Obsidian being available.

## 16. Additional tests required for hook integration

### Directive semantics

- Two patches to different keys both survive.
- Later patch to the same key wins.
- Nested objects are replaced according to shallow-merge behavior.
- A valid block wins over an approval request.
- Later modifications do not escape handler scope validation.
- A hook returning `None` leaves unrelated tools alone.
- No in-place mutation of incoming `args`.


### Failure semantics

- Pre-tool exception blocks as expected on the supported revision.
- Timeout does not leave a late callback mutating shared arguments.
- Repeated timeout suppression does not wedge unrelated functionality.
- Normalization middleware failure cannot bypass handler policy.
- Obsidian unavailability does not block unrelated tools.


### Scope and concurrency

- Concurrent calls with repeated provider IDs remain distinct.
- Parent and child sessions do not share vault scope accidentally.
- Profile switch invalidates pending context.
- Canvas closes between normalization and execution.
- Another plugin modifies the connection argument after our hook.
- Missing correlation fields are handled safely.


### Approval integrity

- The displayed target matches the executed target.
- A later scope change is rejected.
- Action timeout causes readback, not blind replay.
- Session/permanent approval behavior is understood before enabling it.
- Execution middleware does not alter approved sensitive arguments.


### Context safety

- Retrieved text stays source data, not high-trust instructions.
- No hidden vault queries on unrelated turns.
- Token budgets are enforced.
- Coverage/truncation survives formatting.
- Stale cached facts are not presented as current.


## 17. Revised project recommendation

The architecture should now be:

```text
Explicit K-Plex tools
    +
Session-scoped knowledge binding
    +
Early request normalization
    +
Fast pre-tool policy/approval hook
    +
Validated CLI-backed handlers
    +
Outcome observation
    +
Optional turn-level context
```

Use `pre_tool_call` modification where it is genuinely useful, but do not make it the only integration mechanism. `tool_request` is the cleaner location for ordinary argument shaping because it runs before policy and approvals; `pre_tool_call` is the right control point for vetoes and approval escalation; the tool handler remains the final authority for our actual Obsidian access.[^10_2][^10_1]

The main blocker is no longer API availability. It is proving composition safety—especially final-argument approval integrity, fail-closed hook behavior, and scope isolation when other plugins are enabled. No build or runtime test has yet been performed, so those remain acceptance gates rather than established guarantees.

<span style="display:none">[^10_10][^10_11][^10_12][^10_13][^10_14][^10_15][^10_16][^10_5][^10_6][^10_7][^10_8][^10_9]</span>

<div align="center">⁂</div>

[^10_1]: https://hermes-agent.nousresearch.com/docs/user-guide/features/hooks

[^10_2]: https://hermes-agent.nousresearch.com/docs/developer-guide/middleware

[^10_3]: https://hermes-agent.nousresearch.com/docs/developer-guide/observer-hooks

[^10_4]: https://hermes-agent.nousresearch.com/docs/plugins/withhuman

[^10_5]: https://hermes-agent.nousresearch.com/docs/plugins/custom-dangerous-patterns

[^10_6]: https://hermes-agent.nousresearch.com/docs/plugins/rtk-rewrite-docker

[^10_7]: https://hermes-agent.nousresearch.com/docs/plugins/agent-hint

[^10_8]: https://hermes-agent.nousresearch.com/docs/plugins/lithermes

[^10_9]: https://hermes-agent.nousresearch.com/docs/plugins/langfuse

[^10_10]: https://hermes-agent.nousresearch.com/docs/user-guide/features/tool-search

[^10_11]: https://hermes-agent.nousresearch.com/docs/plugins/netplane

[^10_12]: https://hermes-agent.nousresearch.com/docs/zh-Hans/plugins/nerve

[^10_13]: https://hermes-agent.nousresearch.com/docs/plugins/tripwire

[^10_14]: https://hermes-agent.nousresearch.com/docs/plugins/jev-curator

[^10_15]: https://hermes-agent.nousresearch.com/docs/plugins/git-autosave

[^10_16]: https://hermes-agent.nousresearch.com/docs/developer-guide/plugins


---

# @GitHub deeper

The deeper trace reveals that HermesKPlex should be designed as a knowledge-aware tool pipeline, not just a collection of Obsidian commands with a pre-tool hook attached.

There are now three particularly important findings:

1. Hermes has multiple dispatch paths, including deferred tool discovery and connector batches; integration tests must cover the concrete tool that executes, not only the model-facing wrapper.
2. Execution middleware can change arguments after the pre-execution checks. Early normalization and pre-tool validation therefore do not establish final execution integrity by themselves.
3. The runtime preserves rewritten argument state and guards against duplicate downstream execution, which we should use rather than reproduce.

The architectural consequence is to introduce an immutable **plan** inside HermesKPlex: hooks can prepare and constrain requests, but the actual Obsidian operation executes only from a freshly validated, scope-bound plan.

I used GitHub read-only. No repository changes or runtime tests were performed.

## 1. Research baseline and new source findings

This round inspected source at the previously identified Hermes checkpoint:

```text
9242ce464b4a84fb8b7a81df18400099d2a1a790
```

Most useful files:

```text
hermes_cli/middleware.py
agent/tool_executor.py
tests/tools/test_connector_dispatch_policy.py
```

I also searched call sites and tests for:

```text
_dispatch_pre_tool_call_hooks
details.modified_args
tool-search unwrapping
pre-tool directive precedence
```

The connector returned code without citation IDs for individual files. I distinguish those direct source findings from the public documentation cited below.

## 2. More precise execution model

The earlier diagram was useful but too linear. There are several representations of a tool call:

```text
Model-emitted call
    │
    ├── Original tool name / arguments
    │
    ▼
Alias resolution and deferred-tool unwrapping
    │
    ▼
Request normalization
    │
    ├── Effective arguments
    └── Middleware trace
    │
    ▼
Pre-tool directives and normal authorization
    │
    └── Arguments may change again
    │
    ▼
Execution middleware
    │
    └── May call next_call(changed_arguments)
    │
    ▼
Concrete handler
    │
    ▼
Observation / persistence / result transformation
```

The public middleware contract confirms that request middleware runs before normal checks, while execution middleware wraps the later execution callback and can pass changed arguments onward.[^11_1]

### Why this changes our design

We should track separately:


| Representation | Meaning |
| :-- | :-- |
| Requested intent | What the model initially asked |
| Normalized request | Defaults and explicit bindings applied |
| Policy-reviewed request | What a hook evaluated |
| Effective handler arguments | What actually reached our handler |
| Execution plan | What our handler has authorized itself to do |
| Observed result | What K-Plex actually returned |

A single mutable dictionary named `args` is not an adequate conceptual model for all of those stages.

## 3. Finding: deferred tools are unwrapped before downstream policy

The inspected `agent/tool_executor.py` contains `_unwrap_tool_search_call()`.

Its purpose is to peel the generic `tool_call` bridge so downstream hooks, guardrails, checkpointing, and activity observation see the underlying tool.

It also validates:

- The concrete tool is in the session’s available scope.
- Deferred arguments match the concrete schema sufficiently for dispatch.
- Connector batches remain a special case handled per entry.


### Implication for our hook matching

For our own operations, match concrete names such as:

```text
kplex_search
kplex_neighbors
kplex_evidence
```

Do not assume every call arrives as the model-visible wrapper `tool_call`.

Conversely, do not inspect only `tool_call` and try to manually unpack everything inside our plugin. The host already owns that machinery.

### Required tests

Run the same operation through:

```text
Direct concrete tool call
Deferred tool_call discovery route
Normal sequential execution
Concurrent tool batch
```

The plugin should produce equivalent authorization and scope behavior.

## 4. Finding: connector batches apply policy per concrete entry

The inspected `test_connector_dispatch_policy.py` is unusually informative.

It tests a batch with individual composed connector tool names and verifies:

1. Request middleware sees each concrete entry.
2. The pre-tool hook sees request-rewritten arguments.
3. Execution middleware sees hook-rewritten arguments.
4. Original arguments remain separately available.
5. A blocked entry is not sent.
6. Another entry can still execute.
7. Even an empty replacement dictionary reaches the downstream transport.
8. Interrupting a batch leaves unstarted entries unsent.

### Why HermesKPlex should care

This matters if we later add knowledge context to selected connector operations—for example, drafting an issue or message informed by a user-selected map.

The right unit is the concrete operation, not the whole batch.

Proposed policy:

```text
Do not enrich an entire generic batch.
Enrich only explicitly supported concrete operations.
```

Initially, keep those integrations disabled. Our own K-Plex tools are a much easier correctness boundary.

## 5. Finding: execution middleware is single-use per frame

The actual `hermes_cli/middleware.py` implements `next_call()` with a single-use guard.

It distinguishes:

- Middleware failing before downstream execution.
- Downstream execution failing.
- Middleware failing after successful downstream execution.

The implementation preserves a successful downstream result rather than rerunning the tool when later post-processing fails.

The documentation describes the same intended behavior.[^11_1]

### What we should not do

Do not implement this retry pattern:

```python
try:
    return next_call(args)
except Exception:
    return next_call(args)
```

A timeout or exception is not proof that a side effect did not occur.

### Proposed usage

For optional telemetry:

```python
def wrap_execution(**kwargs):
    result = kwargs["next_call"](kwargs["args"])
    record_bounded_metadata(result)
    return result
```

For actual K-Plex access, prefer our normal handler. We do not need execution middleware merely to call Obsidian.

## 6. The final-argument problem is broader than hook ordering

The previous answer identified late `modify` directives after an approval request.

The deeper source adds another stage:

```text
Approval checks
    ↓
Execution middleware
    ↓
next_call(modified_args)
```

The documented execution middleware contract permits that final argument substitution.[^11_1]

This does not establish that Hermes is insecure in every configuration. Plugins are trusted code, and middleware is explicitly behavior-changing. But it means we must not claim that our pre-hook alone freezes execution targets.

### Stronger local guarantee

Inside the HermesKPlex handler:

```text
Receive effective arguments
    ↓
Validate schema
    ↓
Resolve trusted session binding
    ↓
Check current connection/vault generation
    ↓
Construct immutable local operation
    ↓
Execute only that operation
```

Never launch a CLI request directly from a loosely validated hook-provided dictionary.

### Limitation of this guarantee

If another trusted plugin replaces execution entirely, our handler may not run.

Therefore:

> Handler validation protects the operations that our handler executes; it does not sandbox other enabled plugins.

That distinction belongs in the security documentation.

## 7. Proposed immutable execution-plan model

This is an internal HermesKPlex design, not an existing Hermes API.

```python
@dataclass(frozen=True)
class KnowledgeReadPlan:
    operation: str
    connection_id: str
    binding_generation: int
    vault_reference: str
    canonical_request_json: str
    request_id: str
```

Why canonical JSON instead of a mutable nested dictionary?

Because `frozen=True` prevents attribute reassignment but does not make a nested dictionary immutable.

Alternative implementations can use recursively immutable values. The requirement is that the plan cannot be altered accidentally after validation.

### Build the plan inside the handler

```text
Tool arguments are data.
Session binding is trusted plugin state.
The plan is their validated intersection.
```

The model should not choose:

- Executable path.
- Arbitrary JavaScript.
- CLI command name.
- Working directory used to infer vault.
- A private filesystem destination.
- An unapproved remote transport.

It may choose bounded semantic inputs such as:

```text
query
returned node ID
returned canvas ID
limit within policy
strict completeness preference
```


## 8. Separate semantic request from transport request

This separation makes the integration easier to test and extend.

### Semantic request

```json
{
  "operation": "evidence",
  "sourceId": "opaque-source-id",
  "targetId": "opaque-target-id",
  "limit": 64
}
```


### Internal transport plan

```text
Known executable
Explicit vault target
Fixed dispatcher program
Encoded semantic request
Deadline
Output budget
Scope generation
```


### Benefits

- No transport details leak into agent schemas.
- A later bridge can reuse the semantic interface.
- Tests can verify operation semantics separately from CLI quoting.
- Hooks can inspect intent without parsing JavaScript.
- Diagnostics need not retain private payloads.

This also keeps the vendor fork close to upstream.

## 9. A better hook strategy: enrich intent, not execution machinery

The new pre-tool modification capability is valuable, but its best use is not generating CLI code.

### Appropriate modifications

For our own tools:

- Apply configured default limits.
- Attach an explicitly selected connection identifier.
- Resolve a user-configured scope alias.
- Apply a session-selected completeness mode.
- Supply an explicitly chosen canvas reference.


### Inappropriate modifications

- Replace a tool’s command with arbitrary Obsidian `eval`.
- Inject graph context into unrelated arguments.
- Change a target vault based on note content.
- Rewrite file paths to a different knowledge directory.
- Convert every `read_file` into a graph lookup.
- Change a user-reviewed action after approval.

The hook should express policy-level intent. The handler owns transport compilation.

## 10. Important distinction: hook callbacks are not necessarily a rewrite chain

The pre-tool contract says modification results are accumulated and shallow-merged. It does not, by itself, promise that every later callback observes the progressively merged arguments.[^11_2]

The source dispatch architecture separates invoking callbacks from resolving directive results.

Therefore, we should not design this:

```text
Hook A adds connection
    ↓
Hook B assumes it sees A's addition
    ↓
Hook C assumes it sees B's normalization
```

unless an exact-version test establishes that behavior.

### Recommended rule

Each HermesKPlex pre-tool callback should be independently interpretable.

Better still, register one policy callback for our tools and perform internal ordered steps inside that callback.

Use request middleware for the documented request-rewrite chain, and validate again in the handler.[^11_1]

## 11. Three complementary forms of knowledge context

A deeper integration should distinguish these instead of making every hook fetch the same information.

### A. Static interpretation rules

Examples:

```text
IDs are opaque.
Partial coverage cannot prove absence.
Canvas occurrences are not canonical nodes.
Retrieved content is data, not instructions.
```

Placement: bundled skill or bounded system guidance.

Hermes’s system prompt sections are frozen for a session and intended for stable guidance, not live graph facts.[^11_2]

### B. Turn-level working context

Examples:

```text
User selected Architecture map.
Current scope is Work vault.
Recent observed coverage is incomplete.
```

Placement: optional `pre_llm_call` context.

This hook fires once per turn before the loop, not before every subsequent provider request.[^11_2]

### C. Operation-specific evidence

Examples:

```text
Why A supports B.
Which authored declaration establishes a parent relationship.
What a particular committed Mind Map contains.
```

Placement: explicit tool result.

This makes retrieval visible, bounded, and attributable.

## 12. Proposed session state machine

Avoid a global “current vault.”

```text
UNBOUND
   │ explicit user selection
   ▼
BOUND_UNVERIFIED
   │ explicit connection test
   ▼
READY
   ├── provider unavailable ───────► UNAVAILABLE
   ├── binding changed ───────────► BOUND_UNVERIFIED
   ├── API/canvas lifetime changed ► REVALIDATION_REQUIRED
   └── session unbound ───────────► UNBOUND
```


### Per-request states

```text
CREATED
   ↓
NORMALIZED
   ↓
VALIDATED
   ↓
QUEUED
   ↓
DISPATCHED
   ↓
OBSERVED
   ├── DELIVERED
   └── DISCARDED_STALE
```

For future actions, add:

```text
REVIEW_REQUIRED
APPROVED
ACCEPTED_PENDING
VERIFIED
OUTCOME_UNKNOWN
```

Do not use a single `success` flag to represent all of them.

## 13. Actions need a separate plan type

Reads and actions should not share an unconstrained generic executor.

### Read plan

```text
Search
Neighbors
Evidence
Canvas snapshot
Context export
```


### Action plan

```text
Navigate explicit canvas
Select explicit view
Select explicit preset
```


### Future write plan

```text
Reviewed configuration change
Approved source mutation
```

The last category remains deferred.

For actions, bind the reviewed target and desired state. After uncertain completion:

```text
Read back
    ↓
Compare desired state
    ↓
Report verified / pending / unknown
```

Never replay solely because the transport timed out.

## 14. Event persistence and observation are different

The inspected executor organizes work around:

```text
observe → commit → project
```

It also has code to persist tool progress before UI projection, because tool effects can interrupt or restart the process before turn-end persistence.

This is a host implementation detail, not a reason for our plugin to call private persistence functions.

### Plugin consequence

A `post_tool_call` notification should not be treated as:

```text
An independently durable knowledge record.
```

If we maintain a local activity journal, give it its own clear semantics:

```text
Observed by HermesKPlex
Not necessarily a vault fact
Not automatically user-approved
Not a whole-session audit guarantee
```

Do not infer exact-once durable delivery from an observer callback.

## 15. Designing safe capture proposals

A valuable deeper feature is “turn useful agent work into reviewable knowledge.”

Proposed flow:

```text
Successful relevant tool result
    ↓
Bounded observation metadata
    ↓
Local proposal
    ↓
User reviews content and destination
    ↓
Separate approved persistence workflow
```

Proposal categories:

- Decision candidate.
- Open question.
- Evidence reference.
- Project checkpoint.
- Follow-up task.


### Keep the language honest

Prefer:

```text
Agent-reported decision candidate
```

over:

```text
Established fact
```

until the user reviews it.

### Preserve origin

A proposal should retain:

```text
source session
source operation
observation time
review status
intended destination
content fingerprint
```

Do not automatically interpret every tool result as an ontology relationship.

## 16. The most useful Desktop improvement: show the effective scope

The deeper execution pipeline argues for a compact “what will this use?” inspector.

```text
Knowledge operation
────────────────────────────────────
Connection: Local Obsidian
Vault: Work
Mode: Read-only
Operation: Explain connection
Limit: 64 decisions
Coverage requirement: Best available
Scope binding: Current
```

For altered requests:

```text
Adjusted:
- limit reduced from 200 to 64
- connection resolved from this session's explicit binding
```

This should be a factual explanation, not a secret debug log.

Do not show arbitrary evaluation source in ordinary UI.

## 17. Test the composition, not only each callback

The new source findings suggest a dedicated hostile-composition suite.


| Scenario | Required outcome |
| :-- | :-- |
| Request middleware supplies a valid connection | Handler resolves the same authorized scope |
| Later hook changes connection | Handler rejects unauthorized target |
| Execution middleware changes arguments | Handler validates effective arguments |
| Execution middleware calls downstream twice | Host single-use contract is exercised |
| Middleware raises after a successful read | No second CLI execution |
| Middleware fails before execution | Handler still enforces required scope |
| Deferred tool invocation | Concrete operation receives the same policy |
| Connector batch integration | Each supported entry handled separately |
| Empty argument replacement | Not mistaken for “no change” |
| Stale UI generation | Result discarded before insertion |
| Repeated provider call IDs | No cross-request state collision |
| API reload | Stale canvas references rejected |

Also test two non-equivalent notions of cancellation:

```text
Our waiter stopped waiting.
The original app stopped processing.
```

Do not report the second merely because the first occurred.

## 18. A narrow set of generic Hermes improvements worth investigating

These are proposals, not findings that the current host already guarantees them.

### A. Final-plan approval binding

A generic facility that binds approval to:

```text
tool identity
effective arguments
target identity
execution scope
expiry
```

Any later material change invalidates it.

### B. Post-normalization policy stage

A stage after all argument transforms but before effects would simplify exact policy evaluation.

### C. Explicit effect classification

Distinguish:

```text
read
local UI action
configuration write
source write
external communication
```

A CLI-backed tool can be read-only semantically while still launching a desktop app.

### D. Provenance for hook argument patches

Middleware supports trace metadata. A consistent trace for hook modifications would help users understand which plugin changed a request.

These should remain generic proposals, not special cases in Hermes core.

## 19. Updated implementation order

I would now split the first implementation into these small slices:

1. Pure request models and immutable plan builder.
2. CLI transport against synthetic fixtures.
3. Handler-level scope and capability enforcement.
4. Explicit K-Plex tools.
5. Request middleware for convenience defaults.
6. Fast pre-tool policy callback.
7. Redacted post-tool activity.
8. Desktop effective-scope inspector.
9. Optional selected-map turn context.
10. Reviewable knowledge-capture proposals.

Only after those work should we consider modifying other tools’ behavior or adding a persistent bridge.

## Conclusion

The deeper integration should make Hermes aware of the selected knowledge scope and K-Plex’s evidence model while preserving the original application as the semantic authority.

The strongest boundary is:

```text
Hooks prepare and constrain intent.
Handlers validate the final effective request.
Immutable plans define our actual effects.
K-Plex supplies graph semantics and evidence.
Desktop makes scope and changes visible.
```

The source confirms that Hermes already provides much of the orchestration machinery—including deferred-tool unwrapping, per-entry connector policies, and single-use execution chains. We should build on those contracts rather than bypass them, while treating final-argument approval integrity as an explicit acceptance requirement.[^11_2][^11_1]

<div align="center">⁂</div>

[^11_1]: https://hermes-agent.nousresearch.com/docs/developer-guide/middleware

[^11_2]: https://hermes-agent.nousresearch.com/docs/user-guide/features/hooks


---

# @GitHub we want a more broken down, and deeply nested hierarchical structure into separate folders that are easily nested / composed into each other, model out for this more and research more, as we'll need code-generation since we have the vendor kplex original repository, and ideally we wouldn't want to duplicate code but on plugin build / release / publish, to actually have some code generation either in Python or Rust, to see the original schema and codegen ours to that

Yes. We should organize HermesKPlex as a hierarchy of composable packages, with a build-time contract compiler that reads the pinned K-Plex vendor source and generates our language bindings, operation adapters, validation schemas, and integration metadata.

The central principle is **derivation**: upstream owns the K-Plex data contract; our repository owns exposure policy, transport, Hermes integration, and presentation. Generated files connect those layers without maintaining a second handwritten version of the upstream schema.

My recommendation is:

> Use Python to orchestrate generation and emit Python bindings, but use a small TypeScript-aware extraction stage to understand the actual vendor types. Add Rust output only if we have a concrete Rust consumer or packaging requirement.

A Python- or Rust-only parser of TypeScript would increase duplication: we would be implementing part of TypeScript’s type system instead of reusing its tooling.

I inspected GitHub read-only. Nothing was changed.

## 1. What the source review establishes

I revisited these actual vendor files:

```text
src/application/PublicApi.ts
src/application/publicApiContracts.ts
src/core/graph/model.ts
src/core/graph/relations.ts
```

They show that there is not one ready-made schema file containing everything we need.

Instead, the contract is distributed across:


| Source | Information it owns |
| :-- | :-- |
| `publicApiContracts.ts` | Result envelopes, DTOs, errors, revisions, ports, limits |
| `PublicApi.ts` | Public method signatures, inferred return types, runtime validation, capability discovery |
| `model.ts` | Branded node identity, node kinds, optional file semantics |
| `relations.ts` | Relationship roles and numeric enum values |
| Other imported contracts | Ontology, export, configuration, inspection DTOs |

That matters because “generate from the original schema” actually means:

```text
Resolve a public contract from several upstream TypeScript owners.
```

It does not mean scrape a single interface with regular expressions.

### A particularly useful discovery

Many public methods have inferred return types.

For example, `search()` calls the generic private read helper and receives an inferred `PublicApiResult<...>` result. An extractor that reads only explicit type annotations would miss or weaken that result.

TypeScript’s compiler API provides a `Program`, symbols, and a type checker for resolving this information. Its documentation explicitly describes inspecting class declarations and serializing their type information.[^12_1]

## 2. The architecture should have three separate contract layers

Do not flatten upstream DTOs, our transport protocol, and Hermes tool inputs into one schema.

```text
Upstream K-Plex contract
        │
        │ generated extraction
        ▼
Vendor-facing wire contract
        │
        │ handwritten exposure policy
        ▼
HermesKPlex service contract
        │
        │ generated projections
        ├── Python client/models
        ├── Desktop TypeScript models
        ├── Hermes tool schemas
        ├── Fixed JavaScript dispatcher
        └── Optional Rust models
```


### Layer A — Upstream contract

Examples:

```text
PublicApiResult<T>
PublicNode
PublicEvidence
PublicCanvasSnapshot
PublicApiRevision
PublicCoverage
```

We should derive these rather than rewrite them.

### Layer B — Our transport contract

Examples:

```text
request ID
operation name
connection scope
binding generation
transport error
upstream result
```

K-Plex does not own these concepts. We do.

### Layer C — Our exposed product contract

Examples:

```text
kplex_search
kplex_evidence
read-only default
maximum interactive limit
explicit canvas requirement
automatic-context preference
```

These are intentional policy and UX decisions, not information a generator can safely infer.

## 3. Proposed deeply nested repository hierarchy

The following is a target structure. It is not a request to create every empty folder immediately.

```text
HermesKPlex/
├── vendor/
│   └── kplex/                         # Pinned kplex-Vendor submodule
│
├── contracts/
│   ├── sources/
│   │   └── kplex/
│   │       ├── roots.yaml              # Exported symbols/methods we inspect
│   │       └── projection-rules.yaml   # Explicit TS → JSON wire rules
│   ├── policy/
│   │   ├── operations/
│   │   │   ├── graph.yaml
│   │   │   ├── ontology.yaml
│   │   │   ├── canvas.yaml
│   │   │   └── actions.yaml
│   │   ├── exposure.yaml               # What may become a tool/API
│   │   └── budgets.yaml                # Our stricter product budgets
│   ├── transport/
│   │   ├── request.schema.json
│   │   ├── response.schema.json
│   │   └── failure.schema.json
│   └── compatibility/
│       ├── supported-profiles.yaml
│       └── accepted-transformations.yaml
│
├── toolchain/
│   ├── contract_compiler/
│   │   ├── python/
│   │   │   └── hermeskplex_codegen/
│   │   │       ├── cli/
│   │   │       ├── orchestration/
│   │   │       ├── inputs/
│   │   │       ├── ir/
│   │   │       ├── passes/
│   │   │       │   ├── normalization/
│   │   │       │   ├── wire_projection/
│   │   │       │   ├── policy/
│   │   │       │   └── compatibility/
│   │   │       ├── emitters/
│   │   │       │   ├── json_schema/
│   │   │       │   ├── python/
│   │   │       │   ├── typescript/
│   │   │       │   ├── javascript/
│   │   │       │   ├── hermes/
│   │   │       │   └── rust/           # Optional
│   │   │       └── provenance/
│   │   └── typescript/
│   │       ├── program/
│   │       ├── symbols/
│   │       ├── signatures/
│   │       ├── constants/
│   │       ├── diagnostics/
│   │       └── emit_ir.ts
│   └── packaging/
│       ├── assemble/
│       ├── inspect/
│       ├── notices/
│       └── provenance/
│
├── packages/
│   ├── python/
│   │   └── hermeskplex/
│   │       ├── domain/
│   │       │   ├── identity/
│   │       │   ├── scope/
│   │       │   ├── capabilities/
│   │       │   └── plans/
│   │       ├── application/
│   │       │   ├── knowledge/
│   │       │   │   ├── graph/
│   │       │   │   ├── ontology/
│   │       │   │   └── canvases/
│   │       │   ├── context/
│   │       │   └── proposals/
│   │       ├── ports/
│   │       │   ├── provider/
│   │       │   ├── transport/
│   │       │   ├── storage/
│   │       │   └── clock/
│   │       ├── adapters/
│   │       │   ├── obsidian/
│   │       │   │   └── cli/
│   │       │   │       ├── discovery/
│   │       │   │       ├── compilation/
│   │       │   │       ├── execution/
│   │       │   │       └── decoding/
│   │       │   ├── storage/
│   │       │   └── diagnostics/
│   │       ├── generated/
│   │       │   ├── kplex/
│   │       │   │   └── v1/
│   │       │   │       ├── models/
│   │       │   │       ├── operations/
│   │       │   │       ├── constants/
│   │       │   │       └── client/
│   │       │   └── hermes/
│   │       │       ├── schemas/
│   │       │       └── inventory/
│   │       └── composition/
│   │           ├── providers/
│   │           ├── services/
│   │           └── lifecycle/
│   │
│   └── desktop/
│       ├── src/
│       │   ├── features/
│       │   │   ├── connection/
│       │   │   ├── search/
│       │   │   ├── relationships/
│       │   │   ├── evidence/
│       │   │   ├── canvases/
│       │   │   └── context_preview/
│       │   ├── shared/
│       │   │   ├── ui/
│       │   │   ├── formatting/
│       │   │   └── scope/
│       │   ├── generated/
│       │   │   ├── kplex/
│       │   │   └── transport/
│       │   └── composition/
│       │       ├── routes/
│       │       ├── panes/
│       │       ├── commands/
│       │       └── settings/
│       └── build/
│
├── integrations/
│   └── hermes/
│       ├── agent/
│       │   ├── entrypoint/
│       │   ├── tools/
│       │   ├── hooks/
│       │   │   ├── pre_tool/
│       │   │   ├── post_tool/
│       │   │   └── pre_llm/
│       │   ├── middleware/
│       │   │   └── tool_request/
│       │   └── skills/
│       ├── backend/
│       │   ├── routes/
│       │   ├── validation/
│       │   └── lifecycle/
│       └── manifest/
│
├── tests/
│   ├── extraction/
│   ├── generation/
│   ├── contracts/
│   │   ├── valid/
│   │   ├── invalid/
│   │   └── cross_language/
│   ├── runtime/
│   ├── hooks/
│   ├── packaging/
│   └── native/
│
├── docs/
│   ├── architecture/
│   │   ├── decisions/
│   │   ├── boundaries/
│   │   └── generation/
│   ├── compatibility/
│   └── validation/
│
└── .github/
    └── workflows/
```


### Do not equate folders with packages

Most of these folders should remain ordinary modules.

Create an independently versioned package only when it has a genuine consumer or lifecycle boundary. Otherwise we will produce a monorepo with dozens of packages and very little useful modularity.

The initial Python runtime can be one package with strict internal dependency rules.

## 4. Dependency rules make the hierarchy meaningful

Proposed allowed direction:

```text
Hermes entrypoints
        ↓
Application services
        ↓
Domain values and abstract ports

Concrete adapters
        ├── implement ports
        └── consume generated upstream bindings

Composition roots
        └── wire services, ports, and adapters
```


### Enforce these rules

- `domain/` must not import Hermes, subprocess, or generated transport clients.
- `application/` must not construct subprocess commands.
- `generated/` must not import handwritten application services.
- Desktop features must not invoke processes.
- CLI decoding must not decide user authorization.
- Hooks must call application policies, not duplicate them.
- Generated schemas must not silently determine exposure policy.

Folders provide navigation; import checks provide architecture.

## 5. Code generation should act like a compiler

The complete pipeline should be:

```text
Pinned vendor checkout
        ↓
TypeScript program and symbol resolution
        ↓
Extracted contract IR
        ↓
Wire projection
        ↓
Exposure/policy composition
        ↓
Normalized contract IR
        ↓
Language and platform emitters
        ↓
Compilation + conformance tests
        ↓
Release assembly
```

There are now three distinct compilers:


| Compiler | Purpose |
| :-- | :-- |
| K-Plex semantic compiler | Notes/source facts → graph and evidence |
| Our contract compiler | Upstream API types → bindings and schemas |
| Build compilers | TypeScript/Python packaging → installable artifacts |

We reuse the first, implement a small version of the second, and configure the third.

## 6. Why a TypeScript extraction stage is necessary

The inspected upstream types include:

```ts
type NodeId = string & {
  readonly __nodeId: unique symbol
};
```

Also:

```ts
PublicApiResult<T>
Readonly<T>
Exclude<Role, "sibling">
Promise<...>
Parameters<...>
```

Some methods have inferred return types. Some nested API operations are properties assigned in the constructor.

A textual parser will not resolve these correctly.

### Recommended implementation

Python launches a pinned local Node script that:

1. Loads a dedicated schema-extraction `tsconfig`.
2. Creates a TypeScript program.
3. Checks diagnostics.
4. Resolves selected exported symbols.
5. Examines public method call signatures.
6. Resolves parameter and return types.
7. Emits a stable intermediate representation.

The official compiler API supports resolving symbols and types through the program’s checker.[^12_1]

This does not require shipping Node as a HermesKPlex runtime dependency. It is build tooling.

## 7. Intermediate representation: do not use raw TypeScript text

Proposed IR:

```json
{
  "irVersion": 1,
  "provider": {
    "name": "kplex",
    "apiVersion": 1,
    "vendorCommit": "<actual pinned commit>"
  },
  "operations": [
    {
      "path": ["search"],
      "parameters": [
        {
          "name": "options",
          "optional": false,
          "typeRef": "SearchOptions"
        }
      ],
      "resultTypeRef": "SearchResult",
      "async": false
    }
  ],
  "types": {},
  "constants": {},
  "provenance": {}
}
```

The names `SearchOptions` and `SearchResult` may be stable generated names for anonymous or inferred upstream types.

### IR must preserve

- Optional versus required.
- Nullable versus non-nullable.
- Literal discriminators.
- Union members.
- Arrays and ordering.
- Maps/index signatures.
- Named references.
- Numeric enum values.
- Function members.
- Promise/async boundary.
- Source symbol provenance.
- Unknown/unsupported constructs.


### Unsupported types should fail generation

Do not silently turn an unresolved type into:

```python
Any
```

or:

```rust
serde_json::Value
```

That would make the build green by discarding the contract.

Opaque JSON fields should be explicit, reviewed cases.

## 8. Wire projection is an explicit transformation

The original TypeScript contract is not identical to JSON.

### Rule A — Branded identity

Upstream:

```ts
NodeId = string & brand
```

Wire:

```text
string
```

Python:

```text
opaque string alias or validated wrapper
```

Do not interpret the ID as a filesystem path.

### Rule B — Callback removal

Upstream:

```ts
isCancelled?: () => boolean
```

Wire:

```text
not representable as JSON
```

Remove it only through a named, tested projection rule.

Our transport cancellation is a separate mechanism. Omitting the callback must not be described as preserving identical cancellation behavior inside Obsidian.

### Rule C — Promise unwrapping

Upstream:

```ts
Promise<PublicApiResult<T>>
```

Wire:

```text
PublicApiResult<T>
```

IR still records that the invocation must be awaited.

### Rule D — Missing versus null

Upstream distinguishes:

```ts
file?: FileFacet
url: string | null
mtime?: number | null
```

Those correspond to different accepted JSON states.

Generated Python and Rust models must not collapse every optional field into an indistinguishable null.

### Rule E — Numeric enums

The inspected source declares:

```text
RelationType.DEFINED = 1
RelationType.INFERRED = 2

LinkDirection.TO = 1
LinkDirection.FROM = 2
LinkDirection.BOTH = 3
```

However, some public DTO fields are typed as `number`, not as the enum.

Generate the actual DTO constraint faithfully. Do not tighten `number` to an enum without an explicit policy transformation.

## 9. Types alone cannot recover all validation rules

This is a crucial limitation.

The inspected `PublicApi.ts` contains runtime rules such as:

- Search query length checks.
- Integer page-limit validation.
- Nonempty bounded identity checks.
- Per-operation limits.
- Completeness checks.
- Revision fencing.
- Output structure and size budgets.

These are not all encoded in TypeScript types.

Therefore:

```text
Generated type schema ≠ complete behavioral specification
```


### Three kinds of constraints

| Constraint source | Example | Treatment |
| :-- | :-- | :-- |
| Type-derived | `role` is a finite union | Generate |
| Constant-derived | Evidence/page maximum | Extract safely |
| Behavioral | Strict completeness rejects partial results | Test/document; do not infer casually |

### Avoid building a general source-code theorem prover

Do not try to automatically understand every `if` statement in `PublicApi.ts`.

Instead:

- Extract designated constants with a restricted static evaluator.
- Add explicit references from our policy to those constants.
- Keep a small reviewed behavioral overlay.
- Validate that overlay against actual upstream behavior.

If a constraint cannot be extracted confidently, generation should say so.

## 10. Handwritten exposure policy is essential

Automatic discovery must not automatically expose every discovered API operation.

For example, discovering:

```text
applyOntologyConfiguration
configuration.apply
```

must not publish a write tool.

Proposed policy:

```yaml
operations:
  search:
    upstream_path: [search]
    exposure:
      agent_tool: kplex_search
      desktop: true
    effect: published-graph-read

  evidence:
    upstream_path: [evidence]
    exposure:
      agent_tool: kplex_evidence
      desktop: true
    effect: published-graph-read

  navigate:
    upstream_path: [navigate]
    exposure:
      agent_tool: false
      desktop: explicit-action
    effect: canvas-navigation

  applyOntologyConfiguration:
    exposure:
      agent_tool: false
      desktop: false
```

This policy names operations and effects. It should not duplicate complete request/response field definitions.

### Fail closed on newly discovered operations

A vendor update that adds an API method should produce:

```text
New operation discovered: review exposure policy.
Not exposed automatically.
```

That is the correct default.

## 11. What should be generated

### Generate

- Upstream wire DTOs.
- Operation input/result schemas.
- Python models.
- Python operation client methods.
- TypeScript Desktop types.
- Operation-to-dispatch metadata.
- Fixed JavaScript call dispatch.
- Error/role constants.
- Upstream limit references.
- Hermes tool parameter schemas.
- Manifest tool/hook inventory fragments.
- Compatibility and provenance reports.
- Documentation tables.


### Keep handwritten

- Approval policy.
- Session binding.
- Immutable execution plans.
- Process lifecycle.
- Vault/executable configuration.
- Retry policy.
- Context selection.
- UI layouts.
- User-facing tool descriptions.
- Authorization decisions.
- Knowledge-capture proposals.


### Generate glue, not meaning

Do not generate a Python relationship classifier from the TypeScript compiler.

Do not translate layout algorithms.

Do not turn upstream comments into trusted system instructions automatically.

## 12. Python versus Rust

### Recommended first implementation: Python coordinator + TypeScript extractor

Reasons specific to this project:

- Hermes’s agent plugin is Python.
- Python can orchestrate extraction, schema normalization, packaging, and checks.
- Existing generators can emit Pydantic models from JSON Schema.
- No native extension is needed.
- We avoid Rust binary distribution before demonstrating a need.

`datamodel-code-generator` supports JSON Schema input and multiple Python outputs, including Pydantic models, dataclasses, TypedDict, and msgspec structures.[^12_2]

### Optional Rust consumer

Rust is reasonable if we later need:

- A standalone distribution tool.
- A native packaging utility.
- A separately deployed local relay.
- A high-assurance shared parser with an actual consumer.

`typify` generates Rust types from JSON Schema and can run through CLI, macros, build scripts, or persistent generation. Its own documentation warns that some schema constructs and numeric constraints are not perfectly represented.[^12_3]

### Do not use Rust merely as another generation layer

This would add complexity without solving the TypeScript-resolution problem:

```text
TypeScript → custom Rust parser → Python
```

Better:

```text
TypeScript-aware extraction → stable IR/JSON Schema
                               ├── Python
                               ├── TypeScript
                               └── Rust, if needed
```


## 13. Candidate tools and their limitations

| Tool | Proposed role | Caution |
| :-- | :-- | :-- |
| TypeScript compiler API | Resolve public signatures and inferred types | Pin version and test extraction |
| `ts-json-schema-generator` | Emit schemas for admitted DTOs | Requires custom handling for our wire projection |
| `datamodel-code-generator` | Python models | Test optionality, unions, aliases, validation settings |
| `typify` | Optional Rust DTOs | Generated types do not enforce every schema constraint |
| JSON Schema validator | Boundary conformance | Match supported dialect and actual constraints |

`ts-json-schema-generator` supports aliases, generics, conditional types, Promise unwrapping, and custom parsing/formatting. It also exposes function handling modes, including explicit failure.[^12_4]

Use function handling that fails by default. A broad “hide every function” setting could conceal an unintended contract change.

## 14. A concrete extraction entry point

We can generate a temporary TypeScript file containing type references, not copied definitions:

```ts
import type { PublicApi } from
  "<resolved-vendor-path>/src/application/PublicApi";

type SearchRequest =
  Parameters<PublicApi["search"]>[^12_0];

type SearchResponse =
  Awaited<ReturnType<PublicApi["search"]>>;
```

Advantages:

- No handwritten duplicate field lists.
- Vendor changes flow into the result.
- Inferred return types are resolved.
- A missing/renamed method fails.

The extractor should also record whether the parameter is optional or has a default. Merely indexing `Parameters<T>[^12_0]` is not enough to describe invocation arity perfectly.

Nested methods such as `configuration.list` require member-path resolution, not a flat list of class methods.

## 15. Generated dispatcher must remain allowlisted

A generated dispatcher should enumerate only approved operations:

```javascript
switch (request.operation) {
  case "search":
    return await api.search(request.arguments);

  case "evidence":
    return await api.evidence(request.arguments);

  default:
    throw new Error("Unsupported operation");
}
```

Do not generate:

```javascript
return api[request.operation](request.arguments);
```

over the full provider object.

The generated dispatcher must also preserve actual call arity. Not every operation necessarily has the same invocation shape.

Capability discovery remains necessary at runtime: build-time existence in the vendor source does not prove the user installed that exact plugin.

## 16. Prevent generated schemas from overclaiming fidelity

Several cross-language hazards need explicit tests.

### Optional fields

Python generators may represent both missing and null with `None` unless configured and tested carefully.

### Numeric coercion

Reject unintended conversions such as:

```text
"20" → 20
true → 1
```

where the boundary contract requires actual numeric/integer inputs.

### String length units

JavaScript string lengths and offsets can use UTF-16 code units. Python string length counts differently for some Unicode characters.

Do not map an upstream UTF-16 bound directly to a Python character count and claim equivalence.

### Arrays

Preserve order and multiplicity. Evidence declarations and repeated scene occurrences must not become sets.

### Unknown fields

Recommended policy:

- Our request schemas: strict.
- Upstream responses: deliberate compatibility policy, not accidental silent dropping.
- New response fields: report drift without necessarily disabling otherwise compatible reads.
- New required request fields: block affected operations until reviewed.


## 17. Build and generation lifecycle

Run the same compiler in development, CI, and release.

```text
codegen extract
    ↓
codegen normalize
    ↓
codegen validate
    ↓
codegen emit
    ↓
codegen verify
    ↓
build runtime packages
    ↓
package release
```

These are proposed commands, not existing scripts.

### Do not generate only during publish

That hides contract changes until the last moment.

Instead:

- Generate on vendor updates.
- Verify on pull requests.
- Reproduce on release.
- Publish only already-reviewed outputs.


### Runtime installation should not require generation

Users should receive:

```text
Python bindings
Desktop ESM
Fixed dispatcher
Schemas
Manifest
Skill
Compatibility metadata
```

They should not need TypeScript extraction, Rust compilation, or a vendor checkout simply to use the plugin.

## 18. Generated source policy

I recommend checking in generated runtime bindings and schemas initially.

Benefits:

- Reviewable vendor diffs.
- Easier source-based installation.
- No hidden network/build step at installation.
- Release provenance is inspectable.

Keep transient extraction intermediates outside the distributable package unless useful as evidence.

### Every generated file should identify

```text
Do not edit
Generator version
Vendor commit
Contract fingerprint
Generation configuration fingerprint
```

Avoid wall-clock timestamps in deterministic generated code.

### Generated is not necessarily runtime-dynamic

A pre-generated client is ordinary source code. It need not carry the generator or compiler in production.

## 19. Determinism and provenance

A generation fingerprint should include:

```text
Vendor commit
Relevant source closure hashes
Extractor version
TypeScript version
Generator versions
Projection rules
Exposure policy
Emitter configuration
Formatting configuration
```


### Determinism check

Generate twice into clean directories:

```text
Run A
Run B
    ↓
Byte comparison
```

Also test different checkout paths so absolute filesystem paths do not leak into output.

Stable ordering must be imposed on:

- Types.
- Properties.
- Operations.
- Definitions.
- Imports.
- Manifest inventories.

Do not rely on incidental directory traversal order.

## 20. Schema drift should become a review artifact

Proposed report:

```text
Vendor contract change
────────────────────────────────────
search:
  request unchanged
  response added optional field

neighbors:
  role union changed

canvas:
  new optional fold-state member

new operation:
  not exposed

unresolved construct:
  generation blocked
```

Separate compatibility classes:


| Change | Default handling |
| :-- | :-- |
| Documentation-only | Regenerate docs |
| New optional response property | Review; often compatible |
| New union member | Review downstream exhaustive handling |
| Removed response property we use | Breaking |
| New required request property | Breaking for affected operation |
| Optional capability added | Not auto-exposed |
| Limit reduced | Revalidate policy and fixtures |
| Behavioral change | Native acceptance required |

General schema compatibility is not trivial. Start with conservative rules and human review rather than claim a complete automated proof.

## 21. GitHub Actions structure

Proposed parent workflows:

```text
.github/workflows/
├── verify-contracts.yml
├── verify-python.yml
├── verify-desktop.yml
├── verify-vendor.yml
├── verify-package.yml
├── native-acceptance.yml
└── release.yml
```


### Contract workflow

```text
Checkout pinned submodule
    ↓
Verify vendor identity
    ↓
Install locked toolchain
    ↓
Extract contract
    ↓
Generate into clean directory
    ↓
Compare checked-in outputs
    ↓
Compile generated Python/TypeScript
    ↓
Run cross-language fixtures
    ↓
Upload drift/provenance report
```


### Important checks

- No `git submodule update --remote`.
- No unpinned `npx` dependency download.
- No remote `$ref` retrieval during generation.
- No importing/running the Obsidian application entry point to discover types.
- No arbitrary evaluation of upstream constants.
- No generated path escaping output roots.
- No automatic publication after unreviewed schema drift.

Source inspection should be static where possible.

### Release workflow

Build once from exact inputs, verify the archive, then attest and publish that artifact. Do not run a different generator configuration in publish.

## 22. Test corpus for the generator itself

Minimum fixtures:

```text
Branded string
Optional nullable field
Literal-discriminated union
Generic result envelope
Readonly arrays
Record/index signature
Numeric enum
Imported type alias
Excluded union member
Inferred method return
Defaulted optional parameter
Promise result
Nested operation path
Callback projection
Recursive JSON type
Unsupported function/object
```

Then add actual upstream examples:

```text
PublicNode
PublicEvidence
PublicCanvasSnapshot
PublicApiResult<PublicNode | null>
search request/result
canvas request/result
```

A generator that works on a toy interface but fails on `PublicCanvasSnapshot` is not yet useful.

## 23. How this composes with the hook architecture

Generated operation metadata feeds the handwritten hook policy:

```text
Generated operation registry
    ├── request schema
    ├── response schema
    ├── required capability
    └── upstream path

Handwritten policy
    ├── allowed surface
    ├── effect category
    ├── approval requirement
    ├── session binding rules
    └── budget

Composition root
    ├── register tool
    ├── register middleware
    ├── register policy hook
    └── construct handler
```

This avoids separate handwritten lists in:

- Tool registration.
- Hook matching.
- Manifest declarations.
- Dispatcher generation.
- Desktop controls.

But authorization remains handwritten and reviewed. A generated list of discovered methods must never become automatic permission.

## 24. Recommended first implementation slice

Before building the full hierarchy, prove this path:

```text
Pinned vendor
    ↓
Extract search / node / evidence
    ↓
Generate JSON schemas
    ↓
Generate Python bindings
    ↓
Generate allowlisted dispatcher
    ↓
Type-check and validate fixtures
    ↓
Invoke original K-Plex in a disposable vault
```

Acceptance criteria:

1. No handwritten duplicate DTO definitions.
2. An upstream field change produces an explicit generation diff.
3. Unsupported types fail instead of degrading to `Any`.
4. Callback removal is explicit.
5. Optional/null semantics survive.
6. Existing upstream envelopes remain intact.
7. New operations remain unexposed.
8. Runtime users do not need build tools.
9. Generated artifacts contain exact provenance.
10. No K-Plex compiler or layout implementation is duplicated.

## Recommended decision

Adopt a contract-compiler architecture with a deeply nested but dependency-enforced package structure.

Use:

```text
Python orchestration
    +
TypeScript-aware extraction
    +
Canonical IR / JSON Schema
    +
Generated Python and Desktop bindings
    +
Handwritten exposure and authorization policy
```

Keep Rust as an optional target, not a prerequisite. This achieves the underlying goal—minimal schema duplication and vendor-aligned builds—without replacing TypeScript’s type resolver, over-generating unsafe tools, or forcing end users to compile the project. The available generators support the necessary building blocks, but their limitations mean conformance tests and explicit wire-projection rules are essential.[^12_2][^12_3][^12_4][^12_1]

<span style="display:none">[^12_10][^12_11][^12_12][^12_13][^12_14][^12_15][^12_16][^12_17][^12_18][^12_19][^12_20][^12_21][^12_22][^12_23][^12_24][^12_25][^12_26][^12_27][^12_28][^12_29][^12_30][^12_31][^12_32][^12_5][^12_6][^12_7][^12_8][^12_9]</span>

<div align="center">⁂</div>

[^12_1]: https://github.com/Microsoft/TypeScript/wiki/Using-the-Compiler-API

[^12_2]: https://github.com/datamodel-code-generator/datamodel-code-generator

[^12_3]: https://github.com/oxidecomputer/typify

[^12_4]: https://github.com/vega/ts-json-schema-generator

[^12_5]: https://pydantic.dev/docs/validation/latest/integrations/dev-tools/datamodel_code_generator/

[^12_6]: https://pydantic.dev/docs/validation/2.12/integrations/datamodel_code_generator/

[^12_7]: https://ajv.js.org/guide/typescript.html

[^12_8]: https://www.npmjs.com/package/ts-json-schema-generator

[^12_9]: https://www.npmjs.com/package/typescript-json-schema

[^12_10]: https://github.com/datamodel-code-generator/datamodel-code-generator/blob/main/docs/index.md

[^12_11]: https://jsonrender.com/blog/generating-deterministic-json-schemas-from-typescript-types.html

[^12_12]: https://mozg.sh/b/mozg/pydantic-integrations/notes/datamodel-code-generator

[^12_13]: https://jsonic.io/guides/json-schema-vs-typescript

[^12_14]: https://github.com/sberan/tjs

[^12_15]: https://github.com/datamodel-code-generator/datamodel-code-generator/blob/main/docs/python-model.md

[^12_16]: https://github.com/datamodel-code-generator

[^12_17]: https://github.com/guardicore/datamodel-code-generator

[^12_18]: https://github.com/datamodel-code-generator/datamodel-code-generator/blob/main/docs/supported_formats.md

[^12_19]: https://github.com/microsoft/TypeScript/issues/37711

[^12_20]: https://github.com/microsoft/TypeScript/issues/52489

[^12_21]: https://github.com/microsoft/TypeScript/issues/59637

[^12_22]: https://github.com/microsoft/TypeScript/issues/52823

[^12_23]: https://github.com/microsoft/TypeScript/wiki/Codebase-Compiler-Checker

[^12_24]: https://github.com/microsoft/TypeScript/issues/5036

[^12_25]: https://github.com/microsoft/TypeScript/issues/12632

[^12_26]: https://github.com/microsoft/TypeScript/issues/13225

[^12_27]: https://github.com/microsoft/TypeScript/issues/6686

[^12_28]: https://github.com/microsoft/TypeScript/issues/58977

[^12_29]: https://github.com/microsoft/TypeScript/wiki/Using-the-Compiler-API/20d39bf0a6d4e5c85e1f45e7fcade1cc2954abfa

[^12_30]: https://github.com/microsoft/TypeScript/wiki/Using-the-Compiler-API/38206fde051e37bcf0cb11a29068a1e9f9cc8a14

[^12_31]: https://github.com/microsoft/TypeScript/wiki/Using-the-Compiler-API/fdac0fd80fcd4143154898d0bf1aab97955fc266

[^12_32]: https://github.com/microsoft/TypeScript/issues/3015


---

# @GitHub continue

The next refinement should be a feature-composed contract compiler: small nested modules, each with explicit inputs and outputs, assembled into a build graph. It should generate bindings from the pinned K-Plex source, but not infer product permissions or runtime semantics from types alone.

The latest GitHub inspection also uncovered a useful alternative schema source—K-Plex’s embedded GraphQL schema—and a concrete Hermes precedent for keeping generated documentation and validation tied to one machine-readable authority.

My recommendation is to make the intermediate representation, or **IR**, the stable boundary between extraction, policy, generation, packaging, and compatibility checking.

No GitHub changes were made.

## 1. New source findings

This round inspected:

```text
K-Plex vendor:
  tsconfig.json
  src/adapters/obsidian/publicApiGraphql.ts

Hermes:
  scripts/gen_plugin_api_docs.py
```

These add three important constraints to the previous design.

### Finding A — Extraction must respect the vendor compiler configuration

The inspected K-Plex configuration uses:

```text
strictNullChecks: true
noImplicitAny: true
moduleResolution: Bundler
target: ES2021
jsx: react-jsx
DOM and ES2021 libraries
@/* path mapping
```

A generator using a different TypeScript configuration can resolve a different contract—or fail to preserve meaningful nullability.

We should derive an extraction configuration from the pinned vendor configuration, with explicitly recorded overrides, rather than create an unrelated default configuration.

### Finding B — GraphQL is already a schema projection, but not the full API

The inspected `publicApiGraphql.ts` contains a literal SDL schema and a bounded, synchronous, read-only query adapter.

It exposes useful operations such as:

```text
status
coverage
ontology
node
search
neighbors
evidence
canvases
canvas
context
capabilities
```

But it is not equivalent to the complete JavaScript API:

- It exposes only selected synchronous reads.
- It does not expose the full action/configuration surface.
- Several finite TypeScript unions become GraphQL strings.
- The GraphQL node shape omits some TypeScript DTO fields.
- The inspected canvas SDL does not include every field in the TypeScript snapshot contract.
- It unwraps upstream envelopes into GraphQL values and field errors.

Therefore, GraphQL is a useful secondary contract surface and cross-check—not a universal replacement for TypeScript extraction.

### Finding C — Hermes already uses a single-source generation pattern

The inspected Hermes script generates its public-plugin API documentation from:

```text
hermes_cli/data/plugin_api.json
```

The same JSON file is used by plugin-import validation.

Its `--check` mode renders the expected output and fails if the committed documentation is stale.

That is the pattern we should follow:

```text
One authority
    ├── executable validation
    ├── generated bindings
    ├── generated inventory
    └── generated documentation
```

Not four separately maintained lists.

## 2. Refine the hierarchy around feature capsules

The previous tree separated technical layers thoroughly. We can improve composability by introducing feature capsules inside those layers.

A capsule represents one bounded capability, such as graph search or canvas export.

```text
features/
├── graph/
│   ├── search/
│   ├── node/
│   ├── neighbors/
│   └── evidence/
├── ontology/
│   ├── vocabulary/
│   ├── guidance/
│   └── completion/
└── canvas/
    ├── discovery/
    ├── snapshot/
    ├── context/
    └── markdown_export/
```

Each capsule has the same internal vocabulary.

### Example: search capsule

```text
features/graph/search/
├── contract/
│   ├── source.yaml
│   └── exposure.yaml
├── application/
│   ├── request.py
│   ├── service.py
│   └── formatting.py
├── integration/
│   ├── hermes_tool.py
│   └── desktop_projection.ts
└── tests/
    ├── contract_cases.json
    ├── test_policy.py
    └── test_service.py
```

Generated files live in a separate generated tree mirroring the feature identity:

```text
generated/
├── contracts/
│   └── graph/search/
├── python/
│   └── graph/search/
├── typescript/
│   └── graph/search/
└── dispatch/
    └── graph/search/
```


### Why keep generated output separate?

It makes these distinctions unambiguous:

```text
Handwritten policy
Generated upstream binding
Handwritten orchestration
Generated registration metadata
```

Do not mix generated and handwritten code in a file using editable marker regions. Prefer imports and composition.

## 3. A capsule descriptor should reference types, not repeat them

Proposed descriptor:

```yaml
id: graph.search

upstream:
  provider: kplex
  root: PublicApi
  member_path:
    - search

projection:
  transport: json
  ruleset: kplex-public-read-v1

exposure:
  agent:
    tool_name: kplex_search
  desktop:
    feature: knowledge-search

policy:
  effect: published-graph-read
  scope: explicit-session-binding
  budget_profile: interactive-search
```

This descriptor deliberately does not contain:

```yaml
query:
  type: string
limit:
  type: number
```

Those fields should come from the resolved upstream request type.

### Product-only arguments remain explicit

Suppose our tool adds:

```text
connection_id
```

That is not an upstream K-Plex request field. The compiler should compose:

```text
Generated upstream request
    +
Handwritten integration scope
    =
Hermes tool input
```

The transport compiler then separates them again:

```text
connection_id → resolve trusted binding
query/limit   → upstream request
```

This prevents accidentally forwarding integration metadata into the original API.

## 4. Make composition explicit through a build graph

Rather than a large script that performs everything in one function, define stages with content-addressed inputs.

```text
Vendor verification
    ↓
TypeScript program
    ↓
Public symbol extraction
    ↓
Type normalization
    ↓
JSON wire projection
    ↓
Feature-policy composition
    ↓
Contract validation
    ├── Python emitter
    ├── TypeScript emitter
    ├── Dispatcher emitter
    ├── Hermes schema emitter
    └── Documentation emitter
            ↓
        Package assembly
            ↓
        Archive verification
```


### Stage contract

Every stage should declare:

```text
Input artifacts
Configuration
Tool versions
Output artifacts
Diagnostics
Fingerprint
```

Example:

```text
wire_projection
inputs:
  extracted-types.ir.json
  projection-rules.yaml

outputs:
  wire-types.ir.json
  projection-report.json
```

This makes it possible to regenerate only affected outputs without hiding dependencies.

## 5. More detailed compiler nesting

A focused compiler layout:

```text
toolchain/contract_compiler/
├── driver/
│   ├── commands/
│   │   ├── extract.py
│   │   ├── generate.py
│   │   ├── check.py
│   │   ├── diff.py
│   │   └── package.py
│   └── pipeline/
│       ├── graph.py
│       ├── executor.py
│       └── artifacts.py
│
├── frontends/
│   ├── typescript/
│   │   ├── configuration/
│   │   ├── program/
│   │   ├── symbols/
│   │   ├── signatures/
│   │   └── constants/
│   └── graphql/
│       ├── extraction/
│       ├── parsing/
│       └── comparison/
│
├── ir/
│   ├── types/
│   ├── operations/
│   ├── effects/
│   ├── provenance/
│   └── validation/
│
├── passes/
│   ├── resolve_names/
│   ├── normalize_types/
│   ├── project_wire/
│   ├── compose_features/
│   ├── enforce_exposure/
│   ├── apply_budgets/
│   └── classify_changes/
│
├── backends/
│   ├── json_schema/
│   ├── python/
│   ├── typescript/
│   ├── javascript_dispatch/
│   ├── hermes_inventory/
│   └── documentation/
│
└── verification/
    ├── deterministic_output/
    ├── compilation/
    ├── fixtures/
    └── compatibility/
```

These are logical modules. Start with ordinary source directories, not a separate installable package for every folder.

## 6. Use two contract frontends without creating two authorities

There are now two extractable upstream representations.

### TypeScript frontend: primary authority

Use it for:

- Full selected JavaScript API signatures.
- DTOs.
- Envelopes.
- Optional/null distinctions.
- Literal unions.
- Async methods.
- Export types.
- Nested public members.


### GraphQL frontend: projection cross-check

Use it to answer:

- Which synchronous reads are exposed through GraphQL?
- Which DTO fields are intentionally absent?
- Which types become less precise?
- Which defaults are present in SDL?
- Which return shapes differ from direct calls?

Do not merge the schemas indiscriminately.

### Proposed comparison report

```text
TypeScript → GraphQL projection

PublicNode:
  present: id, name, kind, resolution, aliases, tags
  absent from inspected SDL: selected styling metadata
  relaxed: kind literal union → String

PublicCanvasSnapshot:
  present: scene, camera, occurrences, connections
  absent from inspected SDL: selected fold-state details

Operations:
  direct-only: exportMarkdown, actions, selected async reads
```

The report should distinguish expected omissions from unexpected drift.

## 7. Extract GraphQL SDL statically, not by launching Obsidian

The inspected adapter defines a literal:

```ts
const schemaText = `...`;
```

We can read that through the TypeScript AST.

Recommended rules:

- Accept a plain static string/template literal.
- Reject interpolation unless a restricted resolver explicitly supports it.
- Parse SDL with a real GraphQL parser.
- Do not import and execute the entire adapter to retrieve the string.
- Do not start Obsidian as part of static generation.

If the vendor later changes to dynamic schema construction, fail extraction with a clear diagnostic.

This keeps contract generation separate from runtime application acceptance.

## 8. Choose one normalized schema dialect

JSON Schema tools do not all have identical dialect support or output conventions.

The compiler should explicitly choose and record:

```text
Canonical schema dialect
Reference layout
Null representation
Union representation
Additional-property policy
Naming rules
Extension keywords
```

Do not allow each emitter to invent these independently.

`ts-json-schema-generator` supports custom parsing/formatting and dedicated schema-generation configuration, but also has configurable behavior for functions, extra properties, and type exposure. Those settings must be pinned and tested.[^13_1]

### Important distinction

The IR can preserve more information than JSON Schema.

For example:

```text
source symbol
optional parameter versus optional property
await requirement
effect classification
upstream capability name
UTF-16 offset semantics
```

Keep those in IR metadata rather than trying to encode everything into generic JSON Schema constraints.

## 9. Directional compatibility is more useful than one “breaking” flag

Compatibility depends on whether we send or receive data.

### Requests we send

If upstream adds a required request field, our old client may no longer be valid.

If upstream accepts additional optional fields, our current requests may remain valid.

### Responses we receive

An added optional field may be harmless if we preserve or deliberately tolerate unknown fields.

A new union member may break an exhaustive UI renderer even if the JSON remains valid.

### Generated code itself

A schema change can be compatible on the wire but change generated class names or imports.

Therefore report three axes:

```text
Request compatibility
Response compatibility
Generated-source compatibility
```

A fourth axis is behavioral compatibility, which static schema comparison cannot establish.

## 10. Stable generated names need their own design

Anonymous upstream types are common in the inspected API.

Do not derive names from:

- Absolute paths.
- Source line numbers.
- Raw TypeScript pretty-printed strings.
- Traversal order.

Proposed stable naming:

```text
Operation:
  GraphSearch

Request:
  GraphSearchRequest

Response:
  GraphSearchResponse

Anonymous nested object:
  GraphSearchResponseValue

Shared upstream type:
  PublicCoverage
```

For collisions:

```text
PublicCoverage__<stable short symbol fingerprint>
```

Use fingerprints only when necessary. Human-readable names matter for review.

### Preserve original wire names

Python can expose:

```text
source_id
```

while serializing:

```text
sourceId
```

But aliases must be generated systematically, and round-trip tests must verify them.

Do not let a formatting convention alter the actual wire contract.

## 11. Missing, null, and defaults deserve explicit model support

This is more subtle than it first appears.

These are different:

```json
{}
```

```json
{"file": null}
```

```json
{"file": {}}
```

The upstream `file?: FileFacet` permits absence, not necessarily explicit null.

For requests, generated serializers must omit absent optional fields rather than populate them with null.

### Rust-specific caution

Typify documents that optional properties can become `Option<T>`, while some optional fields with defaults can become defaulted values. It also notes limitations around complex unions and bounded numeric constraints.[^13_2]

Therefore, generated Rust types alone may not preserve every presence distinction or validation rule.

If Rust output is introduced, require the same acceptance corpus as Python and TypeScript.

## 12. Build outputs should be feature-selected

Do not generate every possible operation and merely hide it in the UI.

Define build profiles:

```text
read-only-core
    ├── graph search
    ├── node/neighbors/evidence
    ├── ontology guidance
    └── canvas inspection

read-only-export
    └── read-only-core + Markdown export

interactive-actions
    └── explicit canvas navigation/view selection

reviewed-writes
    └── deferred; not included by default
```

The selected profile controls:

- Dispatcher cases.
- Tool inventory.
- Manifest declarations.
- Client facade exports.
- Desktop actions.
- Tests.
- Package notices.


### Security advantage

A disabled write operation should ideally be absent from the generated dispatcher, not just hidden from the model.

That reduces accidental exposure.

## 13. One registry can compose tools, hooks, and Desktop capabilities

Generate an operation catalog:

```text
Operation ID
Upstream member path
Request validator
Response decoder
Required provider capability
Effect category
Exposure profile
Default product budget
```

Handwritten composition uses it:

```text
Agent tool factory
    └── operation catalog

Pre-tool policy
    └── operation catalog + trusted session state

Desktop route handlers
    └── operation catalog

Dispatcher generator
    └── admitted operation catalog

Documentation
    └── admitted operation catalog
```

This is the same broad pattern the inspected Hermes documentation generator uses: one machine-readable list drives both user documentation and enforcement-related tooling.

However, approval rules remain reviewed policy—not something inferred from operation names.

## 14. Proposed generated Python composition

Illustrative target:

```python
catalog = load_generated_catalog(profile="read-only-core")

provider = KplexProvider(
    transport=ObsidianCliTransport(configuration),
    catalog=catalog,
)

service = KnowledgeService(
    provider=provider,
    bindings=binding_store,
    policy=read_only_policy,
)

register_hermes_tools(ctx, catalog, service)
register_hermes_hooks(ctx, catalog, service.policy)
```

The generated catalog should contain data and safe references, not dynamic imports from arbitrary schema-provided module names.

### Keep effects centralized

Only the transport adapter launches the CLI.

Only the binding service resolves configured scope.

Only the plan builder creates executable requests.

Generated code should not scatter subprocess calls across every operation module.

## 15. Packaging needs a separate assembly manifest

The development hierarchy can be deep. The installable package should be compact.

Proposed assembly specification:

```yaml
package:
  id: hermes-kplex

include:
  - handwritten_python_runtime
  - generated_python_bindings
  - compiled_desktop_plugin
  - generated_dispatcher
  - generated_operation_catalog
  - bundled_skill
  - compatibility_manifest
  - required_notices

exclude:
  - vendor_development_tree
  - codegen_toolchain
  - native_test_vaults
  - local_validation_receipts
  - caches
```

A separate complete source archive can contain the vendor source and build materials where required.

### Verify the assembled package

Tests should inspect the archive, not only the source tree:

- Required entrypoints exist.
- Python imports resolve.
- Generated dispatcher is included.
- Desktop bundle contains only permitted external imports.
- No build-time absolute paths remain.
- No personal vault content or secrets are present.
- Selected operation profile matches the published manifest.


## 16. Avoid “generation on user install”

The intended lifecycle should be:

```text
Developer/vendor update
    → generate
    → review

Pull request
    → regenerate in check mode
    → compare

Release
    → reproduce checked outputs
    → assemble artifacts

User installation
    → install ready-to-run package
```

Do not require the end user to:

- Initialize the vendor submodule.
- Install Node.
- Run TypeScript extraction.
- Compile Rust.
- Download a schema from a moving branch.

That would undermine “easily pluggable.”

## 17. Proposed lock and provenance files

Use one reviewed contract lock:

```text
contracts/contract.lock.json
```

Proposed contents:

```json
{
  "formatVersion": 1,
  "vendor": {
    "repository": "HermesKPlex/kplex-Vendor",
    "commit": "<actual commit>"
  },
  "extraction": {
    "typescriptVersion": "<locked version>",
    "configurationFingerprint": "<hash>"
  },
  "contractFingerprint": "<hash>",
  "projectionFingerprint": "<hash>",
  "policyFingerprint": "<hash>",
  "generatorFingerprint": "<hash>",
  "profile": "read-only-core"
}
```

Do not populate these with invented values. The generation process should derive them.

Keep the lock human-reviewable and avoid noisy timestamps.

## 18. CI should verify both drift and omission

Beyond “generated files are unchanged,” add:

### Coverage check

Every admitted operation must have:

```text
source binding
request schema
response schema
capability rule
effect classification
dispatcher case
tool/API exposure decision
tests
```


### Orphan check

Fail for:

- Generated tool with no dispatcher case.
- Dispatcher case absent from exposure policy.
- Manifest entry without registration.
- Desktop action absent from the build profile.
- Unused generated schema accidentally shipped.


### Contract closure check

Every referenced type must resolve locally.

No remote schema fetching should occur during a release build.

### Determinism check

Generate from two clean directories and compare bytes.

This catches absolute paths, unstable ordering, and nondeterministic identifiers.

## 19. Suggested first compiler milestone

Do not implement all emitters at once.

### Milestone A — Static extraction report

Produce:

```text
public operations
parameter types
resolved result types
optional/defaulted arguments
referenced DTO closure
unsupported constructs
source provenance
```

No runtime bindings yet.

### Milestone B — Three read operations

Generate:

```text
search
node
evidence
```

Outputs:

```text
JSON schemas
Python models
fixed dispatcher
operation catalog
documentation table
```


### Milestone C — Actual wire conformance

Against a disposable original-app test vault:

- Obtain real responses.
- Validate them.
- Decode them.
- Preserve missing/null distinctions.
- Preserve unknown-field policy.
- Compare transport and model round trips.


### Milestone D — Feature-profile assembly

Build the read-only Hermes plugin package with generated inventory and archive verification.

Only then add canvases, exports, nested configuration members, or optional Rust output.

## 20. Recommended architectural decision

The structure should be deep in responsibility, not merely deep in folders:

```text
Vendor source
    ↓
Language-aware frontends
    ↓
Stable IR
    ↓
Explicit wire projection
    ↓
Feature capsules and exposure policy
    ↓
Generated bindings and registries
    ↓
Handwritten application services
    ↓
Hermes agent/Desktop composition
    ↓
Verified release package
```

The latest source inspection supports two refinements:

1. Use the GraphQL schema as a secondary projection audit, not the full contract authority.
2. Follow Hermes’s own generation pattern: a single machine-readable authority with deterministic rendering and a non-writing `--check` path.

The next concrete deliverable should be a contract-extraction specification and three-operation generation prototype—not another large handwritten client or a broad Rust implementation. The selected tooling can support the design, but custom projection rules and conformance testing remain essential, especially for callbacks, branded IDs, inferred returns, optional/null fields, and runtime-only validation.[^13_1][^13_2]

<div align="center">⁂</div>

[^13_1]: https://github.com/vega/ts-json-schema-generator

[^13_2]: https://github.com/oxidecomputer/typify

