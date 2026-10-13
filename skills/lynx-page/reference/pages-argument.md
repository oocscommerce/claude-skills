# `pages` argument of `OOCS\LynxPageBuilder\Model\PreviewPagebuilder`

Declared in the **global** `etc/di.xml` of the module that owns the page. The item name is the page code the editor
URL uses (`?page=<code>`); keep it vendor-prefixed (`vendor_faq`).

| Key | Type | Meaning |
|---|---|---|
| `label` | string | Name in the editor page switcher (translated with `__()` when shown; sample-data pages get the suffix `(Sample)`) |
| `path` | string | Where the layout is stored: the page's full action name (`faq_index_index`). Also the value used in components' `available_on` |
| `route` | string | Storefront route the preview opens, e.g. `faq/index/index`; `''` for the home page. Must be the full `front/controller/action` when `sample_data` is true |
| `sample` | string | Appends a sample entity of the previewed store view to `route`: `product` (adds `/id/<id>`, optionally filtered by `type`), `category` (a category that lists products), `search_term` (adds `/q/<popular term>`) |
| `type` | string | Product type for `sample` = `product` (`simple`, `configurable`, …) |
| `group` | string | Groups pages under one entry in the switcher (`Products`, `Account`); the group row sits where its first page sorts |
| `sort_order` | number | Position across all modules: home 10, 404 11, products 20, catalog 30–49, account 70, orders 75, checkout 80–89, other 90+ (group `Other`). Equal numbers keep declaration order; missing goes last. The lowest opens when the editor URL names no page |
| `sample_data` | boolean | `true` when the controller would send the editor away without context (signed-in customer, order just placed, reset token). The preview dispatches a sample action under the page's own name instead of the controller, and providers implementing `SampleProviderInterface` overlay sample props |

## Examples

Plain page:

```xml
<item name="vendor_faq" xsi:type="array">
    <item name="sort_order" xsi:type="number">65</item>
    <item name="label" xsi:type="string">FAQ</item>
    <item name="path" xsi:type="string">faq_index_index</item>
    <item name="route" xsi:type="string">faq/index/index</item>
</item>
```

Page that needs an entity (a product page of a custom type):

```xml
<item name="vendor_product_subscription" xsi:type="array">
    <item name="sort_order" xsi:type="number">27</item>
    <item name="label" xsi:type="string">Subscription product page</item>
    <item name="path" xsi:type="string">catalog_product_view_type_subscription</item>
    <item name="route" xsi:type="string">catalog/product/view</item>
    <item name="sample" xsi:type="string">product</item>
    <item name="type" xsi:type="string">subscription</item>
    <item name="group" xsi:type="string">Products</item>
</item>
```

Page that needs a signed-in customer:

```xml
<item name="vendor_account_points" xsi:type="array">
    <item name="sort_order" xsi:type="number">72</item>
    <item name="label" xsi:type="string">My points page</item>
    <item name="path" xsi:type="string">rewards_account_index</item>
    <item name="route" xsi:type="string">rewards/account/index</item>
    <item name="group" xsi:type="string">Account</item>
    <item name="sample_data" xsi:type="boolean">true</item>
</item>
```

## How the preview uses it

- Normal page: the preview frame opens `/lynx_pagebuilder/<route>?token=…` and forwards to the real controller with
  the browser's own session.
- `sample_data` page: the preview router matches the first three route parts against each `route`, dispatches
  `OOCS\LynxPageBuilder\Controller\Preview\Sample` under the page's module/controller/action (so the `lynx_<handle>`
  layout file still loads), and `LayoutResolver` calls `resolve()` then `resolveSample($resolved, $props)` on providers
  that implement `OOCS\LynxComponent\Api\SampleProviderInterface`. Other providers resolve normally.
- Endpoints in sample props should be `#`; components skip posting when a URL is `#`.
- In a preview request page props include `preview` (truthy); components that would navigate away on their own (a
  countdown redirect) check `usePage().props.preview`.
