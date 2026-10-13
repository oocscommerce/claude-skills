# Layout XML on Lynx store views

Applies only to store views running a Lynx theme (or a child of one).

## Which files are read

1. Only files named `lynx_*.xml` are collected (modules' `view/frontend/layout/`, themes' `<Vendor_Module>/layout/`;
   `layout/override/` works as usual).
2. Every handle of the request gets a `lynx_` twin: a simple product page has `default`, `catalog_product_view`,
   `catalog_product_view_type_simple` and also `lynx_default`, `lynx_catalog_product_view`,
   `lynx_catalog_product_view_type_simple`.

| Standard file | Lynx file |
|---|---|
| `default.xml` | `lynx_default.xml` |
| `catalog_product_view.xml` | `lynx_catalog_product_view.xml` |
| `catalog_category_view.xml` | `lynx_catalog_category_view.xml` |
| `cms_index_index.xml` | `lynx_cms_index_index.xml` |
| `customer_account_index.xml` | `lynx_customer_account_index.xml` |
| `vendor_faq_index_index.xml` | `lynx_vendor_faq_index_index.xml` |

The page layout is always `lynx_column`. Do not put `layout="…"` on `<page>`: a page layout declared in a layout file
makes Inertia visits answer HTML instead of JSON. After removing one, clean the `layout` cache.

## Page skeleton

```text
<head>                         title, meta, links from page config and lynx_*.xml
  head.additional              block for extra <head> output
<body id="html-body">
  lynx.after.body.start        store config for the client, notices
  lynx.page.wrapper
    lynx.main.container
      lynx.inertia             mount point: page JSON, SSR markup when on
  lynx.before.body.end         scripts at the end of the body
```

## Examples

Title and meta:

```xml
<?xml version="1.0"?>
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <head>
        <title>Frequently Asked Questions</title>
        <meta name="description" content="Answers about orders, shipping and returns"/>
        <meta name="robots" content="INDEX,FOLLOW"/>
    </head>
</page>
```

Headless block that sets page config in `_prepareLayout()` and returns `''` from `_toHtml()` (pattern of
`OOCS\LynxCatalog\Block\CategoryMeta`, `OOCS\LynxCatalog\Block\ProductCacheTags`):

```xml
<referenceContainer name="lynx.main.container">
    <block class="Vendor\Module\Block\QuestionMeta" name="vendor.module.question.meta"/>
</referenceContainer>
```

Extra `<head>` markup:

```xml
<referenceBlock name="head.additional">
    <block class="Magento\Framework\View\Element\Template" name="vendor.module.schema"
           template="Vendor_Module::head/schema.phtml"/>
</referenceBlock>
```

Keep a page out of the FPC: `<update handle="lynx_no_cacheable"/>`.

Arguments of the mount point:

```xml
<referenceBlock name="lynx.inertia">
    <arguments>
        <argument name="ssr" xsi:type="boolean">false</argument>
    </arguments>
</referenceBlock>
```

(`ssr` can only switch SSR off; also `encrypt_history`, `clear_history`, `sri`.)

## Behavior

- Blocks declared in `lynx_*.xml` are created, their view models and plugins run, HTML is printed where placed.
- Block output sits outside the app: printed on a full page load and kept while the shopper navigates.
- On Inertia visits the layout is still built (`_prepareLayout()` runs), then everything except `lynx.inertia` is
  dropped; only the **title** and **body class** travel with the JSON. Meta, canonical, Open Graph are printed on full
  loads only (what crawlers receive).

## What does not work

| In layout XML | On Lynx | Instead |
|---|---|---|
| Blocks in `content`, `sidebar.main`, `product.info.main`… | Containers do not exist | A Lynx container, or a component |
| `<referenceBlock>`/`<move>` on Luma blocks (`product.info.price`, `catalog.leftnav`) | Never declared | A component, or a theme override of the Lynx component |
| `<head>` assets one page needs (`<script src>` in `lynx_catalog_product_view.xml`) | Loaded only when that page is the first one opened | Import the code from the component |
| Widgets via **Content › Widgets** layout updates | Skipped | A component |
| Blocks only in regular layout files, plugins/observers on their output (`toHtml`, `*_html_after`) | Never created, hooks never run | Declare in a `lynx_` file, or hook the data the provider reads |
| `{{baseUrl}}` placeholders in layout XML | Left literal | Build URLs in PHP |

## Porting an extension's `default.xml`

Copy only what belongs in `<head>`, page config or the body edges into `lynx_default.xml`; everything rendered inside
the page becomes a component. After changes: `bin/magento cache:clean layout full_page`.
