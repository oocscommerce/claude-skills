# Component declaration reference

A component is a `type` with `@component` in `etc/components/<Template>.graphqls` (GraphQL SDL syntax, one file per
template, named after it). All `.graphqls` files of enabled modules are merged in `<sequence>` order; directives and
scalars: `OOCS_LynxComponent/etc/components/directives.graphqls`, shared enums (`Align`, `Justify`,
`TextAlign`, `Rounded`…): `enums.graphqls` beside it. Definitions are cached in cache type `lynx_component`.

## `@component` arguments

| Argument | Required | What it does |
|---|---|---|
| `code` | yes | Registered name (`vendor_highlight`). Stored on every node (`name`); the key deferred data is fetched under. Prefix with your vendor; never rename. |
| `template` | yes | `Vendor_Module::<path under src, no extension>`. Registry exposes it as `key` = `Vendor_Module/<path>`, also stored on every node (`component`), and resolved by each theme's component registry to the `.vue`/`.tsx` (theme overrides win). |
| `label` | yes | Name in the picker and outline; default node label. |
| `icon` | no | Phosphor Icons class, regular weight: `ph-image`, `ph-table`, `ph-tag-simple`, `ph-star`. Generic icon when omitted. |
| `group` | no | snake_case key. Picker heading (`product_list` -> "Product List", General last) and the value slots filter on. Omitted -> `general`. No list to register: a new key is a new group. |
| `zones` | no | Zones it can be added to (below). |
| `availableOn` | no | Page paths it can be added to (below). |
| `provider` | no | Class name string, backslashes escaped, no `\Proxy` (`"Vendor\\Module\\Provider\\X"`); implements `OOCS\LynxComponent\Api\ComponentProviderInterface`. Created only when a node needs it; never sent to the browser. See skill provider. |
| `hidden` | no | `true`: not in the picker (`Registry::getComponents()` skips it); still registered, so nodes your code inserts render and their provider runs. |

The string above the `type` is the description: tooltip in the picker, and behind the **?** icon of the settings panel.

`Registry` API (for code that inspects it): `getComponents(string $path = '')` (picker list, filtered by
`availableOn` when a path is given, `hidden` excluded), `getNames()`, `has(string $name)`,
`getDefaults(string $name)`, `getProvider(string $name)`.

## Fields

| Field | Declared as | Becomes |
|---|---|---|
| Setting | `title: LocalizedText! @setting(label: "Title", default: "")` | Settings panel control and a prop; see skill component-settings |
| Slot | `actions: Slot @slot(label: "Buttons", allow: ["text"], max: 1)` | Child area (below); not a prop |
| Provider data | `products: [VendorProduct!]! @optional` | A prop the provider adds |

A string above a field is its description (settings panel **?**, JSDoc in the generated types).

Provider data types:

| SDL | TypeScript |
|---|---|
| `String`, `Int`, `Float`, `Boolean` | `string`, `number`, `number`, `boolean` |
| enum | union of its values |
| object `type` declared in a `.graphqls` (no `@component`) | `Lynx.<TypeName>` interface |
| `[X!]!`, nestable (`[[X!]!]!`) | `X[]`, `X[][]` |
| `StringMap` | `Record<string, string>` (labels, urls) |
| `Json` | `any`; not filtered inside |
| `X!` / `X` / `@optional` | never null / `\| null` / may be missing (`?`), as a deferred provider's data before it arrives |

A type shared by several components goes in the file of the template that mainly uses it; others reuse it. Type and
enum names are global: prefix them with your vendor.

`@inherit(from: "VendorHighlight", except: ["tone"])` on a `type` copies the other type's fields first (but those in
`except`); fields declared here override them. Built-in use: Search Results and Advanced Search Results share the
Product Listing template and its provider data.

**Only declared fields reach the browser.** `OOCS\LynxComponent\Helper\Schema\PropsFilter` drops every key the code's
`.graphqls` does not declare, recursively through data types and lists, keeping `_key`; inside `Json` and `StringMap`
nothing is filtered. No casting, no filling. Developer mode logs
`Lynx props: <code>.<path> is not declared in etc/components/*.graphqls, dropped.` to `system.log`.

