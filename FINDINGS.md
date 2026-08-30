# Findings learned

Consolidated from the QCObjects discovery labs. Ordered by how much pain each
item saved later.

## 1. Version line: use 2.4, not 2.5 (for a browser-only front-end app)

- **Working recipe:** `qcobjects@2.4.99` + `qcobjects-sdk@2.4.66`.
- The **2.5.x line is broken for plain browser apps**:
  - `index.d.ts` emits TypeScript errors (`TS2305` style) — `npx tsc` gripes.
  - The browser bundle embeds Node core references; the page throws
    `Process is not defined` / needs a `window.require` shim just to load.
- On 2.4-line browser bundles, the same lazy `require` of `node:process`,
  `node:fs`, `node:path`, `node:url` exists but in `hello-qcobjects` a benign
  `window.require` stub works (`hello-qcobjects/index.html`).

## 2. There are TWO different things: smart widget vs generic component

This confusion cost the most time.

- **Smart widget** — the custom element you write in HTML, e.g.
  `<greeting-component>`. It is a shell (a `customElements` subclass of
  `_ComponentWidget_`). **It renders nothing itself.** Its constructor creates a
  child *generic component* node (`<quick-component name="...">`) and copies all
  attributes (including `componentClass`) onto it.
- **Generic component** — a `<quick-component>` / `<component>` node. **This is
  the real renderer**, built by `_buildComponentFromElement_`.

So `<greeting-component name="greeting">` becomes:

```html
<greeting-component>
  <quick-component name="greeting" shadowed="true" loaded="true">
    <div class="shadowHost"></div>
  </quick-component>
</greeting-component>
```

## 3. How external `.tpl.html` templates load (the key mechanism)

A `tplsource="default"` component does **not** embed its markup in JS. It loads
a plain `.tpl.html` file over HTTP:

1. `_buildComponentFromElement_` computes
   `templateURI = componentsBasePath + name + tplextension`
   → `templates/components/greeting.tpl.html`.
2. `rebuild()` → `componentLoader` turns that into
   `component.url = basePath + templateURI`
   → `http://<host>/templates/components/greeting.tpl.html`.
3. QCObjects issues an **XHR GET**. Any static server works — the stock
   `qcobjects-http-server` and `python3 -m http.server` are equivalent here
   because the file is just served from the document root.
4. `feedComponent()` parses the response (`parseTemplate` /
   `DefaultTemplateHandler`) into `component.innerHTML`.
5. Because the component is `shadowed`, content lands in the **native shadow
   root** of `<div class="shadowHost">` (QCObjects.js:2677).

### Consequences that bit us

- **Missing `.tpl.html` = blank component.** If the XHR 404s, `rebuild()`
  rejects → `Something went wrong` on `componentLoader` (QCObjects.js ~3528).
  This was the #1 cause of a "blank component" we chased.
- **Content lives in a shadow root.** It's invisible to light-DOM tools:
  `--dump-dom` and naive queries cannot see it. Verify instead by reading
  `document.querySelector(".shadowHost").shadowRoot.innerHTML`.

## 4. Class resolution uses `componentClass`, NOT `name`

`_buildComponentFromElement_` picks the class from
`element.getAttribute("componentClass")`, **defaulting to the base `Component`**
(QCObjects.js ~2054-2065). The `name` attribute only drives the `templateURI` /
registration — it is **not** the class-selection key on the 2.4 line.

- **External pattern (base `Component`):** `tplsource` defaults to `"default"`,
  so the `.tpl.html` path is used; `componentClass` is **not required**.
- **Inline pattern** (`tplsource:"inline"` + `template`/`data` in the class):
  the real class must resolve, otherwise its fields are ignored. Pin it with the
  widget attribute:
  ```html
  <greeting-component id="greeting" name="greeting"
      componentClass="com.qcobjects.components.GreetingComponent"></greeting-component>
  ```

## 5. Nested/subcomponents use the same mechanism, one level deeper

