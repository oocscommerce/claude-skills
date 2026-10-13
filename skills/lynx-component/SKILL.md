---
name: lynx-component
description: Creates and registers a Lynx PageBuilder component in a third-party Magento module - the `components` pool entry in global etc/di.xml (name, group, icon, zones, available_on, slots, settings, provider) plus the Vue `.vue` and React `.tsx` files the Lynx themes render. Use when a developer says "add a Lynx component", "new PageBuilder block", "make my module's widget available in the Lynx editor", "add a slot to a Lynx component", "let my component into the Product Detail slot", "component not showing in the picker", or "headless/tracker component for Backstage". Use with lynx-basics (module structure, styling, CLI); use lynx-component-settings for the settings panel and lynx-provider for the PHP data behind the props.
---
# Lynx PageBuilder component

Produces a component the merchant can add in the Lynx PageBuilder editor: one item in the `components` pool of
`OOCS\LynxComponent\Model\Registry` (global `etc/di.xml`) and one file per theme framework
(`view/frontend/lynx/vue/src/...vue`, `view/frontend/lynx/react/src/...tsx`). A component from a third-party module
works exactly like a built-in one.

## Before you start

- Read **lynx-basics** for the module skeleton, `<sequence>`/composer requirements, styling rules and CLI.
- Find which Lynx themes the store builds (`OOCS_LynxVue`, `OOCS_LynxReact`, or a child theme). Ship both files unless
  the developer targets one framework; a theme with no file for a component renders nothing for its nodes.
- Look for a built-in component close to what you need and imitate it. Entries live in the global `etc/di.xml` of
  `OOCS_LynxThemeComponent` (Heading, Image, Banner, Group, LinkList…) and `OOCS_LynxCatalog` (product blocks), under
  `vendor/oocscommerce/module-lynx-*/`. Files are in each
  module's `view/frontend/lynx/<vue|react>/src/Components/`.
- Decide: does it need server data, translated text, `_i18n`/`image`/`url` settings? Then it needs a provider
  (skill **lynx-provider**). Does it hold children? Then it declares slots.

## Steps

### 1. Module dependency

`etc/module.xml` sequences `OOCS_LynxComponent` (registry and provider contract). Add `OOCS_LynxThemeComponent` only if
the provider uses its `Helper\MediaUrl` / `Helper\StoreLink`, and the module that declares a built-in component whose
slots you extend (step 6). Mirror them in `composer.json` (`oocscommerce/module-lynx-component`,
`oocscommerce/module-lynx-theme-component`).

### 2. Register the component in global `etc/di.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="OOCS\LynxComponent\Model\Registry">
        <arguments>
            <argument name="components" xsi:type="array">
                <item name="vendor_highlight" xsi:type="array">
                    <item name="component" xsi:type="string">Vendor_Module::Components/Highlight</item>
                    <item name="provider" xsi:type="object">Vendor\Module\Provider\Highlight\Proxy</item>
                    <item name="label" xsi:type="string">Highlight</item>
                    <item name="description" xsi:type="string">A boxed title and text with room for buttons</item>
                    <item name="icon" xsi:type="string">ph-star</item>
                    <item name="group" xsi:type="string">text</item>
                    <item name="zones" xsi:type="array">
                        <item name="content" xsi:type="string">content</item>
                    </item>
                    <item name="settings" xsi:type="array">
                        <item name="title" xsi:type="array">
                            <item name="label" xsi:type="string">Title</item>
                            <item name="type" xsi:type="string">text_i18n</item>
                            <item name="default" xsi:type="string">Free shipping over $50</item>
                        </item>
                        <item name="tone" xsi:type="array">
                            <item name="label" xsi:type="string">Style</item>
                            <item name="type" xsi:type="string">select</item>
                            <item name="default" xsi:type="string">soft</item>
                            <item name="options" xsi:type="array">
                                <item name="soft" xsi:type="array"><item name="value" xsi:type="string">soft</item><item name="label" xsi:type="string">Soft</item></item>
                                <item name="outline" xsi:type="array"><item name="value" xsi:type="string">outline</item><item name="label" xsi:type="string">Outline</item></item>
                            </item>
                        </item>
                    </item>
                    <item name="slots" xsi:type="array">
                        <item name="actions" xsi:type="array">
                            <item name="label" xsi:type="string">Buttons</item>
                            <item name="allowed_groups" xsi:type="array">
                                <item name="text" xsi:type="string">text</item>
                            </item>
                        </item>
                    </item>
                </item>
            </argument>
        </arguments>
    </type>
