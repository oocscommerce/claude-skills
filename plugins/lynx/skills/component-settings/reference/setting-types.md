# Setting types reference

Settings are fields with `@setting` on a component's `type` in `etc/components/<Template>.graphqls`. The field type picks
the editor control; a type no control shows is an error when the definitions load.

## `@setting` arguments

| Argument | What it does |
|---|---|
| `label` | Field label (required) |
| `default` | Value of a new node, and what the server fills in when the node has none |
| `min`, `max`, `step` | Number range (`Int`/`Float`) |
| `tab` | Group of collapsible rows in the panel; settings without one go to **General**, first |
| `source` | `String` field: class or virtual type implementing `OOCS\LynxComponent\Api\OptionSourceInterface` |
| `labelField` | Array setting: the sub-field whose value titles each item |
| `dependsOn` | Show only while other settings hold given values (top-level settings only) |
| `control` | An editor control the field type does not pick (one added to the SettingsPanel) |

The string above a field is its description: help text behind a **?** icon next to the label (top-level settings).
Declare settings with `!`.

## Type table

| Field type | Control | `default` | Prop value | Extra arguments |
|---|---|---|---|---|
| `String`, `Text` | One-line input | `""` | string | |
| `Textarea` | Multi-line input | `""` | string | |
| `RichText` | Rich text editor (bold, links, lists) | `""` (HTML) | HTML string | |
| `LocalizedText` | `Text` + **Translations** dialog | `""` | `{"default": "…", "<locale>": "…"}` | |
| `LocalizedTextarea` | `Textarea` + **Translations** | `""` | same | |
| `LocalizedRichText` | `RichText` + **Translations** | `""` | same, HTML texts | |
| `Int`, `Float` | Slider + number box, value clamped | number | number | `min` (0), `max` (500), `step` (1) |
| `Boolean` | On/off switch | `true` / `false` | boolean | |
| enum | Dropdown | `"value"` | option value | |
| `String` + `source` | Dropdown, options from code | `"value"` | option value | `source` |
| `Color` | Swatch + picker, design palette first | `""` | `#rrggbb`, `rgba(…)` or empty | |
| `Image` | Media gallery picker, upload, crop (jpg/png) | `""` | path under `pub/media` | |
| `Url` | Link input | `""` | store path or absolute URL | |
| `[Input!]!` (list of an `input`) | Sortable list, each item expandable | `[]` | array of objects | `labelField` |

## String, Textarea

```graphql
"Shown under the button"
note: Textarea! @setting(label: "Note", default: "")
```

For values that are codes (SKUs, anchors, cookie names, CSS), not shopper-facing text.

## RichText

```graphql
content: RichText! @setting(label: "Content", default: "<p>Write something.</p>")
```

Render with `v-html` (Vue) / `dangerouslySetInnerHTML={{ __html: content }}` (React).

## LocalizedText, LocalizedTextarea, LocalizedRichText

```graphql
title: LocalizedText! @setting(label: "Title", default: "Mid-Season Sale")
```

Stored value (the merchant types locale codes of store views in the Translations dialog):

```json
{ "default": "Mid-Season Sale", "de_DE": "Sale zur Saisonmitte", "fr_FR": "Soldes de mi-saison" }
```

Provider: `$props['title'] = $this->localizedText->get($props['title']);` with
`OOCS\LynxComponent\Helper\LocalizedText`. `get(mixed $value): string` picks the current locale's entry, else
`default`, accepts a plain string too (server default, old layouts), returns `''` for empty, and passes the result
through `__()` so dictionary phrases translate. For `LocalizedRichText` the dictionary key is the whole HTML string.
The generated prop type is `string`.

## Int, Float

```graphql
limit: Int! @setting(label: "Products", default: 8, min: 1, max: 24, step: 1)
```

## Boolean

```graphql
show_title: Boolean! @setting(label: "Show Title", default: true)
```

## Select