The reference app's `nav` is **not** a top-level element — it's nested inside
`header.tpl.html`:

```html
<quick-component componentClass="Nav"></quick-component>
```

After a component's template is fetched, the builder scans the parsed body and
recursively builds nested generic nodes: `Component.__buildSubComponents__()` →
`_buildComponentsFromElements_(elements, component)` (QCObjects.js ~2605-2615),
with the same tag filter (`quick-component:not([loaded]),component:not([loaded])`)
and the same `_buildComponentFromElement_` → `ClassFactory(componentClass)`.

So `Nav` resolves from the **nested element's `componentClass="Nav"`**
attribute in the template markup — `name="nav"` on the class is only its
registration identifier, not a class-selection key.

## 6. Debugging the framework

- The logger hides most output: `logger.debugEnabled = false`,
  `logger.infoEnabled = true` (QCObjects.js:361). Enable the full trace:
  ```js
  window.logger.debugEnabled = true;
  ```
- Trace lines to look for:
  - `template source for <name> is default|inline` — which path it took.
  - `type for <name> is Component` — fell back to base class.
  - `LOADING COMPONENT DATA {...} FROM <url>` then `Data received "…"` — `.tpl.html`
    XHR succeeded.
  - `CREATING COMPONENT` → `COMPONENT <name> is shadowed` →
    `APPENDING Shadowed COMPONENT` → `was built successfully!` — success.
  - `Something wrong loading the component` — XHR failed (usually a 404).

## 7. Config that makes the external-template path work

Set these in the ESM entry (`src/js/init.js`):

```js
CONFIG.set("sourceType", "module");          // ESM
CONFIG.set("componentsBasePath", "templates/components/");
CONFIG.set("tplextension", "tpl.html");
```

And `config.json` sets the served root:
```json
{ "documentRoot": "$config(projectPath)browser/" }
```

## 8. Build tooling that worked

esbuild ESM bundle (no Parcel needed for a plain front-end):

```
npx -y esbuild@0.17.19 src/js/*.js --bundle --outdir=browser/js \
  --keep-names --global-name=global --sourcemap --splitting \
  --chunk-names=chunks/[name]-[hash] --format=esm --target=es2021
```

Plus an assets step that copies `src/index.html` and `src/templates/` into
`browser/`. `--splitting` + `--format=esm` yields dynamic chunks; the page loads
`<script type="module" src="js/init.js">`.

## 9. Skills CLI (companion meta-finding)

- `npx skills add <owner>/<repo>` clones the GitHub repo and discovers skills.
- Skills live in `skills/<name>/SKILL.md` (walked up to 3 levels deep).
- `npx skills add qcobjects-skills/scaffolding` works; the trailing
  "PromptScript: global install unsupported" line after a successful
  `-g -y` install is a harmless CLI quirk.

## 10. Class resolution: keep ONE class per namespace (2.4.x quirk)

`ClassFactory("a.b.C")` splits the last segment and looks the class up in
`Package("a.b")`. Its filter matches classes where
`__definition.__classType === "C"` **OR** `typeof classFactory === "function"
&& classFactory.name` (QCObjects.js:1141-1146). The second clause matches
EVERY named class in the package, and `.reverse()[0]` returns the **last
registered one** — so registering several classes in one namespace makes
`componentClass="a.b.C"` resolve to the wrong class for all but the last.

Working pattern (used by the UI kit):

```js
Package("com.qcobjects.components.card", [Card]);          // one namespace each
// element:
// <kit-card componentClass="com.qcobjects.components.card.Card">
```

(Before the fix, `ClassFactory("com.qcobjects.components.Card")` silently
returned the `Panel` class, so `{{title}}` never matched its `data`.)

## 11. `data` field vs `data-*` attributes on nested components

The base `Component` constructor merges the element's `data-*` attributes into
`data` (QCObjects.js:2220-2221):

```html
<quick-component componentClass="…StatChip" name="statchip" data-stat="14" data-label="smart widgets">
```

