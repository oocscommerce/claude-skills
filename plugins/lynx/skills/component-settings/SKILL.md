---
name: component-settings
description: Declares the settings panel of a Lynx PageBuilder component - `@setting` fields in its etc/components/<Template>.graphqls - with the field type that picks the control (String/Text, Textarea, RichText, LocalizedText/Textarea/RichText, Int/Float, Boolean, enum select, `source` select, Color, Image, Url, list of `input` for arrays), `default`, description, `dependsOn`, `tab`, an `OptionSourceInterface` `source`, and the provider/component code each type needs. Use when a developer says "add a setting to my Lynx component", "translatable text field in PageBuilder", "image picker setting", "dropdown with attribute options", "repeater / list of links setting", "show this setting only when…", or "setting value is empty / not translated / image URL wrong". Use with the basics, component and provider skills.
---
# Lynx component settings

Produces the `@setting` fields of a component's `.graphqls` type: the fields the merchant fills in the editor's settings
panel. Each setting becomes a prop of the same name, with its `default` filled in by the server. Some types are
stored in a portable form and must be resolved by the component's provider.

## Before you start

- The component is declared (skill **component**) in `etc/components/<Template>.graphqls` of the module.
- Look at a built-in declaration with similar fields (`OOCS_LynxThemeComponent` `etc/components/`: `Banner.graphqls`,
  `Image.graphqls`, `LinkList.graphqls`, `Group.graphqls`; `OOCS_LynxCatalog`: `Attribute.graphqls` with a `source`).
- Pick field types with the table below; the full per-type reference is `reference/setting-types.md`.

| Field type | Editor control | `default` | Value in props | Provider needed |
|---|---|---|---|---|
| `String`, `Text` | One-line input | `""` | string | no |
| `Textarea` | Multi-line input | `""` | string | no |
| `RichText` | Rich text editor | `""` | HTML string | no |
| `LocalizedText` / `LocalizedTextarea` / `LocalizedRichText` | Base control + **Translations** dialog | `""` | `{default, <locale>…}` object | **yes**, `LocalizedText::get()` |
| `Int`, `Float` | Slider + number box | number | number | no |
| `Boolean` | Switch | `true` / `false` | boolean | no |
| enum | Dropdown, options in enum order | `"value"` | chosen option value | no |
| `String` + `source:` | Dropdown with options from code | `"value"` | chosen option value | no |
| `Color` | Picker with the design palette | `""` | `#rrggbb` / `rgba(…)` / empty | no |
| `Image` | Media gallery picker (upload, crop) | `""` | path under `pub/media` | **yes**, `MediaUrl::url()` |
| `Url` | Link input | `""` | store path or absolute URL | **yes**, `StoreLink::path()` |
| `[Input!]!` (list of an `input`) | Sortable list with sub-fields | `[]` | array of objects | when sub-fields need it |

A type no control shows (a data type, `StringMap`…) is an error when the definitions load.

## Steps

### 1. Declare each setting

A field with `@setting(label, default, min, max, step, tab, source, labelField, dependsOn, control)`; the string
above the field is the description (the **?** tooltip). Declare settings with `!` (a stored value is never null).
With many settings, `tab` groups them into collapsible rows, as the Theme tab does: a setting without one goes to
**General**, first; the other groups follow in order of first use. No setting with `tab` = one list.

```graphql
type VendorPicks @component(code: "vendor_picks", template: "Vendor_Module::Components/Picks", label: "Our Picks") {
  "Shown above the list"
  title: LocalizedText! @setting(label: "Title", default: "Our picks")
  limit: Int! @setting(label: "Products", default: 8, min: 1, max: 24)
  show_title: Boolean! @setting(label: "Show Title", default: true)
}
```

Give **every** setting a `default` of the field's kind: a number, `true`/`false`, `[]` for a list, a string for the
rest, localized types included (declare a plain string; the editor wraps it as `{default: …}`). An empty string
default is fine where empty means something ("no image", "theme color").

### 2. Select options: an enum or from code

Static: an `enum`, options in declaration order, label from `@label`. Value names are `SCREAMING_SNAKE_CASE` and store
lowercased (`MD` stores `"md"`); `@value` when the stored value differs (`16_9`). On/off is a `Boolean`, not a Yes/No
select. Prefix enum names with your vendor (names are global), or reuse a shared one from
`OOCS_LynxComponent/etc/components/enums.graphqls` (`Align`, `Justify`, `TextAlign`, `Rounded`…).

```graphql
enum VendorPicksDisplay {
  CARD @label(text: "Card")
  LIST @label(text: "List")
}
display: VendorPicksDisplay! @setting(label: "Display", default: "card")
```

