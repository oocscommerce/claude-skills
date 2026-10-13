# Setting types reference

Declared under `settings` of a `components` entry (`OOCS\LynxComponent\Model\Registry`, global `etc/di.xml`).
The editor picks the field by `type` (`text` when unknown).

## Keys every setting takes

| Key | XML type | What it does |
|---|---|---|
| `label` | `string` | Field label |
| `type` | `string` | One of the types below |
| `default` | per type | Value of a new node, and what the server fills in when the node has none |
| `description` | `string` | Help text behind a **?** icon next to the label (top-level settings) |
| `depends` | `array` | Show only while other settings hold given values (top-level settings only) |

## Type table

| Type | Field | `default` | Prop value | Extra keys |
|---|---|---|---|---|
| `text` | One-line input | `string` | string | `placeholder` |
| `textarea` | Multi-line input | `string` | string | `placeholder`, `rows` (default 3), `mono` (`1`: monospace, keeps whitespace) |
| `richtext` | Rich text editor (bold, links, lists) | `string` (use CDATA for HTML) | HTML string | |
| `text_i18n` | `text` + **Translations** dialog | `string` | `{"default": "…", "<locale>": "…"}` | as `text` |
| `textarea_i18n` | `textarea` + **Translations** | `string` | same | as `textarea` |
| `richtext_i18n` | `richtext` + **Translations** | `string` | same, HTML texts | |
| `number` | Slider + number box, value clamped | `number` | number | `min` (0), `max` (500), `step` (1) |
| `toggle` | On/off switch | `boolean` | boolean | |
| `select` | Dropdown | `string` | option `value` | `options` or `source` |
| `color` | Swatch + picker, design palette first | `string` | `#rrggbb`, `rgba(…)` or empty | |
| `image` | Media gallery picker, upload, crop (jpg/png) | `string` | path under `pub/media` | |
| `url` | Link input | `string` | store path or absolute URL | `placeholder` (default "about-us or https://…") |
| `array` | Sortable list, each item expandable | `array` | array of objects | `fields`, `labelField` |

## text, textarea

```xml
<item name="note" xsi:type="array">
    <item name="label" xsi:type="string">Note</item>
    <item name="type" xsi:type="string">textarea</item>
    <item name="default" xsi:type="string"></item>
    <item name="placeholder" xsi:type="string">Optional</item>
    <item name="rows" xsi:type="number">4</item>
</item>
```

For values that are codes (SKUs, anchors, cookie names, CSS with `mono` `1`), not shopper-facing text.

## richtext

```xml
<item name="content" xsi:type="array">
    <item name="label" xsi:type="string">Content</item>
    <item name="type" xsi:type="string">richtext</item>
    <item name="default" xsi:type="string"><![CDATA[<p>Write something.</p>]]></item>
</item>
```

Render with `v-html` (Vue) / `dangerouslySetInnerHTML={{ __html: content }}` (React).

## text_i18n, textarea_i18n, richtext_i18n

```xml
<item name="title" xsi:type="array">
    <item name="label" xsi:type="string">Title</item>
    <item name="type" xsi:type="string">text_i18n</item>
    <item name="default" xsi:type="string">Mid-Season Sale</item>
</item>
```

Stored value (the merchant types locale codes of store views in the Translations dialog):

```json
{ "default": "Mid-Season Sale", "de_DE": "Sale zur Saisonmitte", "fr_FR": "Soldes de mi-saison" }
```

Provider: `$props['title'] = $this->localizedText->get($props['title']);` with
`OOCS\LynxComponent\Helper\LocalizedText`. `get(mixed $value): string` picks the current locale's entry, else
`default`, accepts a plain string too (server default, old layouts), returns `''` for empty, and passes the result
through `__()` so dictionary phrases translate. For `richtext_i18n` the dictionary key is the whole HTML string.
The component declares `title: string`.

## number

```xml
<item name="limit" xsi:type="array">
    <item name="label" xsi:type="string">Products</item>
    <item name="type" xsi:type="string">number</item>
    <item name="default" xsi:type="number">8</item>
    <item name="min" xsi:type="number">1</item>
    <item name="max" xsi:type="number">24</item>
    <item name="step" xsi:type="number">1</item>
</item>
```

## toggle

```xml
<item name="show_title" xsi:type="array">
    <item name="label" xsi:type="string">Show Title</item>
    <item name="type" xsi:type="string">toggle</item>
    <item name="default" xsi:type="boolean">true</item>
</item>
```

## select

Static options (item names are free; `value` is stored):

```xml
<item name="size" xsi:type="array">
    <item name="label" xsi:type="string">Size</item>
    <item name="type" xsi:type="string">select</item>
    <item name="default" xsi:type="string">md</item>
    <item name="options" xsi:type="array">
        <item name="sm" xsi:type="array">
            <item name="value" xsi:type="string">sm</item>
            <item name="label" xsi:type="string">Small</item>
        </item>
        <item name="md" xsi:type="array">
            <item name="value" xsi:type="string">md</item>
            <item name="label" xsi:type="string">Medium</item>
        </item>
    </item>
</item>
```

Another module can add an option by merging an item under the same path in its own global `etc/di.xml`.

Options from code: `source` = object implementing `OOCS\LynxComponent\Api\OptionSourceInterface`
(`toOptionArray(): array<int, array{value: string, label: string}>`), as `\Proxy`. `Registry::getComponents()`
replaces `source` by `options` (also inside `array` `fields`). A `Magento\Framework\Data\OptionSourceInterface` is
ignored.

```xml
<item name="source" xsi:type="object">Vendor\Module\Model\Config\Source\ProductAttribute\Proxy</item>
```