</config>
```

Key points (full table in `reference/registry-keys.md`):

- **Item name** (`vendor_highlight`) is stored in every layout that uses the component. Prefix it with your vendor and
  never rename it.
- **`component`** is `Vendor_Module::<path under src without extension>`. The registry turns it into the key
  `Vendor_Module/Components/Highlight`, which each theme resolves to its `.vue` or `.tsx`. The key is also saved on
  every node, so moving the file later breaks existing nodes.
- **`icon`**: a Phosphor Icons class, regular weight (`ph-star`, `ph-image`, `ph-table`, `ph-tag-simple`). The editor
  ships the font.
- **`group`**: snake_case key; picker heading and what slots filter on. Omitted = `general`. Reuse a built-in group to
  get into the built-in slots that accept it (groups table in `reference/registry-keys.md`).
- **`zones`**: `header`, `content`, `footer`, `backstage`. Omitted = header, content and footer, never backstage.
- **`available_on`**: page paths (full action names), trailing `*` = prefix match (`catalog_product_view*`). Omitted =
  every page.
- **`settings`**: the settings panel; every setting becomes a prop. See skill **lynx-component-settings**.
- **`slots`**: named child areas, with optional `allowed_groups` and `max_per_component`.
- **`provider`**: always the `\Proxy` of the class (the pool is built on every request). See skill **lynx-provider**.
- **`hidden`**: `true` keeps it out of the picker; code can still place it and its provider still runs.

Copy the `align` setting (None / Wide / Full) from any built-in block if the component should be able to reach past
the content width inside a constrained Group (`reference/registry-keys.md#align`).

### 3. The Vue file - `view/frontend/lynx/vue/src/Components/Highlight.vue`

```vue
<script setup lang="ts">
// Settings arrive with their di.xml default filled in: required props, no defaults here
const props = withDefaults(
    defineProps<{
        title: string
        tone: string
        // Added by the provider, so it may get a default
        labels?: Record<string, string>
    }>(),
    { labels: () => ({}) }
)

// Options live in di.xml and can grow: map with a fallback
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

### 4. The React file - `view/frontend/lynx/react/src/Components/Highlight.tsx`

```tsx
type Props = {
    title: string
    tone: string
    labels?: Record<string, string>
    slots?: NodeSlots
}

// Options live in di.xml and can grow: map with a fallback
const TONE: Record<string, string> = {
    soft: 'bg-base-200',
    outline: 'border-(length:--border) border-base-300',
}

export default function Highlight({ title, tone, slots }: Props) {
    return (
        <div className={`w-full rounded-box p-6 ${TONE[tone] ?? TONE.soft}`}>
            {title && <h2 className="text-xl font-semibold text-base-content">{title}</h2>}
            {slots?.actions && <div className="mt-4 flex flex-wrap gap-3">{slots.actions}</div>}
        </div>
    )
}
```

`NodeSlots` and `LayoutItem` are global types declared by `OOCS_LynxTheme`; use them without importing.

Props contract (both frameworks):

| What | Vue | React |
|---|---|---|
| Settings | props named like the setting keys, defaults already filled | same |
| Default slot | `<slot />` | `slots?.default` |
| Named slot | `<slot name="actions" />` | `slots?.actions` (already rendered) |
| Empty slot test | `$slots.actions` | `slots?.actions` (an empty slot is not passed) |
| `align` | never a prop: becomes `data-align` on your root element | never a prop: DynamicNode wraps the node |
| Deferred/optional node | `_key` prop, data props `undefined` until fetched | same |

### 5. Editor preview behaviour

The editor previews the storefront in an iframe; there `usePage().props.preview` is `true`. Use it only to keep a
component visible or inert while the merchant edits, never for storefront logic:

```ts
import { usePage } from '@inertiajs/vue3'      // React: '@inertiajs/react'
// Show the link wherever it is placed, so the merchant can find it
const preview = !!usePage().props.preview
```

Built-in examples: `ProductAlert`, `StockAlertUnsubscribe` (`OOCS_LynxCatalog`), `LogoutSuccess`, `LoginAsCustomer`
(`OOCS_LynxCustomer`) skip redirects/side effects or keep an otherwise empty block visible in the preview.
In the preview, DynamicNode adds `data-lynx-id` / `data-lynx-label` to your root (Vue fallthrough) so the editor can
find and highlight the node; this is why one root element matters.

### 6. Optional: extend a built-in component's slots

Magento merges `di.xml` arrays by item name across modules, so write only the path to what you add, in your global
`etc/di.xml`, and sequence the module that declares the component:

```xml
<type name="OOCS\LynxComponent\Model\Registry">
    <arguments>
        <argument name="components" xsi:type="array">
            <item name="default_product_detail_simple" xsi:type="array">
                <item name="slots" xsi:type="array">
                    <item name="info_after" xsi:type="array">
                        <item name="allowed_groups" xsi:type="array">
                            <item name="vendor_badges" xsi:type="string">vendor_badges</item>
                        </item>
                    </item>
                </item>
            </item>
        </argument>
    </arguments>
