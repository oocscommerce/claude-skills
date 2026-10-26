---
name: component
description: Creates and registers a Lynx PageBuilder component in a third-party Magento module - the `etc/components/<Template>.graphqls` declaration (`@component` code, template, provider, group, icon, zones, availableOn; settings, slots, provider data fields, generated `Lynx.*` types) plus the `.vue` file the Lynx theme renders. Use when a developer says "add a Lynx component", "new PageBuilder block", "make my module's widget available in the Lynx editor", "add a slot to a Lynx component", "let my component into the Product Detail slot", "component not showing in the picker", or "headless/tracker component for Backstage". Use with the basics skill (module structure, styling, CLI); use the component-settings skill for the settings panel and the provider skill for the PHP data behind the props.
---
# Lynx PageBuilder component

Produces a component the merchant can add in the Lynx PageBuilder editor: one `type … @component(…)` in
`etc/components/<Template>.graphqls` and one `.vue` file
(`view/frontend/lynx/vue/src/...vue`). A component from a third-party module
works exactly like a built-in one.

## Before you start

- Read **basics** for the module skeleton, `<sequence>`/composer requirements, styling rules and CLI.
- Find which Lynx themes the store builds (`OOCS_LynxVue` or a child theme).
- Look for a built-in component close to what you need and imitate it. Declarations live in `etc/components/*.graphqls`
  of `OOCS_LynxThemeComponent` (Heading, Image, Banner, Group, LinkList…) and `OOCS_LynxCatalog` (product blocks),
  under `vendor/oocscommerce/module-lynx-*/`. Files are in each
  module's `view/frontend/lynx/vue/src/Components/`.
- Decide: does it need server data, translated text, `Localized*`/`Image`/`Url` settings? Then it needs a provider
  (skill **provider**). Does it hold children? Then it declares slots.

## Steps

### 1. Module dependency

`etc/module.xml` sequences `OOCS_LynxComponent` (registry and provider contract). Add `OOCS_LynxThemeComponent` only if
the provider uses its `Helper\MediaUrl` / `Helper\StoreLink`, and the module that declares a built-in component you
change (step 6). Mirror them in `composer.json` (`oocscommerce/module-lynx-component`,
`oocscommerce/module-lynx-theme-component`).

### 2. Declare the component in `etc/components/Highlight.graphqls`

One `.graphqls` file per template, named after it, in GraphQL SDL syntax (Lynx has no GraphQL API; the extension only
lets IDEs highlight it, e.g. VS Code `"files.associations": { "*.graphqls": "graphql" }`). Every `.graphqls` of the enabled
modules is merged in `<sequence>` order; shared directives and scalars are in
`OOCS_LynxComponent/etc/components/base.graphqls`, shared enums (`Align`, `Justify`…) in `enums.graphqls` beside it.

```graphql
enum VendorHighlightTone {
  SOFT @label(text: "Soft")
  OUTLINE @label(text: "Outline")
}

"A boxed title and text with room for buttons"
type VendorHighlight @component(
  code: "vendor_highlight"
  template: "Vendor_Module::Components/Highlight"
  provider: "Vendor\\Module\\Provider\\Highlight"
  label: "Highlight"
  group: "text"
  icon: "ph-star"
  zones: ["content"]
) {
  title: LocalizedText! @setting(label: "Title", default: "Free shipping over $50")
  tone: VendorHighlightTone! @setting(label: "Style", default: "soft")
  actions: Slot @slot(label: "Buttons", allow: ["text"])
  "Every string the component shows, translated by the provider"
  labels: StringMap! @optional
}
```

Key points (full reference in `reference/registry-keys.md`):

- **`code`** (`vendor_highlight`) is stored in every layout that uses the component. Prefix it with your vendor and
  never rename it.
- **Type and enum names are global** across all modules (a duplicate is an error): prefix them too (`VendorHighlight`).
- **`template`** is `Vendor_Module::<path under src without extension>`. The registry turns it into the key
  `Vendor_Module/Components/Highlight`, which each theme resolves to its `.vue`. The key is also saved on
  every node, so moving the file later breaks existing nodes.
- **`provider`**: the class name as a string, backslashes escaped, no `\Proxy` (the registry creates a provider only
  when a node needs it). See skill **provider**.
- **`icon`**: a Phosphor Icons class, regular weight (`ph-star`, `ph-image`, `ph-table`, `ph-tag-simple`). The editor
  ships the font.
- **`group`**: snake_case key; picker heading and what slots filter on. Omitted = `general`. Reuse a built-in group to
  get into the built-in slots that accept it (groups table in `reference/registry-keys.md`).
- **`zones`**: `header`, `content`, `footer`, `backstage`. Omitted = header, content and footer, never backstage.
- **`availableOn`**: page paths (full action names), trailing `*` = prefix match (`catalog_product_view*`). Omitted =
  every page.
