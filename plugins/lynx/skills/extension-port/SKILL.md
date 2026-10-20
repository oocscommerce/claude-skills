---
name: extension-port
description: Ports the storefront of an existing Luma Magento extension to a Lynx store in a separate compatibility module. Inventories what the extension sends to the storefront and maps each piece (block + phtml → component + provider, KnockoutJS/jQuery widget → Vue and React component, layout XML → lynx_<handle>.xml, RequireJS mixins and block-output plugins → provider plugins or slots, customer-data sections and AJAX controllers reused), carries over cache tags with PageCacheIdentities, adds lynx_no_cacheable for personal pages, uses Theme Fallback for whole pages not ported yet, and ends with a compatibility checklist. Also covers rebuilding a Luma store design with an AI assistant over the Lynx MCP server. Use when a developer says "make my extension Lynx compatible", "port Luma module to Lynx", "extension does not show on Lynx", "Hyvä compat module for Lynx", "migrate store to Lynx". Use with the basics skill.
---

# Lynx Extension Port

Produces a compatibility module `Vendor_<Extension>Lynx` that gives an existing Luma extension a Lynx storefront.
The extension's server side (models, controllers, observers, service plugins, cron, admin, REST, GraphQL, emails)
is reused; only the storefront UI is rebuilt. Worked example: `Vendor_ProductQuestions` →
`Vendor_ProductQuestionsLynx`.

## Before you start

- Read `basics`.
- Inventory the extension's storefront: `view/frontend/layout/*.xml`, `templates/*.phtml`, `web/js/**` (widgets,
  KO components, `requirejs-config.js` mixins/maps), `etc/frontend/sections.xml`, `CustomerData/*`,
  `etc/frontend/routes.xml` + `Controller/`, `etc/widget.xml`, plugins/observers on blocks (`afterToHtml`,
  `*_html_after`), checkout `layoutProcessor`s, `i18n/*.csv`.
- Note per block: which page/container, which repository/view model/config it reads, its `getIdentities()`, whether it
  is `cacheable="false"` or shows per-customer data.
- Map each item with the table below (long version: [reference/mapping.md](reference/mapping.md)).

| Luma piece | On Lynx | Skill |
|---|---|---|
| Block + `.phtml` placed by layout XML | Component + PHP provider calling the same services; merchants place it in PageBuilder | `component`, `provider` |
| KO UI component, jQuery widget, `x-magento-init` config | Same UI in Vue and React; the JSON config becomes provider props | `component` |
| RequireJS mixin / `map` of core JS | Not loaded. Plugin on the Lynx provider's `resolve()`, a component in a slot, or a theme override | `theme` |
| Plugin/observer on block output | Never runs (block is never created). Hook the data the Lynx provider reads | — |
| `sections.xml` + `SectionSourceInterface` | Unchanged; read with `customerDataStore`, reload after your own requests | below |
| AJAX controller | Reused; call with `fetch` or Inertia `router.post` | `forms-analytics` |
| Page with its own route + layout | Layout-driven page (preferred) or Inertia page component | `page` |
| `customer_account_navigation` tab | Account sidebar entry + section component | `account-tab` |
| Payment / shipping UI, checkout JS | Renderer in the Lynx checkout pools or a checkout slot component | `payment-method`, `shipping-method`, `checkout-step` |
| `<script>`/`<link>` in `default.xml` | Same in `lynx_default.xml`, or a headless Backstage component | `forms-analytics` |
| Tracking script on DOM/KO events | Tracker on the analytics bus | `forms-analytics` |
| reCAPTCHA key + predispatch observer | Unchanged; `data-recaptcha="<key>"` on the component's form | `forms-analytics` |
| `i18n/*.csv`, `$t()` phrases | Same CSVs; provider sends `__()` strings as `labels` | — |
| `widget.xml` widget on CMS pages | `{{widget}}` still renders HTML without scripts; offer a component | — |

Lynx and Hyvä cannot run together; a Hyvä compatibility module does not help. Port to components.

## Steps

### 1. Create the compatibility module (separate package)

```
Vendor/ProductQuestionsLynx/
├── composer.json            requires the extension + oocscommerce/module-lynx-component (+ page-builder, customer if used)
├── registration.php
├── etc/module.xml           <sequence>: Vendor_ProductQuestions, OOCS_LynxComponent, OOCS_LynxPageBuilder, OOCS_LynxCustomer
├── etc/components/Questions.graphqls   component declaration (settings, props)
├── etc/di.xml               editor pages (GLOBAL scope)
├── Provider/Questions.php
├── i18n/en_US.csv           only phrases the extension does not have
└── view/frontend/
    ├── layout/lynx_<handle>.xml
    └── lynx/{vue,react}/src/Components/Questions.{vue,tsx}
```

Do not put Lynx code inside the extension itself: `<sequence>` does not make a module optional, and a provider
extending `OOCS\LynxComponent\Model\AbstractComponentProvider` breaks `setup:di:compile` on stores without Lynx.
In the extension's `composer.json`, add the compat package under `suggest`, not `require`. Lynx's own add-ons do
the same (`OOCS_LynxPaymentStripe` only holds the Lynx front of the Stripe module).

