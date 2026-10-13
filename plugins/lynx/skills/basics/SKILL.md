---
name: basics
description: Shared foundation for making a Magento 2.4 module work on the Lynx storefront (Inertia.js SPA with Vue 3 or React 19 themes, PageBuilder layouts of components with PHP providers). Explains the moving parts, where a module's frontend code lives (`view/frontend/lynx/<vue|react>/src`, `Vendor_Module/...` imports, stores and utils), how to find the store's Lynx theme and framework, the `lynx_` layout XML prefix, server-side translations, styling with design tokens and `l-` daisyUI classes, Theme Fallback, Hyvä incompatibility, the `lynx:*` CLI and build/type-check/lint. Use first whenever a developer asks to "make my module work with Lynx", "Lynx compatible", "where do Lynx components go", "Lynx build", "Lynx styling", "why is my layout XML ignored on Lynx", or before any other lynx-* skill; it indexes them (component, provider, page, payment-method, theme, ssr…).
---
# Lynx basics

The shared ground every other `lynx-*` skill builds on: what Lynx changes in Magento, where a module's Lynx code goes,
how it is styled, translated, built and checked. Read it before writing any Lynx code, then switch to the task skill
listed at the end.

## Before you start

1. **Find Lynx.** Packages are `oocscommerce/*`: `vendor/oocscommerce/module-lynx-*` and
   `vendor/oocscommerce/theme-frontend-lynx-*`.
   Read code there to imitate built-in behavior; never edit it.
2. **Find the store's Lynx theme(s) and framework** (decides which files you write):
   - `bin/magento lynx:workspace:setup --list` lists the Lynx themes installed (prints and stops; no npm run).
   - The root `package.json` key `lynx.themes` lists the themes the project builds, each with `dir` and `parents`.
   - Which store view runs which theme: **Content › Design › Configuration**, or read-only
     `bin/magento config:show design/theme/theme_id --scope=stores --scope-code=<store>` then
     `SELECT theme_id, code, parent_id FROM theme WHERE area = 'frontend';`.
   - Framework: a theme whose parent chain reaches `OOCS/LynxVue` is Vue (entry `lynx/src/main.ts`, dev dependency
     `@oocs/lynx-vue`); one reaching `OOCS/LynxReact` is React (`main.tsx`, `@oocs/lynx-react`). `OOCS/LynxBlank` is a
     shared parent with no build; never assign it.
   - Any theme whose code, or a parent's code, starts with `OOCS/Lynx` is a Lynx theme
     (`OOCS\LynxComponent\Helper\ThemeIdentity::THEME_CODE`). Store views on other themes behave like plain Magento.
3. **Ship both frameworks** (`lynx/vue/src` and `lynx/react/src`) unless the developer targets one theme. A theme with
   no file for a component renders nothing for it.

## How Lynx works

Magento still routes, runs controllers, observers, plugins, FPC, sessions and the admin. On a Lynx store view the
controller's result becomes an **Inertia.js page** (component name + props): the first visit gets HTML with the page
JSON embedded (server-rendered when SSR is on), later clicks fetch only JSON (`X-Inertia` header, `_lynx` query param
keeps it apart in the FPC). There is no REST/GraphQL layer to write.

| Part | What it is | Code |
|---|---|---|
| Inertia page | Page component + props. Layout-driven pages use `OOCS_LynxTheme::Default` with a `layout` prop; a controller can declare its own page with `InertiaService::setPage()` | `OOCS\LynxInertia\Service\InertiaService` |
| Design | A merchant-edited set of layouts + design settings, chosen per store view in **Stores › Configuration › Lynx › Design** | `OOCS_LynxPageBuilder` |
| Layout | JSON tree of nodes stored per design under the full action name (`catalog_product_view_type_simple`, `contact_index_index`) | table `lynx_pagebuilder_layout` |
| Zones | `header`, `content` (inside `<main id="page-content">`, per page), `footer`, `backstage` (hidden; trackers, scripts) | components declare allowed zones |
| Component | A registered name in the `components` pool of `OOCS\LynxComponent\Model\Registry` (global `etc/di.xml`) → a `.vue`/`.tsx` file | skill component |
| Provider | PHP class turning a node's saved settings into props (`OOCS\LynxComponent\Api\ComponentProviderInterface::resolve()`, base `Model\AbstractComponentProvider`); can mark data `deferred()`/`optional()` | skill provider |
| LayoutResolver | Walks every node, merges defaults, calls the provider | `OOCS\LynxComponent\Model\LayoutResolver` |
| Rendering | `Pages/Default` + page shell `Layouts/Default`; `DynamicNode` loads each node's component chunk on demand | `OOCS_LynxTheme` |
| Design settings | Colors, corners, borders, widths, fonts per design → CSS variables on `:root` | see Styling |