- **`hidden: true`** keeps it out of the picker; code can still place it and its provider still runs.
- The string above the `type` is the description (picker tooltip, **?** in the settings panel).
- **Fields** are of three kinds: `@setting` (the settings panel, skill **component-settings**), `Slot` + `@slot`
  (named child areas: `allow` = accepted groups, `max` = times per component), and plain fields = the data the
  provider adds. **The server sends only declared fields**: every key the provider returns must be in the `.graphqls`.

Copy the `align` setting (`align: Align! @setting(label: "Align", default: "none")`) from any built-in block if the
component should be able to reach past the content width inside a constrained Group (`reference/registry-keys.md#align`).

### 3. The Vue file - `view/frontend/lynx/vue/src/Components/Highlight.vue`

Props types are generated from the `.graphqls` into the global `Lynx` namespace (`<theme>/lynx/dist/props.d.ts`), by
`bin/magento lynx:static:deploy`: `Lynx.<TypeName>` for a component's props, a data type or an enum, named as the `type` in the `.graphqls`.

```vue
<script setup lang="ts">
// Settings arrive with their .graphqls default filled in; only provider data (@optional) gets a default here
const props = withDefaults(defineProps<Lynx.VendorHighlight>(), { labels: () => ({}) })

// Another module can extend the enum: map with a fallback
const TONE: Record<string, string> = {
    soft: 'bg-base-200',
    outline: 'border-(length:--border) border-base-300',
}
</script>

<template>
    <div class="w-full rounded-box p-6" :class="TONE[props.tone] ?? TONE.soft">
        <h2 v-if="title" class="text-xl font-semibold text-base-content">{{ title }}</h2>
        <div v-if="$slots.actions" class="mt-4 flex flex-wrap gap-3">
            <slot name="actions" />
        </div>
    </div>
</template>
```

`Lynx`, `NodeSlots` and `LayoutItem` are global types; use them without importing. A template shared by several codes
lists them all: `Lynx.VendorA | Lynx.VendorB` (fields only one of them declares become optional).

Props contract:

| What | How |
|---|---|
| Settings | props named like the setting fields, defaults already filled |
| Provider data | `!` = never null, no `!` = may be `null`, `@optional` = may be missing (`?`) |
| Default slot | `<slot />` |
| Named slot | `<slot name="actions" />` |
| Empty slot test | `$slots.actions` |
| `align` | never a prop: becomes `data-align` on your root element |
| Deferred/optional node | `_key` prop, data props `undefined` until fetched |

### 4. Editor preview behaviour

The editor previews the storefront in an iframe; there `usePage().props.preview` is `true`. Use it only to keep a
component visible or inert while the merchant edits, never for storefront logic:

```ts
import { usePage } from '@inertiajs/vue3'
// Show the link wherever it is placed, so the merchant can find it
const preview = !!usePage().props.preview
```

Built-in examples: `ProductAlert`, `StockAlertUnsubscribe` (`OOCS_LynxProductAlert`), `LogoutSuccess`, `LoginAsCustomer`
(`OOCS_LynxCustomer`) skip redirects/side effects or keep an otherwise empty block visible in the preview.
In the preview, DynamicNode adds `data-lynx-id` / `data-lynx-label` to your root (Vue fallthrough) so the editor can
find and highlight the node; this is why one root element matters.

### 5. Optional: change a built-in component

`extend type` in your own `etc/components/*.graphqls`, with the declaring module in your `<sequence>` (the later module
wins). A field you redeclare needs `@replace`, so write the whole slot again with your group added:

```graphql
extend type ProductDetailSimple {
  info_after: Slot @slot(label: "Below Add to Cart", allow: ["layout", "text", "media", "form", "product_list", "product_attribute", "general", "vendor_badges"]) @replace
}
```

Each product type has its own type (`ProductDetailSimple`, `ProductDetailConfigurable`, `ProductDetailBundle`,
`ProductDetailGrouped`, `ProductDetailDownloadable`; read the `.graphqls` files under `etc/components/` of
`OOCS_LynxCatalog`). Often simpler: give your component a group the slot already accepts. A brand-new slot on a
built-in component also needs a theme override of that component's file that renders the slot; other changes
(settings, provider data, registration args, removing the component) are in `reference/registry-keys.md`.

### 6. Headless component (tracker, script)

Draws nothing, goes in the `backstage` zone (declare `zones` → `backstage`). No `<template>` at all, and
`defineOptions({ render: () => null })` (an empty `<template>` breaks hydration of the nodes after it). Run the script
in `onMounted`. Example: `ReCaptcha.vue` in `OOCS_LynxReCaptcha`.
To show a placeholder in the editor instead, render one root `<div>` only while in the preview, as `GoogleTag.vue`
(`OOCS_LynxGoogleTag`) does.

## Rules