### 2. Each block → component + provider

Declare it in `etc/components/<Template>.graphqls` (`type … @component(…)`), with a field for every key the provider
returns: the server drops undeclared props. Pick `group` by where merchants must drop it: Product Detail
slots and Product Media's Below Gallery accept `layout`, `text`, `media`, `form`, `product_list`, `product_attribute`,
`general` but not `product_page`; Product's Images and Details slots accept those and `product_page` too; `general`
lets it go into a Tab under the product. Use `availableOn: ["catalog_product_view*"]` for product-page blocks (every
type has its own path). Full `.graphqls`, provider, Vue and React: [reference/product-questions-example.md](reference/product-questions-example.md).

The provider plays the block's role:

```php
public function resolve(array $props): array
{
    $product = $this->registry->registry('current_product');
    if (!$product) {
        return $props;                                  // editor/other pages: settings only
    }
    // The tags the Luma block returned from getIdentities()
    $this->pageCacheIdentities->add([Question::CACHE_TAG]);
    $props['items'] = /* same repository call the block made */;
    $props['urls'] = ['post' => $this->urlBuilder->getUrl('productquestions/question/post')];
    // From the extension's sections.xml: what to reload after a successful post
    $props['sections'] = $this->sectionInvalidation->getForPath('productquestions/question/post');
    $props['labels'] = ['title' => (string) __('Questions & Answers') /* … */];

    return $props;
}
```

| Need | Use |
|---|---|
| Current product / category | `current_product` / `current_category` from `Magento\Framework\Registry` |
| FPC tags (replaces block `getIdentities()`) | `OOCS\LynxComponent\Model\PageCacheIdentities::add(array)` |
| Sections an action invalidates | `OOCS\LynxComponent\Helper\SectionInvalidation::getForPath('<route>')` (exact keys + `*`, no wildcards) |
| Merchant text settings (`LocalizedText`…) | `OOCS\LynxComponent\Helper\LocalizedText::get()` |
| Lazy block (fetched when its tab opens) | override `optional(array $props)`; see `provider` |

### 3. Widget JS → component

- `form_key` from `useFormKeyStore().ensure()` as a form field; header `X-Requested-With: XMLHttpRequest` so
  `isAjax()` controllers answer JSON; `credentials: 'same-origin'`.
- Show the controller's message with `useMessagesStore().addServerMessage(type, text)`; on success
  `useCustomerDataStore().reload(props.sections)`.
- Controller answers with redirect + flash messages instead of JSON: use Inertia `router.post(url, data,
  { forceFormData: true })`.
- reCAPTCHA: `data-recaptcha="<key>"` on the `<form>`, a `<div data-recaptcha-slot>`, read `g-recaptcha-response`
  synchronously in the submit handler (details in `forms-analytics`).
- All imports from `OOCS_LynxTheme/stores/global/{formKeyStore,messagesStore,customerDataStore}` in both themes.

### 4. Customer data

| Luma | Lynx |
|---|---|
| `customerData.get('x')` observable | `useCustomerDataStore().section('x')` (reactive) |
| `customerData.reload(['x'], true)` | `customerData.reload(['x'])` |
| Global AJAX listener invalidating from `sections.xml` | None: provider sends `getForPath()` list, component reloads |
| `customerData.invalidate([...])` | `customerData.invalidate([...])` |
| `localStorage` `mage-cache-storage` | IndexedDB `lynx_customer_data`; scripts reading localStorage find nothing |

Also on the store: `isLoggedIn`, `displayName`, `customer`, `cart`, `cartCount`, `wishlistCount`. The `wishlist`
section has an empty `items` list on Lynx stores.

### 5. Layout XML → `lynx_<handle>.xml`

Only `lynx_*.xml` files are read; each handle gets a `lynx_` twin (`catalog_product_view.xml` →
`lynx_catalog_product_view.xml`). Copy only `<head>` (title, meta, robots, `<script>`/`<link>`), page config,
headless blocks that set page config in `_prepareLayout()`, and cache behavior. Containers that exist:
`head.additional`, `lynx.after.body.start`, `lynx.main.container`, `lynx.before.body.end`. Never set `layout="…"` on
`<page>` (Inertia visits would get HTML instead of JSON).

### 6. Pages and personal data

- Prefer a layout-driven page (controller unchanged, content from components, `pages` entry + starting layout); a
  page component via `OOCS\LynxInertia\Service\InertiaService::setPage()` only for standalone screens. See
  `page`; account tabs: `account-tab`.
- **`lynx_no_cacheable`**: Luma's `cacheable="false"` blocks and `<update handle="customer_account"/>` are in regular
  layout files and are not read. Add `<update handle="lynx_no_cacheable"/>` to the `lynx_` file of every page with
  personal data, or one customer's page is served to the next.
- Providers on cached pages never read the customer session; per-visitor state comes from customer data in the
  browser.

### 7. Not porting a page yet: Theme Fallback