Packages you depend on most: `OOCS_LynxComponent` (registry, provider interfaces, LayoutResolver),
`OOCS_LynxInertia` (InertiaService, SSR client), `OOCS_LynxPageBuilder` (designs, editor, editable pages list),
`OOCS_LynxTheme` (layout handles, page shell, global stores, build kits and commands), `OOCS_LynxThemeComponent`
(default components), `OOCS_LynxCatalog`, `OOCS_LynxCustomer`, `OOCS_LynxCheckout`. Full table:
[reference/magento-on-lynx.md](reference/magento-on-lynx.md).

## Module structure

```
app/code/Vendor/Module/                    (or a Composer package inside the project root)
├── etc/module.xml                         <sequence>: the OOCS_Lynx* modules you use
├── etc/di.xml                             components/pages pools (global scope)
├── Provider/…                             component providers
├── view/frontend/layout/lynx_<handle>.xml only lynx_ files are read on Lynx store views
├── i18n/en_US.csv                         every string the shopper sees
└── view/frontend/lynx/
    ├── vue/
    │   ├── package.json                   optional, only for npm dependencies (unique "name")
    │   └── src/
    │       ├── Components/                registered components (Highlight.vue)
    │       ├── Pages/                     controller-declared page components
    │       ├── stores/                    Pinia stores of your feature
    │       └── utils/                     plain functions
    ├── react/src/…                        same tree: .tsx components, jotai stores
    └── css/                               optional plain CSS, imported by a theme as "Vendor_Module/css/x.css"
```

`etc/module.xml` sequences what you use, mirrored in `composer.json` (`oocscommerce/module-lynx-*`):

```xml
<module name="Vendor_Module">
    <sequence>
        <module name="OOCS_LynxComponent"/>
        <module name="OOCS_LynxInertia"/>
        <module name="OOCS_LynxPageBuilder"/>
    </sequence>
</module>
```

Rules:

- The folder is fixed: `view/frontend/lynx/<vue|react>/src`. Never put sources in `view/frontend/web/` (static deploy
  publishes everything there).
- Keep the module inside the Magento root (`app/code`, `vendor/`, a path repo under the root): npm workspaces skip
  anything outside it.
- Same file names and exports in `vue/src` and `react/src`, so the two read alike.
- Components are self-contained: types, class strings and logic in the file; no shared helper/types files per folder.

### Imports by module name

Any `.vue`/`.ts` (React: `.tsx`/`.ts`) under a module's `src/` is imported as `<Module_Name>/<path in src>`; write the
extension for `.vue`, omit it for `.ts`/`.tsx`:

```ts
import DynamicNode from 'OOCS_LynxTheme/Components/DynamicNode.vue'
import { useMessagesStore } from 'OOCS_LynxTheme/stores/global/messagesStore'
import { config } from 'OOCS_LynxTheme/utils/storeConfig'
import { useWishlistStore } from 'Vendor_Module/stores/wishlistStore'
```

- The prefix is the module name from `registration.php`. Imports go through theme fallback: a theme override of a file
  wins everywhere, relative imports inside the module included.
- Importing another module is a hard build dependency: only import modules in your `<sequence>` and `composer.json`.
  For an optional integration use a `di.xml` pool or the `analyticsStore` bus, never a static import.
- Types used only in `import type` may live in `.d.ts`; types used at runtime go in a `.ts`.

### Stores, utils, global stores