From code: a `String` field with `source`, a class (or virtual type) implementing
**`OOCS\LynxComponent\Api\OptionSourceInterface`** (`toOptionArray(): array` of `['value' => …, 'label' => …]`),
no `\Proxy`:

```graphql
attribute: String! @setting(label: "Attribute", default: "sku", source: "Vendor\\Module\\Model\\Config\\Source\\ProductAttribute")
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

`Registry` creates the source and calls `toOptionArray()` only when it builds the editor's component list
(`getComponents()`), also for `source` on sub-fields of an array setting. To reuse one source class with different
constructor arguments, declare a `virtualType` in `etc/di.xml` and name it in `source`, as `OOCS_LynxCatalog` does
(`reference/setting-types.md#select`).

### 3. Show a setting only when others match: `dependsOn`

```graphql
label: LocalizedText! @setting(label: "Label", default: "", dependsOn: {show_label: "true", display: ["card", "list"]})
```

Several keys = AND; a list of values = any of them; values compare as strings (`Boolean` -> `"true"`/`"false"`). The
panel updates while editing. Saving **removes** a hidden setting from the node, so the server renders it with its
`default`. Not applied to array sub-fields.

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
- Provider details (class, labels, caching): skill **provider**.

### 5. Use the generated props in the component

Settings arrive in `Lynx.<TypeName>` (generated by `bin/magento lynx:static:deploy`), already resolved by the
provider; no default in the component:

```ts
const props = defineProps<Lynx.VendorPicks>()
```

Render `RichText` with `v-html`; links with Inertia's `Link` (Lynx sends off-origin links
to the browser itself). Keep a `Color` setting optional at the class level so the design applies until the merchant
picks one:

```vue
<h2 :class="color ? '' : 'text-base-content'" :style="color ? { color } : undefined">{{ title }}</h2>
```

## Rules

- **`default` in the `.graphqls` is the only default.** The editor saves every default on a new node and fills missing ones
  when a node is opened; `LayoutResolver` fills them on the server (`Registry::getDefaults()`) before the provider and
  component run, including nodes code inserts. So providers read `$props['limit']` directly and components give
  settings no default.
- **Server-side defaults are top-level only.** Sub-fields of array items are not filled by the server: read them
  with `?? ''` / `?? false` in the provider.
- **Localized settings require a provider.** Without one the component receives the raw object.
- **Use plain `String` / `Textarea` for codes** (SKUs, anchors, cookie names, CSS); `Localized*` for shopper-facing
  text; `Url` for links.
- **`Image` stores a path relative to `pub/media`** (`wysiwyg/sale/hero.jpg`) so layouts move between environments
  and stores; only the provider turns it into a URL. Never build media URLs in the component.
- **`Url` stores a store path or absolute URL**; the editor reports an error only for a value with a scheme that does
  not parse.
- **Implement Lynx's `OptionSourceInterface`**, not `Magento\Framework\Data\OptionSourceInterface`: the registry
  ignores the Magento one (the select shows no options).
- **Select values can grow** (another module can `extend enum`): map them with `Record<string, string>` and a
  fallback.
- **Do not name a setting `align`** unless you mean the Group alignment (None/Wide/Full) that DynamicNode consumes;
  use `text_align`, `justify`, `vertical_align`.
- **Setting keys are stored in layouts.** Renaming a key loses the saved value (the node falls back to the default).
- **Settings are what the MCP save tool accepts as props**: a node sent through `pagebuilder_save_layout` with a prop
  that is not a declared setting is refused. Provider-added props (`labels`, lists) are never settings.
- **Settings are merchant content**, not UI labels: their default text is fine in English in the `.graphqls`; UI strings the
  component itself shows still come from the provider's `labels`.

## Check

1. `bin/magento cache:clean lynx_component` (plus `bin/magento lynx:compile components` when the site runs compiled),
   then `bin/magento lynx:static:deploy` so `props.d.ts` has the new settings.
2. Editor: open the component's settings panel - every field shows the right control, defaults are filled, `?`
   tooltips show, `dependsOn` hides/shows live, a `source` select lists its options.
3. Save, reload the editor: values persist; a hidden dependent setting is gone from the node and the default renders.
4. Translations: add a locale entry (e.g. `de_DE`) to a localized field; the store view with that locale shows it,
   others show `default`.
5. Image and URL: storefront HTML has the full media URL and a store-relative link (with store code if enabled).
6. Array: add, reorder, delete items; new items start with sub-field defaults; `labelField` titles each item.
7. The storefront theme renders the props.

## Reference

- `reference/setting-types.md` - every field type with SDL, `@setting` arguments (`min`/`max`/`step`, `source`,
  `labelField`, `control`), enums, array `input` types, item defaults and value shapes.
- Skills: **basics**, **component**, **provider**.
- https://docs.oocscommerce.com/lynx/developer/component-settings