- **Declare every prop the provider returns.** The server drops keys the `.graphqls` of that code does not declare
  (recursively, inside data types and lists; `Json`/`StringMap` are kept whole). Developer mode logs
  `Lynx props: <code>.<path> is not declared in etc/components/*.graphqls, dropped.` to `system.log`.
- **Do not hand-write props types.** Use `Lynx.<TypeName>` (the component's `type` name in the `.graphqls`) and the data types' `Lynx.<TypeName>`; change the `.graphqls`, not the
  component, when a prop changes.
- **One root element**. The preview markers and `data-align` go on the root; with several roots (a
  `div` next to a `Teleport`) wrap everything in one `<div>`.
- **Settings get no default in the component**: the server fills in the `.graphqls` `default`. `withDefaults` /
  destructuring defaults are only for provider data (`labels`, lists, `urls`).
- **Gated features are `@optional` fields.** The provider adds a URL/label only when the setting is on and the config
  allows it (no build-then-`unset`); the `.graphqls` declares them `@optional` and the component renders the part only
  when present (`add_to_wishlist: String! @optional` in the urls type, `v-if="urls.add_to_wishlist"`, as Product
  Detail).
- **Enums can grow** (`extend enum` in another module): map option values with `Record<string, string>` and a
  fallback, even though the generated type is a string union.
- **No visible text in the component.** Button labels, placeholders, `aria-label`, messages come from the provider in a
  `labels` prop translated with `__()`; merchant-typed text comes through settings. Reuse Magento's own phrases.
- **Self-contained**: types, class maps and logic inside the `.vue`; no shared helper/types files per folder.
- **Styling**: theme tokens (`bg-base-200`, `text-base-content`, `rounded-box`, `border-(length:--border)`,
  `max-w-site`…) and `l-` daisyUI controls (`l-btn l-btn-primary`, `l-input`, `l-skeleton`…). No hardcoded colors,
  radius, shadows, borders or widths; font sizes from the theme scale, never below `text-sm`. Write class names in
  full (maps, not string concatenation). Details in basics.
- **Content components do not frame themselves**: root `w-full`, no outer `max-w-*`, `mx-auto` or padding; the
  merchant wraps them in a Group.
- **Name alignment settings other than `align`** (`text_align`, `justify`, `vertical_align`): `align` is taken by
  DynamicNode.
- **SSR-safe**: no `window`/`document` in setup or render; use `onMounted`.
- **Slot keys are permanent**: children are stored under them; renaming empties the slot on every saved page.
- **Zones, `availableOn`, slot `allow` and `max` are add-time rules** of the editor (the MCP
  `pagebuilder_save_layout` tool also checks zones, pages, groups, settings and slots). A node already in a layout
  renders wherever that layout is used; header/footer are shared by every page.
- **`availableOn` needs a page the editor knows.** A path that no `OOCS\LynxPageBuilder\Model\PreviewPagebuilder`
  `pages` entry declares never shows the component in the picker (declaring a page: skill **page**).
- **Module disabled or component removed**: on the storefront `LayoutResolver` drops every node whose code is no
  longer registered and logs `Lynx layout <path> (<zone>) holds node "<name>" of component "<key>", which is not
  registered; it is skipped.` The editor preview keeps the node as a "Component not available" placeholder, so the
  merchant can delete it. Nothing breaks; re-enabling the module brings the nodes back.

## Check

1. First time: `bin/magento setup:upgrade`, then `bin/magento lynx:theme:setup <themes>` (e.g.
   `OOCS_LynxVue`) so the module enters the build.
2. After a `.graphqls` change: `bin/magento cache:clean lynx_component`, plus `bin/magento lynx:compile components` when
   the site runs compiled. `bin/magento lynx:static:deploy` rewrites `props.d.ts` and builds new/changed
   `.vue` (a plain `npm run build` does not regenerate the types). A child theme with its own
   `vite.config.js` keeps the `globalTypeFiles` line of the parent's.
3. Editor: the component appears in the picker under its group, only on allowed pages and zones; icon and description
   show; slots appear in the outline; the slot's `allow` filters the picker opened from the slot.
4. Preview: selecting the node highlights it (one root); changing Button color / Corners / Border width in design
   settings restyles it; `align` Wide/Full works inside a constrained Group.
5. Storefront on each Lynx theme the store builds: renders, hydrates without console warnings, works with SSR on if the
   store uses it.
6. Disable the module on a copy: pages still render, the log shows the "not registered; it is skipped" line.

## Reference

- `reference/registry-keys.md` - `@component` args, field kinds and types, zones, `availableOn` values, groups table,
  slots, `align`, `@inherit`, `extend type`, how a node is rendered.
- Skills: **basics**, **component-settings**, **provider**, **page** (pages for `availableOn`).
- https://docs.oocscommerce.com/lynx/developer/components
- https://docs.oocscommerce.com/lynx/developer/themes (overriding components)