Pinia on Vue, jotai on React. Put a feature store in the module that owns the feature (it gets its own chunk). Reuse
`OOCS_LynxTheme`'s instead of writing your own:

| Import | Gives |
|---|---|
| `OOCS_LynxTheme/stores/global/messagesStore` | `add('success' \| 'error' \| 'warning' \| 'notice', text)` |
| `OOCS_LynxTheme/stores/global/customerDataStore` | Customer data sections, login state, `invalidate(sections)` |
| `OOCS_LynxTheme/stores/global/formKeyStore` | `ensure()` the form key before a POST |
| `OOCS_LynxTheme/stores/global/analyticsStore` | `track(event, params)`, `addTracker(fn)` |
| `OOCS_LynxTheme/utils/storeConfig` | `config<T>(key, fallback)`: store config printed in the page |
| `OOCS_LynxTheme/utils/price` | `formatPrice(amount)` in the store currency |

A folder named `stores/global/` (in any module) is bundled into the always-loaded chunk: use it only for a store every
page needs. Store templates for both frameworks: [reference/stores.md](reference/stores.md).

A non-node component that must run on every page (a global listener, an SDK hook) goes in the `pageComponents`
argument of `OOCS\LynxTheme\ViewModel\StoreConfig` (`<item name="vendor_module" xsi:type="string">Vendor_Module/Components/X</item>`),
or is appended in an `afterBuild` plugin on it when conditional. Keep those small.

## Layout XML on Lynx

- Only `lynx_*.xml` layout files are collected, and every handle gets a `lynx_` twin: `default.xml` →
  `lynx_default.xml`, `catalog_product_view.xml` → `lynx_catalog_product_view.xml`, your route →
  `lynx_<route>_<controller>_<action>.xml`.
- The page layout is always `lynx_column`. **Never set `layout="…"`** on `<page>` in a `lynx_` file: Inertia visits
  would get HTML instead of JSON.
- Containers that exist: `head.additional`, `lynx.after.body.start`, `lynx.main.container` (next to the mount point
  `lynx.inertia`), `lynx.before.body.end`. `content`, `sidebar.main`, `product.info.main`… do not exist; Luma blocks
  (`product.info.price`…) are never declared.
- Still useful for: title/meta/robots, headless blocks that set page config in `_prepareLayout()`, `head.additional`
  markup, body-edge scripts, `<update handle="lynx_no_cacheable"/>`, arguments of `lynx.inertia` (`ssr`,
  `encrypt_history`, `clear_history`, `sri`).
- Block output is printed on full page loads only and is not re-rendered on Inertia visits (only the title and body
  class travel with the JSON). Visible, per-page or merchant-placed content is a component.

Examples and the "does not work" table: [reference/layout-xml.md](reference/layout-xml.md).

## Translations: server-side only

No `js-translation.json`, `$t()` or `mage/translate` on Lynx. Every string reaches the component already translated:

- Component labels → provider `labels` prop (`__()` in PHP). Merchant-typed text → `_i18n` settings resolved with
  `OOCS\LynxComponent\Helper\LocalizedText`. Controller pages → a `labels` prop. Global UI → `config('labels')`, added
  by an `afterBuild` plugin on `OOCS\LynxTheme\ViewModel\StoreConfig` (prefix keys with your module). Payment/shipping
  renderers → `labels` in the method's config. Server messages/errors → show Magento's `message` as is.
- Reuse Magento's existing phrases ("Add to Cart", "Sign In") so language packs already cover them; new ones go in
  `i18n/en_US.csv`.
- Values only the browser knows stay placeholders: translate `__('%1 items')` in PHP, replace `%1` in the component.
- No English fallback in components: a missing key renders empty so it shows up in development.
- DB translations and Inline Translation do not apply on Lynx store views.

## Styling

Tailwind CSS 4, scanned automatically (the build writes a `@source` per module into the theme's
`lynx/dist/sources.css`; never add `@source` paths counting `../`). Write class names in full; build variants from a
map, never by string concatenation.

- **Colors**: `text-base-content` (`/70`, `/60` muted), `bg-base-100/200/300`, `border-base-300`,
  `bg-primary text-primary-content` (and `secondary`, `accent`, `neutral`, `info`, `success`, `warning`, `error`, each
  with `-content`), links `text-link`.