**Generated types.** `bin/magento lynx:static:deploy` (also `lynx:workspace:setup`, `lynx:static:test`) writes
`<theme>/lynx/dist/props.d.ts`: a global `Lynx` namespace with each enum as a string union, each data type as an
interface, and one interface per component named as its `type` (settings and data fields, plus `_key?`). The Vue theme lists the file in
`globalTypeFiles` of its `vite.config.js` so the compiler reads runtime props from it.

```ts
const props = defineProps<Lynx.VendorHighlight>()                                  // Vue
export default function Highlight(props: Lynx.VendorHighlight & { slots?: NodeSlots }) // React
defineProps<Lynx.VendorA | Lynx.VendorB>() // one template, several codes: fields of only one are optional
type Props = (Lynx.VendorA | Lynx.VendorB) & { slots?: NodeSlots } // React: brackets before &
```

## Zones

| Zone | Meaning |
|---|---|
| `header` | Shared header zone (one layout for every page) |
| `content` | The page's own content |
| `footer` | Shared footer zone |
| `backstage` | Components that render nothing (trackers, scripts). On every page including checkout, mounted inside a hidden element. Only components that list `backstage` can go there. |
| _(omitted)_ | header, content and footer; not backstage |

```graphql
type VendorHighlight @component(…, zones: ["content"])
```

## availableOn

Full action names; a trailing `*` matches by prefix.

