# Magento on a Lynx store view

## Packages

| Role | Module | Package | Purpose |
|---|---|---|---|
| Foundation | `OOCS_LynxComponent` | `module-lynx-component` | Component registry (`components` pool), provider interfaces, `LayoutResolver`, `lynx_component` cache type. No PageBuilder dependency. |
| Foundation | `OOCS_LynxInertia` | `module-lynx-inertia` | `InertiaService` (pages, shared/deferred/optional props), JSON responses, asset versioning, SSR client (`SsrGate`, `SsrRenderer`), SRI |
| Foundation | `OOCS_LynxPageBuilder` | `module-lynx-page-builder` | Designs, layouts, admin editor, preview/share, design settings, editable pages list (`PreviewPagebuilder`) |
| Foundation | `OOCS_LynxTheme` | `module-lynx-theme` | `lynx_*` handles, `lynx_column`, Theme Fallback, page shell, global stores, build kits `@oocs/lynx-vue` / `@oocs/lynx-react`, build commands |
| Components | `OOCS_LynxThemeComponent` | `module-lynx-theme-component` | Default components: header, footer, group, row, banner, carousel, heading, cart, checkout success, contact form… |
| Components | `OOCS_LynxCatalog` | `module-lynx-catalog` | Product pages per type, category, search |
| Components | `OOCS_LynxCustomer` | `module-lynx-customer` | Sign-in, registration, account area, orders, returns |
| Components | `OOCS_LynxCms` | `module-lynx-cms` | CMS pages built with Lynx |
| Checkout | `OOCS_LynxCheckout` | `module-lynx-checkout` | One-page checkout, payment/shipping renderer pools |
| Checkout | `OOCS_LynxInStorePickup` | `module-lynx-in-store-pickup` | In-store pickup in the Lynx checkout |
| Payments | `OOCS_LynxPaymentPayPal`, `OOCS_LynxPaymentBraintree`, `OOCS_LynxPaymentStripe` | `module-lynx-payment-*` | Payment renderers (Braintree, Stripe are add-ons) |
| Integrations | `OOCS_LynxGoogleTag`, `OOCS_LynxReCaptcha` | `module-lynx-google-tag`, `module-lynx-re-captcha` | GA4 backstage component, reCAPTCHA for Lynx forms |
| Performance | `OOCS_LynxCompiler`, `OOCS_LynxOptimize` | `module-lynx-compiler`, `module-lynx-optimize` | `lynx:compile`; skipping work Lynx does not need |
| AI | `OOCS_LynxMcp` | `module-lynx-mcp` | MCP server |

Themes: `OOCS/LynxBlank` (shared Magento-level files, no build), `OOCS/LynxVue`, `OOCS/LynxReact` (children of Blank).

## Request lifecycle

```text
Browser ──GET /contact──▶ router ──▶ controller (returns a result page)
        layout handles default, contact_index_index + lynx_default, lynx_contact_index_index
        page layout forced to lynx_column
        InertiaService::resolvePage()
          no page declared? → PageBuilder reads the active design's layout:
          header/footer/backstage zones + content stored under "contact_index_index"
        LayoutResolver: each node's provider → props
        first visit: HTML with page JSON (SSR markup when on)  |  Inertia visit: JSON only
Browser: page component OOCS_LynxTheme/Pages/Default inside Layouts/Default; DynamicNode renders each node
```

Header, footer and backstage are sent on full loads and partial reloads and kept by the client across visits; the
content zone is sent on every response.

## Two kinds of pages

| | Layout-driven | Controller-declared |
|---|---|---|
| Controller | Returns a result page, declares nothing | Calls `InertiaService::setPage($component, $props, $url)` |
| Content | PageBuilder layout under the full action name | Your page component (`Vendor_Module::Ask` → `Vendor_Module/Pages/Ask`) |
| Header/footer/backstage | From the design | Not sent |
| Merchant-editable | Yes (after adding to `pages` of `OOCS\LynxPageBuilder\Model\PreviewPagebuilder`) | No |
| Data | Component providers | Controller props |

Details in skill lynx-page.

## Magento features on Lynx

| Feature | On Lynx | Instead |
|---|---|---|
| RequireJS, KnockoutJS, UI components, jQuery widgets, `x-magento-init`, `data-mage-init` | Not loaded | A Lynx component |
| Regular layout XML | Not collected; only `lynx_*.xml` | `lynx_<handle>.xml` for head/page config; components for content |
| Page layouts (`1column`…), Layout field of entities | Ignored, always `lynx_column` | Arrange in PageBuilder |
| `<head>` assets from layout | Only from `lynx_*.xml`, full loads only | Load scripts from a component on mount |
| LESS styles | Not used | Tailwind with design tokens |
| Other modules' pages | Controller runs; only header/footer + the PageBuilder layout for that action show | Return an Inertia page, or Theme Fallback |
| Widgets via layout updates, CMS Layout Update XML | Not applied | Components |
| CMS page Content field | Shown when Render with Lynx is off / no Lynx layout; directives expanded; inline scripts do not run | Component for interactive parts |
| Category Display Mode / CMS block | Not rendered | Render with Lynx + design the category |
| DB translations, Inline Translation | Not applied | `i18n/*.csv`, language packs |
| JS translation (`$t()`, `mage/translate`) | Not loaded | Labels translated in PHP |
| Customer data | Same `sections.xml`/`SectionSourceInterface`; stored in IndexedDB; no global AJAX invalidation; `wishlist` section has an empty `items` list | Reload invalidated sections from your component |
| Checkout | Lynx checkout; `layoutProcessor` plugins, `checkout_index_index.xml`, `checkoutConfig` mixins do not apply; offline methods work, online methods need a renderer | skills lynx-payment-method, lynx-shipping-method, lynx-checkout-step |
| Inertia JSON responses | No CSP header (full loads have it) | — |
| RTL locales | `dir="rtl"` on `<html>` | Logical Tailwind utilities |

What still runs on every Lynx request: routing and controllers, events/observers, plugins on models and services,
page config from controllers and `lynx_*.xml`, FPC and cache tags, sessions, form keys, flash messages.

## Porting checklist

| Extension has | On Lynx |
|---|---|
| Observers, plugins, models, cron, admin, REST/GraphQL | Unchanged |
| Storefront UI in layout XML / `.phtml` / KnockoutJS / jQuery | Lynx component + PHP provider (skill lynx-component) |
| Own storefront page | Inertia page (skill lynx-page), or Theme Fallback |
| Forms to own controllers | Inertia form or `fetch`, then reload invalidated sections (skill lynx-forms-analytics) |
| Tracking scripts | Analytics bus / backstage component |
| Payment or shipping UI | Renderer (skills lynx-payment-method, lynx-shipping-method) |
| New/changed XML config | `bin/magento lynx:compile` if the store uses it |

Full walkthrough: skill lynx-extension-port.