→ `data = { stat: "14", label: "smart widgets" }`.

But a modern subclass `data` field initializes **after** `super()` and
overwrites that merged object. So a class that wants attribute-driven values
(widgets nested in external templates, like the panel's chips) must **not**
declare its own `data` field:

```js
class StatChip extends Component {   // no data field — inherits attributes
  name = "statchip";
  tplsource = "default";
  tplextension = "tpl.html";
}
```

## 12. Events from shadow roots retarget — use `composedPath()`

A `document`-level click listener receives `event.target` as the **host**
element containing the shadow root, not the button inside it. Get the real
target with `event.composedPath()[0]`, then `target.getRootNode()` gives the
`ShadowRoot` for scoped queries (verified in `playground.js` of the UI kit).

## 13. Playground verification harness (extensible)

A generic CDP runner (`cdp_run.mjs`: url + expression file) evaluates arbitrary
inspector probes inside the page against the live shadow DOM — the fastest way
to smoke-test a QCObjects app without code changes. See
`APPENDIX-TOOLING.md` and the `qcobjects-ui-kit` repo.

## 14. Tailwind reaches the shadow root via runtime `@import` chains

My original assumption was wrong: page-level CSS cannot style content inside a
native shadow root, so I expected Tailwind classes to be useless there. But the
`qcobjects-web-2025` reference has a clean pattern that *does* work:

- Each component's `.tpl.html` starts with
  `<style>@import url("css/components/<w>.css")</style>`.
- Each compiled `src/css/components/<w>.css` (a thin SCSS entry, tracked as a
  `*.scss` source at `src/scss/components/*.scss`) is just:
  `@import url("./../tailwind.css"); @import url("./../theme.css");`
- `tailwind.css` is the full generated Tailwind build (`@tailwind base;
  @tailwind components; @tailwind utilities;`).

The browser resolves that nested URL chain **inside the shadow root**, so the
Tailwind utilities land exactly where the widget's own `<style>` lives. Build
pipeline: `sass src/scss:src/css` then
`tailwindcss -i src/css/tailwind-source.css -o src/css/tailwind.css`, then copy
`src/css` → `browser/css` and esbuild the JS. Verified: computed styles on the
shadow content report Tailwind tokens (e.g. `text-sky-400` → `rgb(56,189,248)`,
`bg-slate-800` → `rgb(30,41,59)`, `rounded-xl` → `12px`).

## 15. Per-widget interaction belongs in the component, not a global listener

The UI kit initially handled Counter/Tabset clicks from one `document`-level
delegated listener using the §12 `composedPath()` trick. Better: the widget
classes own their behavior. QCObjects gives you the hooks to do this cleanly:

- `done(standardResponse)` is called after the component builds and its shadow
  root is populated (verified: `this.shadowRoot` is live, `this.body` is the
  host `quick-component`).
- `this.hostElements(selector)` is the "body or shadow root depending on the
  kind" accessor — it returns `shadowRoot.subelements(selector)` for shadowed
  components and `body.subelements(selector)` otherwise, all dynamically
  (`subtags` is a convenience getter for the same). `subelements()` itself is a
  `querySelectorAll`-backed helper available on `Element`, `Document`, and
  `ShadowRoot`.
- Attaching listeners directly to the actual shadow elements (via
  `this.hostElements(...)`) sidesteps shadow-event retargeting entirely — no
  `composedPath()` needed.

Because interaction is now in the class, a component must declare
`componentClass="com.qcobjects.components.tabset.Tabset"` so its class (and thus
its `done()`) is actually instantiated rather than a base `Component` (see §17
for why the kit declares widgets as plain `<quick-component ...>` instead of the
`kit-*` smart-widget tags). Verified end-to-end: Counter `7→8→6`, Tabset
switches tab `a→b` and toggles the matching panel, with zero exceptions and zero
network failures.
## 16. Root/main component `done()` = "root stack built" (intentionally non-blocking)

When you need a single hook for "the page/app is up", make a root component
that owns the whole stack (the official `layout-basic` pattern) and use its
`done()`. In the UI kit that is the root
`<quick-component name="layout" componentClass="com.qcobjects.components.layout.Layout">`
whose `done()` publishes a `window.__layoutReady` snapshot and fires a
`window.__onQcReady` callback.

Important nuance, verified on QCObjects 2.4.99: the root's `done()` fires when
the root's own subtree construction **kicks off**, **not** when every nested
async build has finished. Measuring at the moment the root `done()` runs showed
`allLoaded:false, loaded:1/9` — only the root itself had reached
`loaded="true"` — while ~300ms later all widgets report `loaded="true"`. This
is **intended by the framework**: `done()` deliberately does not block
subcomponent loading (components "pull" their children asynchronously instead
of the parent awaiting them).

Consequences for real code:

- Treat root `done()` as the structural "the root's component-stack
  construction started/finished kicking off" hook, not a hard "all
  descendants loaded" barrier.
- To show the fully-loaded tree, re-dump a short settle later (the playground
  does a `setTimeout(dumpAll, 300)` after `__onQcReady`). Do not replace
  timers/settles with a strict "all loaded" claim asserted inside `done()`.
- The reliable per-component lifecycle signal is each component's own
  `done()` (fires after its template + shadow root are populated) — see §15.

If a hard "every component loaded" barrier is ever required, poll the known
leaves (`quick-component[loaded="true"]`) rather than assuming root `done()`
implies it in 2.4.x.

## 17. Declare components as `<quick-component componentClass="...">`, not `kit-*` smart-widget tags

A symptom reported in the UI kit: every widget 404'd on
`templates/components/kit-card.tpl.html` (also `kit-counter`, `kit-tabset`,
`kit-panel`) while the page still loaded.

Root cause: the widgets were declared with the **smart-widget custom-element
tags** (`<kit-card componentClass="..." name="card">`, registered via
`RegisterWidgets`). QCObjects' `_ComponentWidget_` transform is supposed to copy
the tag's attributes (including `componentClass`) onto an inner
`<quick-component name="...">`. But for widgets nested **inside the shadowed
`<kit-layout>`**, that attribute copy silently dropped `componentClass` (and
`id`/`name`) — the inner node was left as `name="kit-card" shadowed="true"` with
no `componentClass`. `_buildComponentFromElement_` then used `name="kit-card"`
for the template URI → `kit-card.tpl.html` → 404. Only `kit-layout` itself (the
shell, not nested) kept its attributes, which is why it still worked.

The robust fix (also the reference `qcobjects-web-2025` pattern) is to **skip
the smart-widget transform entirely** and declare each widget directly as a
generic component, with `componentClass` as a plain string attribute:

```html
<quick-component id="card" name="card" shadowed="true"
                 componentClass="com.qcobjects.components.card.Card"></quick-component>
```

- `componentClass` stays a string attribute on the real component element, so
  the class is resolved and the right `.tpl.html` is fetched (`card.tpl.html`).
- `shadowed="true"` is needed explicitly (the `kit-*` transform set it
  automatically) to keep rendering into a native shadow root.
- Drop `RegisterWidgets("kit-*", ...)` from `init.js` — no longer required.
- Manual inspection (playground): a `quick-component`'s rendered content lives
  under a `.shadowHost` div (a native `ShadowRoot`); interact via
  `this.hostElements(...)` in the component's `done()` (§15).

Verified after the fix: 7 templates load (layout, card, counter, tabset, panel +
2 statchips) with **0 failed resources**; Counter `7→8→6`, Tabset `a→b` with
panel toggling, Tailwind tokens present (count `rgb(251,146,60)` = `text-orange-400`).

Also corrected a playground bug: `dumpTree` labelled a named `quick-component`
with no `componentClass` as `"(none)"`; the correct fallback class is the base
`Component`, so the label should read `componentClass || "Component"`.
