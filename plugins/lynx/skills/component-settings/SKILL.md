---
name: component-settings
description: Declares the settings panel of a Lynx PageBuilder component - the `settings` array of a `components` pool entry in global etc/di.xml - with the right type (text, textarea, richtext, text_i18n, textarea_i18n, richtext_i18n, number, toggle, select, color, image, url, array), `default`, `description`, `depends`, static `options` or an `OptionSourceInterface` `source`, array `fields`, and the provider/component code each type needs. Use when a developer says "add a setting to my Lynx component", "translatable text field in PageBuilder", "image picker setting", "dropdown with attribute options", "repeater / list of links setting", "show this setting only when…", or "setting value is empty / not translated / image URL wrong". Use with the basics, component and provider skills.
---
# Lynx component settings

Produces the `settings` part of a component entry: the fields the merchant fills in the editor's settings panel.
Each setting key becomes a prop of the same name, with its `default` filled in by the server. Some types are stored
in a portable form and must be resolved by the component's provider.

## Before you start

- The component entry exists (skill **component**) in the module's **global** `etc/di.xml`, under
  `OOCS\LynxComponent\Model\Registry` -> `components` -> `<name>` -> `settings`.
- Look at a built-in entry with similar fields (`OOCS_LynxThemeComponent` `etc/di.xml`: Banner, Image, LinkList,
  Group; `OOCS_LynxCatalog`: Product Attribute with a `source`).
- Pick types with the table below; the full per-type reference with XML is `reference/setting-types.md`.

| Type | Editor field | `default` XML type | Value in props | Provider needed |
|---|---|---|---|---|
| `text` | One-line input | `string` | string | no |
| `textarea` | Multi-line input | `string` | string | no |
| `richtext` | Rich text editor | `string` | HTML string | no |
| `text_i18n` / `textarea_i18n` / `richtext_i18n` | Base field + **Translations** dialog | `string` | `{default, <locale>…}` object | **yes**, `LocalizedText::get()` |
| `number` | Slider + number box | `number` | number | no |
| `toggle` | Switch | `boolean` | boolean | no |
| `select` | Dropdown | `string` | chosen option `value` | no |
| `color` | Picker with the design palette | `string` | `#rrggbb` / `rgba(…)` / empty | no |
| `image` | Media gallery picker (upload, crop) | `string` | path under `pub/media` | **yes**, `MediaUrl::url()` |
| `url` | Link input | `string` | store path or absolute URL | **yes**, `StoreLink::path()` |
| `array` | Sortable list with sub-fields | `array` | array of objects | when sub-fields need it |

An unknown `type` falls back to a plain text input.

## Steps

### 1. Declare each setting

Keys every setting takes: `label`, `type`, `default`, optional `description` (the **?** tooltip) and `depends`.

```xml
<item name="settings" xsi:type="array">
    <item name="title" xsi:type="array">
        <item name="label" xsi:type="string">Title</item>
        <item name="description" xsi:type="string">Shown above the list</item>
        <item name="type" xsi:type="string">text_i18n</item>
        <item name="default" xsi:type="string">Our picks</item>
    </item>
    <item name="limit" xsi:type="array">
        <item name="label" xsi:type="string">Products</item>
        <item name="type" xsi:type="string">number</item>
        <item name="default" xsi:type="number">8</item>
        <item name="min" xsi:type="number">1</item>
        <item name="max" xsi:type="number">24</item>
    </item>
    <item name="show_title" xsi:type="array">
        <item name="label" xsi:type="string">Show Title</item>
        <item name="type" xsi:type="string">toggle</item>
        <item name="default" xsi:type="boolean">true</item>
    </item>
</item>
```

Give **every** setting a `default`, with the XML type that matches: `number` for number, `boolean` for toggle,
`array` for array (`<item name="default" xsi:type="array"/>` for an empty list), `string` for everything else,
`_i18n` included (declare a plain string; the editor wraps it as `{default: …}`). An empty string default is fine
where empty means something ("no image", "theme color").