</type>
```

Each product type has its own entry (`default_product_detail_simple`, `_configurable`, `_bundle`, `_grouped`,
`_downloadable`). Often simpler: give your component a group the slot already accepts. A brand-new slot on a built-in
component also needs a theme override of that component's file that renders the slot (see `reference/registry-keys.md`).

### 7. Headless component (tracker, script)

Draws nothing, goes in the `backstage` zone (declare `zones` → `backstage`). Vue: no `<template>` at all, and
`defineOptions({ render: () => null })` (an empty `<template>` breaks hydration of the nodes after it). React: return
`null`. Run the script in `onMounted` / `useEffect`. Example: `ReCaptcha.vue` in `OOCS_LynxReCaptcha`.
To show a placeholder in the editor instead, render one root `<div>` only while in the preview, as `GoogleTag.vue`
(`OOCS_LynxGoogleTag`) does.

## Rules

- **Global `etc/di.xml` only.** In `etc/frontend/di.xml` or `etc/adminhtml/di.xml` the `components` array replaces the
  whole global pool instead of merging: every other node loses its provider.
- **One root element** (Vue and React). The preview markers and `data-align` go on the root; with several roots (a
  `div` next to a `Teleport`) wrap everything in one `<div>`.
- **Settings are required props without defaults** in the component (`title: string`, no `?`, no `withDefaults` for
  them): the server fills in the di.xml `default`. Defaults in the component are only for provider-added props
  (`labels`, lists, `urls`).
- **Type settings as `string` / `number` / `boolean`**, not literal unions: another module can add select options.
  Map option values with `Record<string, string>` and a fallback.
- **No visible text in the component.** Button labels, placeholders, `aria-label`, messages come from the provider in a
  `labels` prop translated with `__()`; merchant-typed text comes through settings. Reuse Magento's own phrases.
- **Self-contained**: types, class maps and logic inside the `.vue`/`.tsx`; no shared helper/types files per folder.
- **Styling**: theme tokens (`bg-base-200`, `text-base-content`, `rounded-box`, `border-(length:--border)`,
  `max-w-site`…) and `l-` daisyUI controls (`l-btn l-btn-primary`, `l-input`, `l-skeleton`…). No hardcoded colors,
  radius, shadows, borders or widths; font sizes from the theme scale, never below `text-sm`. Write class names in
  full (maps, not string concatenation). Details in lynx-basics.
- **Content components do not frame themselves**: root `w-full`, no outer `max-w-*`, `mx-auto` or padding; the
  merchant wraps them in a Group.
- **Name alignment settings other than `align`** (`text_align`, `justify`, `vertical_align`): `align` is taken by
  DynamicNode.
- **SSR-safe**: no `window`/`document` in setup or render; use `onMounted` / `useEffect`.
- **Slot keys are permanent**: children are stored under them; renaming empties the slot on every saved page.
- **Zones, `available_on`, `allowed_groups`, `max_per_component` are add-time rules** of the editor (the MCP
  `pagebuilder_save_layout` tool also checks zones, pages, groups, settings and slots). A node already in a layout
  renders wherever that layout is used; header/footer are shared by every page.
- **`available_on` needs a page the editor knows.** A path that no `OOCS\LynxPageBuilder\Model\PreviewPagebuilder`
  `pages` entry declares never shows the component in the picker (declaring a page: skill **lynx-page**).
- **Module disabled or component removed**: on the storefront `LayoutResolver` drops every node whose name is no
  longer registered and logs `Lynx layout <path> (<zone>) holds node "<name>" of component "<key>", which is not
  registered; it is skipped.` The editor preview keeps the node as a "Component not available" placeholder, so the
  merchant can delete it. Nothing breaks; re-enabling the module brings the nodes back.

## Check

1. First time: `bin/magento setup:upgrade`, then `bin/magento lynx:workspace:setup <themes>` (e.g.
   `OOCS_LynxVue OOCS_LynxReact`) so the module enters the build.
2. `bin/magento lynx:static:deploy` (or `npm run build`) for new/changed `.vue`/`.tsx`; `bin/magento cache:clean` for
   `di.xml` changes (a di.xml-only change needs no rebuild). Production mode: also `setup:di:compile`.
3. Editor: the component appears in the picker under its group, only on allowed pages and zones; icon and description
   show; slots appear in the outline; `allowed_groups` filters the picker opened from the slot.
4. Preview: selecting the node highlights it (one root); changing Button color / Corners / Border width in design
   settings restyles it; `align` Wide/Full works inside a constrained Group.
5. Storefront on **each** Lynx theme you ship: renders, hydrates without console warnings, works with SSR on if the
   store uses it.
6. Disable the module on a copy: pages still render, the log shows the "not registered; it is skipped" line.

## Reference

- `reference/registry-keys.md` - every entry key, zones, `available_on` values, groups table, slots, `align`,
  extending built-in slots, how DynamicNode renders a node.
- Skills: **lynx-basics**, **lynx-component-settings**, **lynx-provider**, **lynx-page** (pages for `available_on`).
- https://docs.oocscommerce.com/lynx/developer/components
- https://docs.oocscommerce.com/lynx/developer/themes (overriding components)