- **Shape**: `rounded-box`, `rounded-field`, `rounded-selector`, `rounded-button`; border width
  `border-(length:--border)`; shadows `shadow-card`, `shadow-menu`, `shadow-dialog`, `shadow-thin`.
- **Widths**: `max-w-site`, `max-w-content`, `max-w-narrow`; header `min-h-header`. A content component's root is
  `w-full` with no `max-w-*`/`mx-auto`/outer padding (the Group frames it).
- **Controls**: daisyUI 5 with prefix `l-`, only `button, input, select, textarea, checkbox, radio, alert, card, table,
  skeleton` are built: `l-btn l-btn-primary`, `l-btn l-btn-outline l-btn-secondary`, `l-btn-sm`, `l-input`,
  `l-input-error`, `l-select`, `l-textarea`, `l-checkbox l-checkbox-primary`, `l-radio l-radio-primary`,
  `l-card l-card-border`, `l-table`, `l-skeleton`, `l-alert`. `l-modal`, `l-menu`, `l-dropdown`… produce no CSS:
  build those with Tailwind + tokens.
- **Font sizes**: `text-sm` (14px) to `text-5xl` only. Never `text-xs` or `text-[13px]`. `text-lg` and up follow the
  heading scale setting.
- **Never** for colors, corners, borders, shadows, control sizes or page widths: `bg-blue-600`, `text-gray-500`,
  `rounded-lg`, `border-2`, `shadow-md`, `h-10` on a button, `max-w-7xl`. Fixed scales only for icon sizes and inner
  spacing.
- **RTL**: logical utilities (`ms-/me-/ps-/pe-`, `start-/end-`, `text-start/end`, `border-s`, `rounded-e-*`); flip
  directional icons with `rtl:scale-x-[-1]`; `ltr:`/`rtl:` variant pairs where a physical direction is unavoidable.

Which design setting reaches which class: [reference/design-settings.md](reference/design-settings.md).

## Theme Fallback and Hyvä

- **Theme Fallback** (**Stores › Configuration › Lynx › Theme Fallback**, `lynx_fallback/general/*`: Enable Fallback,
  Fallback Theme, Fallback Routes) renders the listed full action names (one per line; a trailing `*` matches a
  prefix, e.g. `paypal_*`) with a standard Magento theme; Inertia visits to them become full page loads. Use it as a
  stopgap for a page you have not ported; the real fix is an Inertia page (skill page).
- **Hyvä**: Lynx cannot be combined with Hyvä (different stacks). An extension's Hyvä compatibility module does nothing
  on Lynx; port the storefront to Lynx components and reuse the PHP.

## CLI and build

| Command | Use |
|---|---|
| `bin/magento lynx:workspace:setup <themes…>` | Writes the root `package.json` (`workspaces`, `lynx.themes`, `lynx.modules`) and runs `npm install`. Name **every** theme you build (`OOCS_LynxVue OOCS_LynxReact Vendor_storefront`); `--list`, `--npm=<binary>`. Rerun after adding/enabling/disabling a module with frontend sources or `composer update`. |
| `bin/magento lynx:static:deploy [themes…]` | `npm run build` in each theme's `lynx/` (none = every theme in `package.json`); `--ssr` also builds the Node SSR bundle |
| `npm run build` | At the project root: every workspace. In a theme's `lynx/`: that theme. Runs `type-check` first. |
| `npm run type-check` | In a theme's `lynx/`: generates `tsconfig.modules.json` (`lynx-tsconfig-vue` / `lynx-tsconfig-react`) then `vue-tsc` / `tsc` |
| `bin/magento lynx:pagebuilder:build` | Builds the admin editor (`npm ci` + `npm run build`) |
| `bin/magento lynx:compile [types…]` | Snapshot of merged module XML in `generated/lynx_compile`; rerun after any XML change if the store uses it |