One class, different arguments: a virtual type and a proxy virtual type of it (a virtual type has no generated Proxy).

```xml
<virtualType name="VendorModuleImageAttributeSource" type="Vendor\Module\Model\Config\Source\ProductAttribute">
    <arguments>
        <argument name="images" xsi:type="boolean">true</argument>
    </arguments>
</virtualType>
<virtualType name="VendorModuleImageAttributeSourceProxy" type="Vendor\Module\Model\Config\Source\ProductAttribute\Proxy">
    <arguments>
        <argument name="instanceName" xsi:type="string">VendorModuleImageAttributeSource</argument>
    </arguments>
</virtualType>
```

Then `<item name="source" xsi:type="object">VendorModuleImageAttributeSourceProxy</item>`. (Built-in example:
`OOCSLynxCatalogProductImageAttributeSourceProxy` in `OOCS_LynxCatalog` `etc/di.xml`.)

## color

```xml
<item name="color" xsi:type="array">
    <item name="label" xsi:type="string">Text Color</item>
    <item name="description" xsi:type="string">Empty uses the default</item>
    <item name="type" xsi:type="string">color</item>
    <item name="default" xsi:type="string"></item>
</item>
```

Keep the default empty and fall back to a theme token class when empty, apply the value with an inline `style`.

## image

```xml
<item name="image" xsi:type="array">
    <item name="label" xsi:type="string">Image</item>
    <item name="type" xsi:type="string">image</item>
    <item name="default" xsi:type="string"></item>
</item>
```

Stored: `wysiwyg/sale/hero.jpg` (relative to `pub/media`). Provider:
`$props['image'] = $this->mediaUrl->url((string) $props['image']);` with
`OOCS\LynxThemeComponent\Helper\MediaUrl` (`url(string $value): string`: current store's media base URL + path;
empty, `/…` and `http(s)://…` unchanged).

## url

```xml
<item name="cta_url" xsi:type="array">
    <item name="label" xsi:type="string">Button URL</item>
    <item name="type" xsi:type="string">url</item>
    <item name="default" xsi:type="string">sale</item>
</item>
```

Stored: `about-us`, `/checkout/cart`, `#top`, `mailto:…`, `https://…`. Provider:
`$props['cta_url'] = $this->storeLink->path((string) $props['cta_url']);` with
`OOCS\LynxThemeComponent\Helper\StoreLink` (`path(string $value): string`: root-relative store path, store code
included when **Add Store Code to URLs** is on; empty, `#…` and any `scheme:` unchanged). Render with Inertia `Link`;
a plain `<a>` only for an internal link that must reload the page or open in a new tab.

## array

```xml
<item name="links" xsi:type="array">
    <item name="label" xsi:type="string">Links</item>
    <item name="type" xsi:type="string">array</item>
    <item name="labelField" xsi:type="string">label</item>
    <item name="default" xsi:type="array">
        <item name="0" xsi:type="array">
            <item name="label" xsi:type="string">About Us</item>
            <item name="url" xsi:type="string">about-us</item>
            <item name="new_tab" xsi:type="boolean">false</item>
        </item>
    </item>
    <item name="fields" xsi:type="array">
        <item name="label" xsi:type="array">
            <item name="label" xsi:type="string">Label</item>
            <item name="type" xsi:type="string">text_i18n</item>
        </item>
        <item name="url" xsi:type="array">
            <item name="label" xsi:type="string">URL</item>
            <item name="type" xsi:type="string">url</item>
        </item>
        <item name="new_tab" xsi:type="array">
            <item name="label" xsi:type="string">Open in New Tab</item>
            <item name="type" xsi:type="string">toggle</item>
        </item>
    </item>
</item>
```

- `labelField`: sub-field whose value titles each item (for an `_i18n` field, its `default`); otherwise "Item N".
- Sub-fields take any type, including nested `array` and `select` with `source`. They show only their `label`
  (no `description` tooltip) and ignore `depends`.
- A new item starts with: toggle `false`, number its `min` (or 0), nested array `[]`, anything else the sub-field's
  `default` (or `''`). A toggle's or number's `default` is not used for new items.
- The server fills only top-level defaults: a provider reads sub-fields with a fallback (`$link['url'] ?? ''`).
- Empty list default: `<item name="default" xsi:type="array"/>`.

Provider loop:

```php
foreach ($props['links'] as $i => $link) {
    $props['links'][$i]['label'] = $this->localizedText->get($link['label'] ?? '');
    $props['links'][$i]['url'] = $this->storeLink->path((string) ($link['url'] ?? ''));
}
```

Component type: `links: Array<{ label: string; url: string; new_tab: boolean }>`.

## depends

```xml
<item name="label" xsi:type="array">
    <item name="label" xsi:type="string">Label</item>
    <item name="type" xsi:type="string">text_i18n</item>
    <item name="default" xsi:type="string"></item>
    <item name="depends" xsi:type="array">
        <item name="show_label" xsi:type="string">true</item>
        <item name="display" xsi:type="array">
            <item name="card" xsi:type="string">card</item>
            <item name="list" xsi:type="string">list</item>
        </item>
    </item>
</item>
```

AND across keys, OR within an array, compared as strings. A hidden setting is removed from the node on save and is
rendered with its `default`. No need to repeat the condition in the `description`.

## Defaults: where they apply

| Where | What happens |
|---|---|
| Editor, new node | Saved with every top-level default (`_i18n` string defaults wrapped as `{default}`) |
| Editor, opening an old node | Missing top-level settings get their default in the panel; Save writes them |
| Server, every render | `LayoutResolver` adds `Registry::getDefaults(<name>)` under the stored props before the provider runs, nodes inserted by code included |
| Array items | Only the editor's new-item defaults above; nothing on the server |