Static options: an `enum`, in declaration order; `@label` is the option label. Value names are `SCREAMING_SNAKE_CASE`
(Magento's coding standard checks `.graphqls`) and store lowercased (`MD` stores `"md"`); `@value` when the stored
value is not the lowercased name. An on/off setting is a `Boolean`, never a Yes/No select:

```graphql
enum VendorSize {
  SM @label(text: "Small")
  MD @label(text: "Medium")
}
enum VendorRatio {
  AUTO @label(text: "Auto")
  V16_9 @label(text: "16:9") @value(text: "16_9")
}
size: VendorSize! @setting(label: "Size", default: "md")
ratio: VendorRatio! @setting(label: "Ratio", default: "auto")
```

Enum names are global: prefix yours, or reuse one from `OOCS_LynxComponent/etc/components/enums.graphqls` (`Align`,
`Justify`, `TextAlign`, `Rounded`…). Another module adds an option with
`extend enum VendorSize { LG @label(text: "Large") }`.

Options from code: `source` = class implementing `OOCS\LynxComponent\Api\OptionSourceInterface`
(`toOptionArray(): array<int, array{value: string, label: string}>`), no `\Proxy`: `Registry` creates it only when it
builds the editor's component list, and replaces `source` by `options` (also inside array sub-fields). A
`Magento\Framework\Data\OptionSourceInterface` is ignored.

```graphql
attribute: String! @setting(label: "Attribute", default: "sku", source: "Vendor\\Module\\Model\\Config\\Source\\ProductAttribute")
```

One class, different arguments: a virtual type in `etc/di.xml`, named in `source`.

```xml
<virtualType name="VendorModuleImageAttributeSource" type="Vendor\Module\Model\Config\Source\ProductAttribute">
    <arguments>
        <argument name="images" xsi:type="boolean">true</argument>
    </arguments>
</virtualType>
```

```graphql
attribute: String! @setting(label: "Attribute", default: "image", source: "VendorModuleImageAttributeSource")
```

(Built-in example: `ImageAttribute.graphqls` and the source virtual types in `OOCS_LynxCatalog` `etc/di.xml`.)

## Color

```graphql
"Empty uses the default"
color: Color! @setting(label: "Text Color", default: "")
```

Keep the default empty and fall back to a theme token class when empty, apply the value with an inline `style`.

## Image

```graphql
image: Image! @setting(label: "Image", default: "")
```

Stored: `wysiwyg/sale/hero.jpg` (relative to `pub/media`). Provider:
`$props['image'] = $this->mediaUrl->url((string) $props['image']);` with
`OOCS\LynxThemeComponent\Helper\MediaUrl` (`url(string $value): string`: current store's media base URL + path;
empty, `/…` and `http(s)://…` unchanged).

## Url

```graphql
cta_url: Url! @setting(label: "Button URL", default: "sale")
```

Stored: `about-us`, `/checkout/cart`, `#top`, `mailto:…`, `https://…`. Provider:
`$props['cta_url'] = $this->storeLink->path((string) $props['cta_url']);` with
`OOCS\LynxThemeComponent\Helper\StoreLink` (`path(string $value): string`: root-relative store path, store code
included when **Add Store Code to URLs** is on; empty, `#…` and any `scheme:` unchanged). Render with Inertia `Link`;
a plain `<a>` only for an internal link that must reload the page or open in a new tab.

## Array

A list of an `input` type; each `input` field is a sub-field with its own `@setting(label: …)`:

```graphql
input VendorLinksItem {
  label: LocalizedText! @setting(label: "Label")
  url: Url! @setting(label: "URL")
  new_tab: Boolean! @setting(label: "Open in New Tab")
}

links: [VendorLinksItem!]! @setting(label: "Links", default: [{label: "About Us", url: "about-us", new_tab: false}], labelField: "label")
```

- `labelField`: sub-field whose value titles each item (for a localized field, its `default`); otherwise "Item N".
- Sub-fields take any type, including nested lists and `String` with `source`. They show only their `label`
  (no **?** tooltip) and ignore `dependsOn`.
- A new item starts with: `Boolean` `false`, number its `min` (or 0), nested list `[]`, anything else the
  sub-field's `default` (or `''`). A `Boolean`'s or number's `default` is not used for new items.
- The server fills only top-level defaults: a provider reads sub-fields with a fallback (`$link['url'] ?? ''`).
- Empty list default: `default: []`.

Provider loop:

```php
foreach ($props['links'] as $i => $link) {
    $props['links'][$i]['label'] = $this->localizedText->get($link['label'] ?? '');
    $props['links'][$i]['url'] = $this->storeLink->path((string) ($link['url'] ?? ''));
}
```

The component receives an array of objects shaped like the `input`.

## dependsOn

```graphql
label: LocalizedText! @setting(label: "Label", default: "", dependsOn: {show_label: "true", display: ["card", "list"]})
```

AND across keys, OR within a list, compared as strings. A hidden setting is removed from the node on save and is
rendered with its `default`. No need to repeat the condition in the description.

## Defaults: where they apply

| Where | What happens |
|---|---|
| Editor, new node | Saved with every top-level default (localized string defaults wrapped as `{default}`) |
| Editor, opening an old node | Missing top-level settings get their default in the panel; Save writes them |
| Server, every render | `LayoutResolver` adds `Registry::getDefaults(<code>)` under the stored props before the provider runs, nodes inserted by code included |
| Array items | Only the editor's new-item defaults above; nothing on the server |
