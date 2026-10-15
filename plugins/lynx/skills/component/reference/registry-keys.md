# Component entry reference

The `components` argument of `OOCS\LynxComponent\Model\Registry`, declared in a module's **global** `etc/di.xml`.

## Entry keys

| Key | Required | XML type | What it does |
|---|---|---|---|
| item `name` | yes | - | Registered name (`vendor_highlight`). Stored on every node (`name`); the key deferred data is fetched under. Prefix with your vendor; never rename. |
| `component` | yes | `string` | `Vendor_Module::<path under src, no extension>`. Registry exposes it as `key` = `Vendor_Module/<path>`, also stored on every node (`component`), and resolved by each theme's component registry to the `.vue`/`.tsx` (theme overrides win). |
| `label` | yes | `string` | Name in the picker and outline; default node label. |
| `description` | no | `string` | Tooltip in the picker, and behind the **?** icon of the settings panel. |
| `icon` | no | `string` | Phosphor Icons class, regular weight: `ph-image`, `ph-table`, `ph-tag-simple`, `ph-star`. Generic icon when omitted. |
| `group` | no | `string` | snake_case key. Picker heading (`product_list` -> "Product List", General last) and the value slots filter on. Omitted -> `general`. No list to register: a new key is a new group. |
| `zones` | no | `array` | Zones it can be added to (below). |
| `available_on` | no | `array` | Page paths it can be added to (below). |
| `settings` | no | `array` | Settings panel; each key is a prop. See skill component-settings. |
| `slots` | no | `array` | Named child areas (below). |
| `provider` | no | `object` | `Vendor\Module\Provider\X\Proxy`, implements `OOCS\LynxComponent\Api\ComponentProviderInterface`. Removed before the config reaches the browser. See skill provider. |
| `hidden` | no | `boolean` | `true`: not in the picker (`Registry::getComponents()` skips it); still registered, so nodes your code inserts render and their provider runs. |

`Registry` API (for code that inspects it): `getComponents(string $path = '')` (picker list, filtered by
`available_on` when a path is given, `hidden` excluded), `getNames()`, `has(string $name)`,
`getDefaults(string $name)`, `getProviderMap()`.

## Zones

| Zone | Meaning |
|---|---|
| `header` | Shared header zone (one layout for every page) |
| `content` | The page's own content |
| `footer` | Shared footer zone |
| `backstage` | Components that render nothing (trackers, scripts). On every page including checkout, mounted inside a hidden element. Only components that list `backstage` can go there. |
| _(omitted)_ | header, content and footer; not backstage |

```xml
<item name="zones" xsi:type="array">
    <item name="content" xsi:type="string">content</item>
</item>
```

## available_on

Full action names; a trailing `*` matches by prefix. The item names are free, the values count.

```xml
<item name="available_on" xsi:type="array">
    <item name="product" xsi:type="string">catalog_product_view*</item>
    <item name="cart" xsi:type="string">checkout_cart_index</item>
</item>
```

| Value | Pages |
|---|---|
| `cms_index_index` | Home page |
| `catalog_category_view` | Category pages |
| `catalog_product_view*` | Every product page (each type has its own path `catalog_product_view_type_<type>`) |
| `catalog_product_view_type_simple` | Simple product pages only |
| `checkout_cart_index` | Cart |
| `checkout_index_index` | Checkout |
| `customer_account_login` | Sign-in page |
| _(omitted)_ | Every page |