### 2. Select options: static or from code

Static: `options` -> one item per option with `value` and `label`. From code: replace `options` with `source`, an
object implementing **`OOCS\LynxComponent\Api\OptionSourceInterface`** (`toOptionArray(): array` of
`['value' => …, 'label' => …]`), referenced as its `\Proxy`:

```xml
<item name="attribute" xsi:type="array">
    <item name="label" xsi:type="string">Attribute</item>
    <item name="type" xsi:type="string">select</item>
    <item name="default" xsi:type="string">sku</item>
    <item name="source" xsi:type="object">Vendor\Module\Model\Config\Source\ProductAttribute\Proxy</item>
</item>
```

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Model\Config\Source;

use Magento\Catalog\Model\ResourceModel\Product\Attribute\CollectionFactory;
use OOCS\LynxComponent\Api\OptionSourceInterface;

class ProductAttribute implements OptionSourceInterface
{
    public function __construct(
        private readonly CollectionFactory $collectionFactory,
    ) {
    }

    /**
     * Attributes with a storefront label
     *
     * @return array<int, array{value: string, label: string}>
     */
    public function toOptionArray(): array
    {
        $collection = $this->collectionFactory->create()
            ->addFieldToFilter('frontend_label', ['notnull' => true])
            ->setOrder('frontend_label', 'ASC');

        $options = [];
        foreach ($collection as $attribute) {
            $options[] = [
                'value' => (string) $attribute->getAttributeCode(),
                'label' => (string) $attribute->getFrontendLabel(),
            ];
        }

        return $options;
    }
}
```

`Registry` calls `toOptionArray()` only when it builds the editor's component list (`getComponents()`), also for
`source` on sub-fields of an `array` setting. To reuse one source class with different constructor arguments, declare
a `virtualType` plus a proxy `virtualType` pointing at it, as `OOCS_LynxCatalog` does
(`OOCSLynxCatalogProductImageAttributeSourceProxy`, see `reference/setting-types.md#select`).

### 3. Show a setting only when others match: `depends`

```xml
<item name="depends" xsi:type="array">
    <item name="show_label" xsi:type="string">true</item>
    <item name="display" xsi:type="array">
        <item name="card" xsi:type="string">card</item>
        <item name="list" xsi:type="string">list</item>
    </item>
</item>
```

Several keys = AND; an array of values = any of them; values compare as strings (toggle -> `true`/`false`). The
panel updates while editing. Saving **removes** a hidden setting from the node, so the server renders it with its
`default`. Not applied to `array` sub-fields.

### 4. Resolve portable types in the provider

```php
$props['title'] = $this->localizedText->get($props['title']);        // OOCS\LynxComponent\Helper\LocalizedText
$props['image'] = $this->mediaUrl->url((string) $props['image']);     // OOCS\LynxThemeComponent\Helper\MediaUrl
$props['cta_url'] = $this->storeLink->path((string) $props['cta_url']); // OOCS\LynxThemeComponent\Helper\StoreLink
foreach ($props['links'] as $i => $link) {
    $props['links'][$i]['label'] = $this->localizedText->get($link['label'] ?? '');
    $props['links'][$i]['url'] = $this->storeLink->path((string) ($link['url'] ?? ''));
}
```

- `LocalizedText::get()` takes the store view locale's entry, else `default` (a plain string passes through), then
  `__()`: a default text a language pack already knows ("Sign In", "Contact Us") is translated with no merchant work.
- `MediaUrl::url()` prefixes the current store's media base URL; empty, `/…` and `http(s)://…` stay unchanged.
- `StoreLink::path()` returns a root-relative store path (with the store code when **Add Store Code to URLs** is on);
  empty, `#anchor`, and anything with a scheme (`mailto:`, `tel:`, `https:`) stay unchanged.