```graphql
type VendorHighlight @component(…, availableOn: ["catalog_product_view*", "checkout_cart_index"])
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
| `layout` | Group, Row, Column, Tabs, Spacer, Divider | Product, Product Media, Product Detail, Product Listing, Checkout, Checkout Cart |
| `tab` | Tab | Tabs (only this group) |
| `text` | Heading, Paragraph, Button, Link List | Product, Product Media, Product Detail, Product Listing, Checkout, Checkout Cart |
| `media` | Image, Video, Banner, Carousel | Product, Product Media, Product Detail, Product Listing, Checkout, Checkout Cart |
| `form` | Contact Form, Newsletter Form | Product, Product Media, Product Detail, Product Listing, Checkout, Checkout Cart |
| `header_footer` | Header, Announcement Bar, Store Switcher, Currency Switcher, Social Links | |
| `utility` | Cookie Consent, Back To Top, Custom Script, Google Analytics 4 | |
| `product_list` | Product Carousel, Related / Up-sell / Cross-sell Products, Product Listing, Compare Products | Product, Product Media, Product Detail, Product Listing, Checkout, Checkout Cart |
| `search` | Search Results, Advanced Search Form, Advanced Search Results, Popular Search Terms | |
| `product_page` | Product, Product Media, Product Detail (every type), Description, Specifications, Reviews | Product: Images, Details |
| `product_attribute` | Product Attribute, Product Image Attribute, Price Alert, Stock Alert | Product: Images, Details; Product Media: Below Gallery; Product Detail: Below Price, Below Add to Cart |
| `category_attribute` | Category Attribute, Category Image Attribute | Product Listing: Below Title, Above Products, Below Products |
| `product_option` | Product Custom Options | Product Detail: Product Options (one per component) |
| `checkout` | Checkout Cart, Checkout, Success, Failure, PayPal Review | |
| `checkout_step` | Extra checkout steps | Checkout: Before Shipping Method, Before Payment (one per component) |
| `express_checkout` | Express payment buttons | Checkout and Checkout Cart: Express Checkout (one per component) |
| `account` | Sign-in, registration and password forms, Logout Success, Login as Customer, My Account, Orders and Returns | |
| `account_section` | Sections of My Account | My Account: Section (one per component) |
| `general` | Not Found, and every component without a group | Product, Product Media, Product Detail, Product Listing, Checkout, Checkout Cart |

Example: `group` `product_attribute` + `availableOn` `catalog_product_view*` can be dropped below the price of
Product Detail.

## Slots

```graphql
default: Slot @slot(label: "Content")
actions: Slot @slot(label: "Buttons", allow: ["text", "media"], max: 1)
```

| Argument | What it does |
|---|---|
| `label` | Slot name in the outline, and on the empty-slot placeholder in the preview ("Show all outlines") |
| `allow` | Accepted groups: the picker opened from this slot lists only these groups; dragging others in is blocked. Omitted: any component. |
| `max` | How many times each component (counted by registered name) may be in the slot; the picker greys out a full one as **Already added**. Omitted: no limit. |
| `name` | The stored slot name when it is no valid field name (`product_info_additional: Slot @slot(label: "Product Options", name: "product.info.additional")`). |

Rendering: Vue `<slot />` / `<slot name="x" />`, test `$slots.x`; React `slots?.default` / `slots?.x`, each already
rendered. An empty slot (no visible child) is not passed at all.

## align

Any block can declare it; DynamicNode removes it from the props and applies it: `data-align="wide|full"` on the Vue
root (attribute fallthrough), or a `<div data-align-wrap style="display:contents">` wrapper in React. Values other
than `wide`/`full` do nothing.

```graphql
"How far this block reaches inside a constrained Group: None keeps the content width like the blocks around it, Wide reaches the site width, Full runs edge to edge. No effect outside a Group"
align: Align! @setting(label: "Align", default: "none")
```

`Align` (None / Wide / Full) is in `OOCS_LynxComponent/etc/components/enums.graphqls`.

## Changing another module's component

`extend type` / `extend enum` in your own `etc/components/*.graphqls`, with the declaring module in your `<sequence>` (the
later module wins). Find the type name in that module's `etc/components/`.

```graphql
extend type VendorHighlight @component(label: "Callout", template: "Other_Module::Components/Callout") {
  tone: VendorHighlightTone! @setting(label: "Style", default: "outline") @replace  # change a field
  title: LocalizedText @remove                                                     # drop a field
  badge: String @optional                                                          # add a field
}
extend type VendorProduct { label: String }           # add a field to a data type
extend enum VendorHighlightTone { STRONG @label(text: "Strong") }  # add a select option
extend type VendorOldBlock @remove                    # unregister a component
```

Redeclaring an existing field without `@replace` is an error, as is `@replace`/`@remove` on a field that does not
exist, a duplicate type name or a duplicate `code`; every error names the file. Fill the data of a field you add with
an `afterResolve` plugin on the component's provider.

Let your group into a built-in slot (redeclare the slot with `@replace`, keeping its label and groups):

```graphql
extend type ProductDetailSimple {
  info_after: Slot @slot(label: "Below Add to Cart", allow: ["layout", "text", "media", "form", "product_list", "product_attribute", "general", "vendor_badges"]) @replace
}
```

Repeat for `ProductDetailConfigurable`, `ProductDetailBundle`, `ProductDetailGrouped`, `ProductDetailDownloadable` as
needed (read each slot's current `allow` in their `.graphqls`).

Add a slot to a built-in component: declare it the same way (it appears in the outline), then override the
component file in your theme so it renders the slot:

```graphql
extend type ProductDetailSimple {
  title_after: Slot @slot(label: "Below Title")
}
```

```vue
<!-- <theme>/OOCS_LynxCatalog/lynx/vue/src/Components/Product/Simple.vue, where the children go -->
<div v-if="$slots.title_after" class="grid gap-4"><slot name="title_after" /></div>
```

Override path: `<theme>/<Vendor_Module>/lynx/<vue|react>/src/<same path as in the module>`.

## How a node is rendered

1. `OOCS\LynxComponent\Model\LayoutResolver::resolve()` walks every zone and slot: drops hidden nodes and nodes whose
   code is no longer registered (logged as an error), numbers node ids per zone, fills each setting's `default` under
   the stored props, then calls the provider (or defers it) and filters the result through `PropsFilter`.
2. The theme's `DynamicNode` (`OOCS_LynxTheme`) looks up `item.component` in the generated component registry; no
   file for the theme -> renders nothing. It merges deferred data (`page.props[<name>]`) over the stored props,
   pulls out `align`, passes slots, and in the preview adds `data-lynx-id` / `data-lynx-label`.
3. React wraps each node in an error boundary and `Suspense`: a node that throws renders nothing and logs
   `[Lynx] node failed: <component>`.
4. In the editor preview an unregistered node is kept with `unavailable: true` and drawn as
   "<label> · Component not available"; the storefront never receives it.
