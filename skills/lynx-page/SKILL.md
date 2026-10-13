---
name: lynx-page
description: Adds a storefront page or route to a Magento module on a Lynx store - the controller (either one that declares no page so PageBuilder builds it from the design's layout, or one that returns its own Inertia page component via InertiaService::setPage), the `lynx_<handle>.xml` layout file (title, robots, lynx_no_cacheable, ssr / encrypt_history flags), the editor entry in the `pages` argument of OOCS\LynxPageBuilder\Model\PreviewPagebuilder, and a data patch that ships a starting layout. Also covers shared props, deferred/optional props, partial reloads, Link vs plain anchor navigation, page title and full page cache. Use when a developer says "add a page to Lynx", "new route/controller on the Lynx storefront", "make my page editable in PageBuilder", "page not in the editor page switcher", "Inertia page component", "share a prop with every page", "defer a prop", "install a default layout for my page". Use with lynx-basics.
---
# Lynx storefront page

Produces a new storefront route on a Lynx store view: a Magento controller, an optional `lynx_<handle>.xml`, and either
a PageBuilder-built page (merchant-editable, with the design's header/footer) or a page component of your own
(standalone screen). Editable pages also get an editor entry and, optionally, a starting layout.

## Before you start

- Read **lynx-basics** (module skeleton, `<sequence>`, styling, CLI). Components placed on the page are made with
  **lynx-component**; their data comes from **lynx-provider**.
- Pick the kind of page:

  | | Layout-driven (default) | Controller-declared |
  |---|---|---|
  | Controller | returns a result page, declares nothing | calls `InertiaService::setPage()` |
  | Content | layout stored under the full action name | your `Pages/<Name>` component |
  | Header, footer, backstage | from the design | not sent |
  | Merchant-editable | yes | no |
  | Data | component providers | controller props |

  Use layout-driven for anything shoppers browse. Controller-declared suits standalone screens (password prompt,
  a single-purpose form) that stand outside the store frame.
- Work out the **full action name**: `<route id>_<controller>_<action>`, lowercase (route id `faq`, `Controller/Index/Index.php`
  → `faq_index_index`). It is the layout handle, the PageBuilder `path`, and the `lynx_` file name.
- Look at a built-in page to imitate: `vendor/oocscommerce/module-lynx-customer/etc/di.xml` (pages + `available_on`),
  `.../Setup/Patch/Data/InstallCustomerLayouts.php`, `.../view/frontend/layout/lynx_*.xml`.

## Steps

Example: an FAQ page at `/faq` in `Vendor_Faq`.

### 1. Module dependencies

`etc/module.xml` `<sequence>`: `OOCS_LynxPageBuilder` (pages argument, layout patch), plus `OOCS_LynxInertia` when the
controller calls `InertiaService`. `composer.json`: `oocscommerce/module-lynx-page-builder`,
`oocscommerce/module-lynx-inertia`.

### 2. Route and controller

`etc/frontend/routes.xml`:

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:App/etc/routes.xsd">
    <router id="standard">
        <route id="faq" frontName="faq">
            <module name="Vendor_Faq"/>
        </route>
    </router>
</config>
```

**Layout-driven** `Controller/Index/Index.php` - declare nothing; Lynx's `ResolveLayoutPage` plugin builds page
component `OOCS_LynxTheme::Default` with a `layout` prop (content zone stored under `faq_index_index`, plus header,
footer, backstage) when no page was declared:

```php
<?php
declare(strict_types=1);

namespace Vendor\Faq\Controller\Index;

use Magento\Framework\App\Action\HttpGetActionInterface;
use Magento\Framework\View\Result\Page;
use Magento\Framework\View\Result\PageFactory;

class Index implements HttpGetActionInterface
{
    /**
     * @param PageFactory $pageFactory
     */
    public function __construct(
        private readonly PageFactory $pageFactory,
    ) {
    }

    /**
     * Result page; PageBuilder supplies the content
     *
     * @return Page
     */
    public function execute(): Page
    {
        $page = $this->pageFactory->create();
        $page->getConfig()->getTitle()->set(__('Frequently Asked Questions'));

        return $page;
    }
}
```

Until a design has a layout for the path, the page shows the header and footer with an empty content area.

**Controller-declared** - call `setPage()` and still return a result page:

```php
$this->inertiaService->setPage(
    'Vendor_Faq::Ask',                       // → Vendor_Faq/Pages/Ask
    ['heading' => __('Ask a question'), 'topics' => $this->getTopics()],
    $this->request->getRequestUri()
);

return $this->pageFactory->create();
```

Inject `OOCS\LynxInertia\Service\InertiaService`, `PageFactory`, `Magento\Framework\App\RequestInterface`.
`Vendor_Faq::Ask` resolves to `Pages/Ask` in the module's `view/frontend/lynx/<vue|react>/src/`; a name without `::`
resolves to `Pages/<Name>` in the theme's own source. Props must be JSON-serializable; translate strings in PHP with
`__()`. Page files (minimal, both frameworks) are in [reference/page-component.md](reference/page-component.md).

### 3. Layout file `view/frontend/layout/lynx_faq_index_index.xml` (optional)

Only `lynx_*.xml` files are read on Lynx store views; every request handle gets a `lynx_` twin. Use it for title,
meta, cache and mount-point flags:

```xml
<?xml version="1.0"?>
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <head>
        <title>Frequently Asked Questions</title>
        <meta name="description" content="Answers about orders, shipping and returns"/>
    </head>
</page>
```

Private page (personal data): add `<update handle="lynx_no_cacheable"/>`, `<meta name="robots"
content="NOINDEX,NOFOLLOW"/>` and arguments on block `lynx.inertia`:

```xml
<body>
    <referenceBlock name="lynx.inertia">
        <arguments>
            <argument name="ssr" xsi:type="boolean">false</argument>
            <argument name="encrypt_history" xsi:type="boolean">true</argument>
        </arguments>
    </referenceBlock>
</body>
```

| `lynx.inertia` argument | Effect |
|---|---|
| `ssr` = `false` | never server-render this page, even if its route is in the SSR list |
| `encrypt_history` = `true` | encrypt the page state in browser history (HTTPS); pages with personal data |
| `clear_history` = `true` | drop earlier encrypted history entries; the page after sign-out |
| `sri` = `true` | Subresource Integrity hashes + import map for the bundle |

A controller can call `$inertiaService->encryptHistory()` / `clearHistory()` instead.

### 4. Make the page editable (layout-driven only)

Global `etc/di.xml` (the admin editor and the storefront preview router both read it):

```xml
<type name="OOCS\LynxPageBuilder\Model\PreviewPagebuilder">
    <arguments>
        <argument name="pages" xsi:type="array">
            <item name="vendor_faq" xsi:type="array">
                <item name="sort_order" xsi:type="number">65</item>
                <item name="label" xsi:type="string">FAQ</item>
                <item name="path" xsi:type="string">faq_index_index</item>
                <item name="route" xsi:type="string">faq/index/index</item>
            </item>
        </argument>
    </arguments>
</type>
```

Keys (`label`, `path`, `route`, `sample`, `type`, `group`, `sort_order`, `sample_data`) are explained in
[reference/pages-argument.md](reference/pages-argument.md). For components that belong only to this page, add
`faq_index_index` to their `available_on` (see **lynx-component**).

Use `sample_data` = `true` when the real controller would redirect the editor away (needs a signed-in customer, a
just-placed order, a token). The preview then skips the controller and providers implementing
`OOCS\LynxComponent\Api\SampleProviderInterface` supply sample props (see **lynx-provider**, **lynx-account-tab**).

### 5. Ship a starting layout (optional)

A data patch writes a row of `lynx_pagebuilder_layout` for the default design (`Design::DEFAULT_ID`, read-only in the
editor; merchant designs inherit every path they have not saved). Insert only, so an existing row is never replaced.
Full patch and node shape: [reference/layout-patch.md](reference/layout-patch.md).

- Depend on `OOCS\LynxPageBuilder\Setup\Patch\Data\CreateDefaultDesign`.
- Every path a patch installs must also have a `pages` entry, or nobody can open it in the editor.
- Changing the starting layout later: a patch runs once, so add a **new** patch class; it still only reaches stores
  whose designs have not saved that path.

### 6. Shared, deferred and optional props

- **Shared** (every page of the request): `before` plugin on `InertiaService::resolvePage()` in
  `etc/frontend/di.xml`, calling `$subject->share('key', $value)`. Lynx already shares `auth.isLoggedIn`, `title`,
  `bodyClass`. Shared values are cached with the page in FPC: share only what is the same for everyone in the cache
  variation; per-customer data belongs in customer-data sections (see **lynx-forms-analytics**).
- **Deferred** (controller-declared pages): `$this->inertiaService->defer(fn () => ..., 'group')` - sent right after the
  page appears, one request per group. **Optional**: `$this->inertiaService->optional(fn () => ...)` - only a partial
  reload naming it resolves it.
- **Partial reload**: `router.reload({ only: ['archive'] })`. On layout-driven pages providers defer their own data and
  the component gets a `_key` prop to reload with `router.reload({ only: [props._key] })` (see **lynx-provider**).

Code for each, Vue and React: [reference/props-and-navigation.md](reference/props-and-navigation.md).

### 7. Links to the page

Use Inertia `Link` / `router.visit()` between Lynx pages (header, footer and client state stay). Use a plain `<a>`
for pages you know leave Lynx: the checkout and pages handed to a standard theme answer an Inertia visit with `409` +
`X-Inertia-Location` and reload in full - the `<a>` saves that round trip. Links to other origins, `mailto:`, `tel:`
are followed by the browser either way.

## Rules

- Never set `layout="..."` on `<page>` in a `lynx_` file: Lynx forces page layout `lynx_column`, and a declared one
  makes Inertia visits receive full HTML instead of JSON. After removing one, clean the `layout` cache.
- No blocks in `content`, `sidebar.main` etc.: they do not exist in `lynx_column`. Only `head.additional`,
  `lynx.after.body.start`, `lynx.main.container`, `lynx.before.body.end`; block output is not re-rendered on Inertia
  visits. Page content is components.
- `<head>` assets added per page load only on a full page load; import code from the component instead.
- Inertia visits carry only the **title** (shared `title`) and **body class** (`bodyClass`); meta, canonical, Open Graph
  apply to full page loads (what crawlers get).
- `_lynx` / `_lynx_only` query params are added by the client to Inertia GETs so FPC stores JSON apart from HTML; the
  server strips them from the returned URL. Never build links with them or rely on them in controllers.
- Controller-declared pages have no header/footer/backstage zones on a full load. Do not use them for browse pages.
- Pages with personal data: `lynx_no_cacheable` + `ssr=false` + `encrypt_history=true` + NOINDEX.
- `PreviewPagebuilder` `pages` go in **global** `etc/di.xml`; an area-level declaration would replace the merged list
  in that area.
- `route` of a `sample_data` page must be the full `front/controller/action` (the preview router compares three parts).
- Labels on the page come from providers/controller props, translated with `__()`; never hardcode UI text in `.vue`/`.tsx`.
- Ship `view/frontend/lynx/vue/src` and `view/frontend/lynx/react/src` page files unless only one theme is targeted.

## Check

1. `bin/magento setup:upgrade` (routes, data patch), `bin/magento cache:clean config layout full_page lynx_component`.
2. New module with page/component files: `bin/magento lynx:workspace:setup <themes>` then
   `bin/magento lynx:static:deploy` (or `npm run build` in the project root).
3. Open `/faq` on a Lynx store view: full load shows the title and header/footer; navigate to it via a `Link` from
   another page - DevTools shows a JSON response (`X-Inertia: true`), not HTML.
4. Admin > PageBuilder > the design > page switcher lists **FAQ**; the preview loads the route; saving a layout shows on
   the storefront.
5. Private page: response has no FPC hit (`X-Magento-Cache-Debug` MISS each time in developer mode) and robots NOINDEX.
6. Check both Vue and React themes if the store builds both.

## Reference

- [reference/pages-argument.md](reference/pages-argument.md) - every `pages` key, sort ranges, sample entities
- [reference/layout-patch.md](reference/layout-patch.md) - starting-layout data patch, node shape, supplying the content
  zone from an observer
- [reference/props-and-navigation.md](reference/props-and-navigation.md) - shared/deferred/optional props, partial
  reload, Link, forms on the page
- [reference/page-component.md](reference/page-component.md) - controller-declared page component, Vue and React
- https://docs.oocscommerce.com/lynx/developer/inertia, https://docs.oocscommerce.com/lynx/developer/layout-xml