- `MediaUrl` / `StoreLink` live in `OOCS_LynxThemeComponent`: add it to `<sequence>` and `composer.json`.
- Provider details (class, `\Proxy`, labels, caching): skill **provider**.

### 5. Type the props in the component

```ts
// Vue
defineProps<{
    title: string            // after LocalizedText
    limit: number
    show_title: boolean
    tone: string             // select: string, not a literal union
    color: string
    image: string            // absolute URL after MediaUrl
    links: Array<{ label: string; url: string; new_tab: boolean }>
}>()
// React: the same shape as `type Props = { … }`
```

Render `richtext` with `v-html` / `dangerouslySetInnerHTML`; links with Inertia's `Link` (Lynx sends off-origin links
to the browser itself). Keep a `color` setting optional at the class level so the design applies until the merchant
picks one:

```vue
<h2 :class="color ? '' : 'text-base-content'" :style="color ? { color } : undefined">{{ title }}</h2>
```

## Rules

- **`default` in di.xml is the only default.** The editor saves every default on a new node and fills missing ones
  when a node is opened; `LayoutResolver` fills them on the server (`Registry::getDefaults()`) before the provider and
  component run, including nodes code inserts. So providers read `$props['limit']` directly and components declare
  settings as required props without defaults.
- **Server-side defaults are top-level only.** Sub-fields of `array` items are not filled by the server: read them
  with `?? ''` / `?? false` in the provider and type them defensively.
- **`_i18n` settings require a provider.** Without one the component receives the raw object.
- **Use plain `text` / `textarea` for codes** (SKUs, anchors, cookie names, CSS); `_i18n` for shopper-facing text;
  `url` for links.
- **`image` stores a path relative to `pub/media`** (`wysiwyg/sale/hero.jpg`) so layouts move between environments
  and stores; only the provider turns it into a URL. Never build media URLs in the component.
- **`url` stores a store path or absolute URL**; the editor reports an error only for a value with a scheme that does
  not parse.
- **Implement Lynx's `OptionSourceInterface`**, not `Magento\Framework\Data\OptionSourceInterface`: the registry
  ignores the Magento one (the select shows no options). Always reference it as `\Proxy`.
- **Select values can grow** (another module can add options by merging di.xml): type them `string` and map with a
  fallback.
- **Do not name a setting `align`** unless you mean the Group alignment (None/Wide/Full) that DynamicNode consumes;
  use `text_align`, `justify`, `vertical_align`.
- **Setting keys are stored in layouts.** Renaming a key loses the saved value (the node falls back to the default).
- **Settings are what the MCP save tool accepts as props**: a node sent through `pagebuilder_save_layout` with a prop
  that is not a declared setting is refused. Provider-added props (`labels`, lists) are never settings.
- **Settings are merchant content**, not UI labels: their default text is fine in English in di.xml; UI strings the
  component itself shows still come from the provider's `labels`.

## Check

1. `bin/magento cache:clean` (di.xml-only changes need no frontend build).
2. Editor: open the component's settings panel - every field shows the right control, defaults are filled, `?`
   tooltips show, `depends` hides/shows live, a `source` select lists its options.
3. Save, reload the editor: values persist; a hidden dependent setting is gone from the node and the default renders.
4. Translations: add a locale entry (e.g. `de_DE`) to an `_i18n` field; the store view with that locale shows it,
   others show `default`.
5. Image and URL: storefront HTML has the full media URL and a store-relative link (with store code if enabled).
6. Array: add, reorder, delete items; new items start with sub-field defaults; `labelField` titles each item.
7. Both themes (Vue and React) render the props.

## Reference

- `reference/setting-types.md` - every type with XML, extra keys (`placeholder`, `rows`, `mono`, `min`/`max`/`step`,
  `options`/`source`, `labelField`/`fields`), array item defaults and value shapes.
- Skills: **basics**, **component**, **provider**.
- https://docs.oocscommerce.com/lynx/developer/component-settings
