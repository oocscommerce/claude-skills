# Mapping a Luma extension to Lynx, and what Magento does differently on a Lynx store

Everything here applies only to store views whose design theme is a Lynx theme or a child of one. Admin, cron, CLI,
REST, GraphQL and non-Lynx store views behave as standard Magento (except compiled configuration, which covers every
area).

## Full mapping

| Luma piece | On Lynx | How |
|---|---|---|
| Block + `.phtml` in a page via layout XML | Component + provider | Provider calls the same repository/view model/service; merchant places the component in PageBuilder |
| KO UI component, jQuery widget, `x-magento-init` / `data-mage-init` | Vue component | The JSON config becomes provider props |
| RequireJS mixin or `map` of core JS (`catalog-add-to-cart`, `Magento_Checkout/js/…`) | No equivalent (not loaded) | `after` plugin on the Lynx provider's `resolve()` to add data; your component in a slot of the Lynx component; or a theme override of its markup |
| Plugin on block output (`afterToHtml`), `*_html_after` observers | Never run: the block is never created | Hook the data the Lynx provider reads |
| View models of Luma blocks | Never run unless the block is declared in a `lynx_` file | Call them from your provider |
| `<referenceBlock>` / `<move>` on Luma blocks (`product.info.price`, `catalog.leftnav`) | Those blocks are not declared | Component, or theme override of the Lynx component |
| Blocks in `content`, `sidebar.main`, `product.info.main`… | Containers do not exist in `lynx_column` | `head.additional`, `lynx.after.body.start`, `lynx.main.container`, `lynx.before.body.end`, or a component |
| Customer data section (`sections.xml` + `SectionSourceInterface`) | Works unchanged | `useCustomerDataStore().section('<name>')`, reload after your own requests |
| AJAX controller returning JSON or a redirect | Reused | `fetch` with `X-Requested-With: XMLHttpRequest`, or Inertia `router.post` with `forceFormData: true` for redirect + flash |
| Storefront page with own route + layout | Controller runs, but its regular layout files are not read | Layout-driven page (preferred) or `InertiaService::setPage()`; or Page Fallback |
| `customer_account_navigation` tab | Account sidebar entry + section component | `account-tab` skill |
| Checkout `layoutProcessor`, `checkout_index_index.xml`, `checkoutConfig` mixins, KO checkout components | Do not apply (Lynx checkout) | Payment renderer, shipping renderer, or component in a checkout/cart slot |
| Express wallet button | Component in group `express_checkout`, placed in the Express Checkout slot | `payment-method` |
| Content beside checkout or cart | Component in **Below Order Summary** (checkout), **Below Items** / **Below Summary** (cart) | `checkout-step` |
| `<script>`/`<link>`/verification tag in `default.xml` | Same in `lynx_default.xml`, or a headless Backstage component | |
| Tracking on DOM events / KO observables | Tracker on the analytics bus (`begin_checkout`, `add_shipping_info`, `add_payment_info`, `purchase`, …) | `forms-analytics` |
| reCAPTCHA key under `recaptcha_frontend/type_for` + observer | Unchanged | `data-recaptcha="<key>"` on the form |
| `i18n/<locale>.csv`, `$t()` phrases | Same CSVs | Provider sends `__()` strings as `labels` |
| `widget.xml` widget on CMS pages | `{{widget}}` renders HTML, scripts do not run | Offer a component |
| Widget instances placed through **Content › Widgets** layout updates | Skipped | Component |
| Emails, admin, REST, GraphQL, cron, observers, plugins on models/services | Unchanged | |

## Page rendering differences

- Single-page app (Inertia): first visit returns HTML with page JSON; later links fetch JSON (`X-Inertia` header).
- Only `lynx_*.xml` layout files are collected; every handle gets a `lynx_` twin. Page layout is always
  `lynx_column`; never set `layout="…"` in a `lynx_` file.
- On Inertia visits the layout is still built (`_prepareLayout()` runs), but only `lynx.inertia` output is sent;
  title and body classes are applied, other head tags are only on full page loads.
- Head assets of one page load only when that page is the first one opened.
- Magento LESS is not used; Tailwind built from design settings.
- Still runs: routing, controllers, events/observers, service plugins, page config set by controllers and `lynx_`
  files, FPC and cache tags, sessions, form keys, flash messages.
- Head data set from a block's `_prepareLayout()`: declare that block (headless) in a `lynx_` file.

## CMS content

- CMS page **Content**: shown when **Render with Lynx** is off or no Lynx layout; directives expanded, scripts do not
  run. Layout Update XML of CMS pages is not applied.
- Category **Display Mode** / **Add CMS Block**: not rendered; the category shows its PageBuilder layout.
- Product/category descriptions: through Magento's output helper, scripts do not run.
- `{{baseUrl}}` inside layout XML stays literal: build URLs in PHP.

## Customer data (client)

- Storage in IndexedDB `lynx_customer_data`, not `mage-cache-storage` localStorage.
- No global AJAX invalidation listener; the provider sends `SectionInvalidation::getForPath('<route>')` (exact route
  keys, lowercase, plus `*`; no wildcard matching). With a wildcard in `sections.xml`, pass section names yourself.
- `wishlist` section: counter and count, empty `items`.

## Translations

- DB translations and Inline Translation do not apply; use CSV or a language pack.
- No JS translation dictionary (`js-translation.json`, `$t()`, `mage/translate`): translate on the server.
- With compiled config, CSV changes need `bin/magento lynx:compile translate`.

## Compiled configuration (`OOCS_LynxCompiler`)

`bin/magento lynx:compile` snapshots merged XML (events, routes, config, view, sections, webapi, extension
attributes, CSP whitelist, ACL, menu, indexer, queue, translations, component `.graphqls`) into `generated/lynx_compile`, read in every area.
`setup:upgrade`, `module:enable/disable` delete it; any covered XML change needs a new run; `cache:flush` does not
clear it. Order: `setup:upgrade` → `setup:di:compile` → `lynx:compile` → static deploy.

## Checkout

`checkout/index/index` is the Lynx checkout, always a full page load. Offline payment methods show without work;
online methods are hidden until they have a Lynx renderer. Multishipping is in the default Page Fallback list.

## Other runtime changes

Layout cache per Lynx theme; Inertia JSON responses carry no CSP header; RTL locales get `dir="rtl"`; UI component
factory created lazily; persistent cart check returns early when there is no persistent quote.

## Detecting Lynx in PHP

| Question | Use |
|---|---|
| Current request on a Lynx theme? | `OOCS\LynxTheme\Helper\Theme::isLynxTheme()` |
| A store view on a Lynx theme (cron, emails, admin)? | `OOCS\LynxComponent\Helper\ThemeIdentity::isLynxForStore($storeId)` |

Both treat a theme as Lynx when its code or a parent's starts with `OOCS/Lynx`.