Merchant setting **Stores › Configuration › Lynx › Theme Fallback**: **Enable Fallback**, a non-Lynx **Fallback
Theme**, and full action names in **Fallback Routes** (one per line, trailing `*` = prefix, e.g. `productquestions_*`).
Those pages render with the fallback theme, so the extension's layout, templates and JS work; links to them become
full page loads. Whole pages only — a block on a Lynx page cannot fall back. **Fallback Routes** already holds Lynx's
defaults: never set it in your `config.xml`; give merchants the line to add in your install guide.

### 8. Branching PHP on Lynx (only if needed)

In the compat module: `OOCS\LynxTheme\Helper\Theme::isLynxTheme()` (current request) or
`OOCS\LynxComponent\Helper\ThemeIdentity::isLynxForStore(int $storeId)` (cron, emails, admin). Inside the extension
(no Lynx dependency), test the theme chain of `Magento\Framework\View\DesignInterface::getDesignTheme()` for codes
starting with `OOCS/Lynx`.

### 9. Install, build, tell merchants

```bash
bin/magento module:enable Vendor_ProductQuestionsLynx
bin/magento setup:upgrade
bin/magento lynx:workspace:setup OOCS_LynxVue OOCS_LynxReact   # the themes the store builds
bin/magento lynx:static:deploy
bin/magento lynx:compile        # when the store uses compiled configuration (components, sections/routes/events)
bin/magento cache:clean
```

A Luma block appeared on install; a component appears once placed. Write in the install guide where to drop it
(e.g. product page → Tabs → add a Tab → **Product Questions**). Ship a starting layout only for **new pages your
module declares**, into the default design.

## Rules

- **Do not write merchants' layouts from a data patch** (e.g. injecting your node into their product page): those
  layouts belong to their designs.
- Components in `etc/components/*.graphqls`; the `pages` pool in global `etc/di.xml`; `paymentMethodRenderers` in
  `etc/frontend/di.xml`. Components use the generated `Lynx.<TypeName>`, not hand-written props types.
- Labels translated in PHP with `__()` and sent as `labels`; reuse Magento phrases ("Sign In", "This is a required
  field."). Extension CSVs keep working; DB / Inline translations do not apply on Lynx (CSV or language pack;
  `bin/magento lynx:compile translate` with compiled config).
- Ship both `vue` and `react` folders: a theme renders nothing for a component it has no file for.
- Tokens and `l-` classes only (`l-btn`, `l-input`, `l-textarea`, `l-alert`, `l-skeleton`), logical classes (`ps-4`,
  `border-s`) for RTL, `text-sm` minimum, one root element, no `window`/`document` during render.
- Head assets in a page-specific `lynx_` file load only on full page loads; import code from the component instead.

## Check

- [ ] Full page load and Inertia navigation (click in, Back) show the same content.
- [ ] SSR on: no hydration warnings. Both Lynx Vue and Lynx React. RTL store view.
- [ ] Editor preview: change Button color, Corners, Border width — the component follows.
- [ ] Guest and signed in; customer data after sign-in/out; section reload after a post.
- [ ] FPC on: no personal data in cached pages across customers; saving the entity in admin refreshes the page.
- [ ] Lazy option inside a Tab; feature disabled in config hides the tab.
- [ ] Component offered on the right pages and slots, with sample product / `sample_data`.
- [ ] reCAPTCHA with each type and with none; other locale has no English left.
- [ ] After `lynx:compile`, new `.graphqls` / `sections.xml` / `routes.xml` / `events.xml` entries are picked up.

## Rebuilding a store design with an AI assistant (MCP)

Moving a store to Lynx moves no data; the look is rebuilt. With the MCP server connected (`mcp-tool`,
https://docs.oocscommerce.com/lynx/pagebuilder/build-with-ai), an assistant that can open web pages works one part
at a time in a separate design, checking each preview:

1. "Create a Lynx design called Migration based on Default, and work only in that design."
2. Brand: read the old store's colors, corners, text sizes → set the design's theme settings.
3. Footer (menu comes from categories), then the home page section by section (images via `media_upload_image`,
   real products via `catalog_search_products`), then CMS pages / categories with **Render with Lynx** left off
   until reviewed.
4. "List every section and the Lynx component you would use. Mark sections no component fits." For those: build
   from Group/Row/Column/Heading/Paragraph/Image/Button, or have a developer write a component (`component`),
   then `lynx:static:deploy`, `cache:clean`, restart `bin/magento lynx:mcp:serve` so the assistant sees it.
5. Review previews (phone + desktop), then assign the design and Lynx theme to the store view. Local layouts move
   with the editor's Export/Import layout; locally uploaded images must be uploaded on live too.

## Reference

- [reference/mapping.md](reference/mapping.md) — full mapping, Magento differences on Lynx stores
- [reference/product-questions-example.md](reference/product-questions-example.md) — complete worked port
- https://docs.oocscommerce.com/lynx/developer/extension-compatibility
- https://docs.oocscommerce.com/lynx/developer/magento-differences
- https://docs.oocscommerce.com/lynx/developer/migrate