The editor only knows the pages declared in the `pages` argument of `OOCS\LynxPageBuilder\Model\PreviewPagebuilder`
(each module's `etc/di.xml`). A component limited to a path no page declares never appears in the picker.

## Groups

| Group | Built-in components | Accepted by slots of |
|---|---|---|
| `layout` | Group, Row, Column, Tabs, Spacer, Divider | Product Detail, Product Listing, Checkout, Checkout Cart |
| `tab` | Tab | Tabs (only this group) |
| `text` | Heading, Paragraph, Button, Link List | Product Detail, Product Listing, Checkout, Checkout Cart |
| `media` | Image, Video, Banner, Carousel | Product Detail, Product Listing, Checkout, Checkout Cart |
| `form` | Contact Form, Newsletter Form | Product Detail, Product Listing, Checkout, Checkout Cart |
| `header_footer` | Header, Announcement Bar, Store Switcher, Currency Switcher, Social Links | |
| `utility` | Cookie Consent, Back To Top, Custom Script, Google Analytics 4 | |
| `product_list` | Product Carousel, Related / Up-sell / Cross-sell Products, Product Listing | Product Detail, Product Listing, Checkout, Checkout Cart |
| `search` | Search Results, Advanced Search Form, Advanced Search Results, Popular Search Terms | |
| `product_page` | Product Detail (every type), Description, Specifications, Reviews | |
| `product_attribute` | Product Attribute, Product Image Attribute | Product Detail: Below Gallery, Below Price, Below Add to Cart |
| `category_attribute` | Category Attribute, Category Image Attribute | Product Listing: Below Title, Above Products, Below Products |
| `product_option` | Product Custom Options | Product Detail: Product Options (one per component) |
| `checkout` | Checkout Cart, Checkout, Success, Failure, PayPal Review | |
| `checkout_step` | Extra checkout steps | Checkout: Before Shipping Method, Before Payment (one per component) |
| `express_checkout` | Express payment buttons | Checkout and Checkout Cart: Express Checkout (one per component) |
| `account` | Sign-in, registration and password forms, Logout Success, Login as Customer, My Account, Orders and Returns | |
| `account_section` | Sections of My Account | My Account: Section (one per component) |
| `general` | Not Found, and every component without a group | Product Detail, Product Listing, Checkout, Checkout Cart |

Example: `group` `product_attribute` + `available_on` `catalog_product_view*` can be dropped below the price of
Product Detail.

## Slots

```xml
<item name="slots" xsi:type="array">
    <item name="default" xsi:type="array">
        <item name="label" xsi:type="string">Content</item>
    </item>
    <item name="actions" xsi:type="array">
        <item name="label" xsi:type="string">Buttons</item>
        <item name="allowed_groups" xsi:type="array">
            <item name="text" xsi:type="string">text</item>
            <item name="media" xsi:type="string">media</item>
        </item>
        <item name="max_per_component" xsi:type="number">1</item>
    </item>
</item>
```

| Key | What it does |
|---|---|
| `label` | Slot name in the outline, and on the empty-slot placeholder in the preview ("Show all outlines") |
| `allowed_groups` | The picker opened from this slot lists only these groups; dragging others in is blocked. Omitted: any component. |
| `max_per_component` | How many times each component (counted by registered name) may be in the slot; the picker greys out a full one as **Already added**. Omitted: no limit. |

Rendering: Vue `<slot />` / `<slot name="x" />`, test `$slots.x`; React `slots?.default` / `slots?.x`, each already
rendered. An empty slot (no visible child) is not passed at all.

## align

Any block can declare it; DynamicNode removes it from the props and applies it: `data-align="wide|full"` on the Vue
root (attribute fallthrough), or a `<div data-align-wrap style="display:contents">` wrapper in React. Values other
than `wide`/`full` do nothing.

```xml
<item name="align" xsi:type="array">
    <item name="label" xsi:type="string">Align</item>
    <item name="description" xsi:type="string">How far this block reaches inside a constrained Group: None keeps the content width like the blocks around it, Wide reaches the site width, Full runs edge to edge. No effect outside a Group</item>
    <item name="type" xsi:type="string">select</item>
    <item name="default" xsi:type="string">none</item>
    <item name="options" xsi:type="array">
        <item name="none" xsi:type="array"><item name="value" xsi:type="string">none</item><item name="label" xsi:type="string">None</item></item>
        <item name="wide" xsi:type="array"><item name="value" xsi:type="string">wide</item><item name="label" xsi:type="string">Wide</item></item>
        <item name="full" xsi:type="array"><item name="value" xsi:type="string">full</item><item name="label" xsi:type="string">Full</item></item>
    </item>
</item>
```

## Extending built-in components

Magento merges `di.xml` arrays by item name, so another module's entry can be extended with only the path to the
change (global `etc/di.xml`, and the declaring module in your `<sequence>`).

Let your group into a built-in slot:

```xml
<item name="default_product_detail_simple" xsi:type="array">
    <item name="slots" xsi:type="array">
        <item name="info_after" xsi:type="array">
            <item name="allowed_groups" xsi:type="array">
                <item name="vendor_badges" xsi:type="string">vendor_badges</item>
            </item>
        </item>
    </item>
</item>
```

Repeat for `default_product_detail_configurable`, `_bundle`, `_grouped`, `_downloadable` as needed.

Add a slot to a built-in component: declare it the same way (it appears in the outline), then override the
component file in your theme so it renders the slot:

```xml
<item name="default_product_detail_simple" xsi:type="array">
    <item name="slots" xsi:type="array">
        <item name="title_after" xsi:type="array">
            <item name="label" xsi:type="string">Below Title</item>
        </item>
    </item>
</item>
```

```vue
<!-- <theme>/OOCS_LynxCatalog/lynx/vue/src/Components/Product/Simple.vue, where the children go -->
<div v-if="$slots.title_after" class="grid gap-4"><slot name="title_after" /></div>
```

Override path: `<theme>/<Vendor_Module>/lynx/<vue|react>/src/<same path as in the module>`.

## How a node is rendered

1. `OOCS\LynxComponent\Model\LayoutResolver::resolve()` walks every zone and slot: drops hidden nodes and nodes whose
   name is no longer registered (logged as an error), numbers node ids per zone, fills each setting's `default` under
   the stored props, then calls the provider (or defers it).
2. The theme's `DynamicNode` (`OOCS_LynxTheme`) looks up `item.component` in the generated component registry; no
   file for the theme -> renders nothing. It merges deferred data (`page.props[<name>]`) over the stored props,
   pulls out `align`, passes slots, and in the preview adds `data-lynx-id` / `data-lynx-label`.
3. React wraps each node in an error boundary and `Suspense`: a node that throws renders nothing and logs
   `[Lynx] node failed: <component>`.
4. In the editor preview an unregistered node is kept with `unavailable: true` and drawn as
   "<label> · Component not available"; the storefront never receives it.