Theme names are the code with `_` for `/` (`frontend/OOCS/LynxVue` → `OOCS_LynxVue`). Only enabled modules listed in
`lynx.modules` are bundled. Output goes to the theme's `web/js/lynx/` and `web/css/lynx/`; in production run
`setup:static-content:deploy` **after** the Lynx build. The build log lists each chunk with the source file it came
from. What to run after which change: [reference/cli-and-workflow.md](reference/cli-and-workflow.md).

## Rules

- Pool arrays are merged per area by argument name, so add to them in the scope Lynx reads them: `components`,
  `pages` (`OOCS\LynxPageBuilder\Model\PreviewPagebuilder`), `pageComponents` in global `etc/di.xml`;
  `paymentMethodRenderers` and `shippingMethodRenderers` in `etc/frontend/di.xml`. A wrong scope silently replaces or
  drops entries.
- Components never hardcode UI text; labels come translated from PHP.
- Data a node needs for its first paint comes from its provider, not a fetch; shopper-specific data (cart, name,
  wishlist) comes from customer data in the browser, because page props are cached in the FPC.
- No browser globals (`window`, `document`, `localStorage`) at module scope or in Vue `<script setup>` top level:
  components render on the SSR server too (skill ssr).
- Widgets, CMS blocks placed via layout updates, KnockoutJS/jQuery/RequireJS, `x-magento-init` do not run on Lynx.
- Customer data lives in IndexedDB (`lynx_customer_data`), not `mage-cache-storage`; after your own POST, reload the
  sections the action invalidates via the customer data store.

## Check (any Lynx change)

1. `bin/magento lynx:workspace:setup <all built themes>` if the module is new/enabled, then `npm run build` (or
   `lynx:static:deploy`); a type error fails the build.
2. Lint with the theme's config, e.g.
   `npx eslint --config <theme>/lynx/eslint.config.mjs 'app/code/Vendor/Module/view/frontend/lynx/vue/src/**/*.{ts,vue}'`
   (React: `**/*.{ts,tsx}` with the React theme's config). It includes the SSR rule.
3. PHP: `vendor/bin/phpcs --standard=Magento2 app/code/Vendor/Module`; `declare(strict_types=1)`, one-line
   docblocks with `@param`/`@return`.
4. `bin/magento cache:clean config layout full_page` after `di.xml`/layout changes (`lynx_component` after writing
   layout rows), then check the page on both a Vue and a React store view, and in the PageBuilder preview.
5. Switch a design setting (button color, corners, border width) in the editor preview: everything you styled must
   follow.

## Other Lynx skills

| Skill | Use for |
|---|---|
| component | A new PageBuilder component: `components` pool entry, `.vue` + `.tsx`, zones, slots |
| component-settings | The settings panel of a component (setting types, `_i18n`, image, options) |
| provider | PHP provider: props, labels, deferred/optional data, cache tags, sample data |
| page | A storefront route: layout-driven or controller-declared Inertia page, editable page entry |
| forms-analytics | Forms (Inertia/fetch, form key, reCAPTCHA), customer data reload, analytics events |
| account-tab | A My Account tab: sidebar entry, section component, editor page |
| payment-method | Renderer for a payment method in the Lynx checkout |
| shipping-method | Renderer under a shipping rate (extra input, pickup) |
| checkout-step | A custom checkout step or changes to the built-in steps |
| product-type | Product page and add-to-cart support for a custom product type |
| theme | Child theme, component overrides, fonts and tokens |
| ssr | SSR-safe components, hydration, testing the renderer |
| extension-port | Porting an existing Luma extension's storefront to Lynx |
| mcp-tool | A tool for the Lynx MCP server |

## Reference

- [reference/magento-on-lynx.md](reference/magento-on-lynx.md): packages, request lifecycle, Magento differences
- [reference/layout-xml.md](reference/layout-xml.md): `lynx_` files, containers, examples
- [reference/design-settings.md](reference/design-settings.md): design setting → CSS variable → classes
- [reference/stores.md](reference/stores.md): Pinia and jotai store templates, npm dependencies
- [reference/cli-and-workflow.md](reference/cli-and-workflow.md): CLI options, what to run after which change
- https://docs.oocscommerce.com/lynx/developer/ , /developer/setup, /developer/magento-differences,
  /developer/layout-xml, /developer/frontend
