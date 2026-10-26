---
name: ssr
description: Makes Lynx storefront code safe for server-side rendering and verifies it - Vue components, stores and theme overrides that render in the Lynx SSR renderer (Node.js `server.mjs` or Cloudflare Worker), without browser globals at module/setup time, with markup that hydrates cleanly, deferred/optional props that show a placeholder, and per-route control (Routes config, `ssr` argument on `lynx.inertia`, plugin on `SsrGate::isEnabled()`). Covers building and running the renderer locally and checking `data-server-rendered`. Use when a developer says "SSR breaks my component", "window is not defined" / "document is not defined" in the Lynx renderer, "hydration mismatch on Lynx", "content flickers after load", "enable SSR for my route", "disable SSR for a page", or "test Lynx SSR locally". Use with the basics skill.
---
# SSR-safe Lynx code

Produces components (and overrides, stores, page components) that render the same markup in the Lynx SSR renderer
and in the browser, plus the configuration that decides which routes are rendered, and a local test of both.

## Before you start

- Read **basics**. SSR runs the theme's own Vue code; every node on a rendered route runs on the
  server first, so this applies to each `.vue` you write, even if the store has SSR off today.
- How it works: on a full-page `GET` (not an Inertia visit) that `OOCS\LynxInertia\Service\SsrGate::isEnabled()`
  accepts, `OOCS\LynxInertia\Service\SsrRenderer` POSTs the Inertia page object plus `storeConfig` to
  `{Renderer URL}/render`, and Magento prints the returned `body` in place of the empty mount point. The result is
  stored in the FPC. Any failure falls back to client rendering and logs a `Lynx SSR:` warning in
  `var/log/system.log`. Later navigations are Inertia JSON visits, rendered in the browser.
- The shared render code is `OOCS_LynxTheme/ssrRender` (`module-lynx-theme/view/frontend/lynx/vue/src/ssrRender.ts`);
  each theme has thin entries `lynx/src/ssr.ts` (Node) and `lynx/src/ssr-worker.ts` (Worker). Read them when
  debugging; do not change them.
- Find the store's settings (store-view scope, **Stores › Configuration › Lynx › SSR**):
  `bin/magento config:show lynx_ssr/general` — `enabled` (default `0`), `url` (default `http://127.0.0.1:13714`),
  `timeout` (ms, default 2000, min 100), `routes` (default `cms_index_index`, `catalog_category_view`,
  `catalog_product_view`), `customer_groups` (default `0`, guests).

## Steps

### 1. Decide whether your route is rendered

`SsrGate::isEnabled()` requires all of: SSR enabled for the store view, `GET`, no `X-Inertia` header, the full action
name matching a line of **Routes** (`fnmatch`, e.g. `catalog_*`), the visitor's customer group selected, and not a
PageBuilder preview (`OOCS\LynxPageBuilder\Plugin\DisableSsrInPreview`).

- **Public, crawlable page** (an FAQ, a brand page): tell the merchant to add its full action name to Routes
  (`vendor_faq_index_index`). Routes are configuration, not code: do not set them from a data patch.
- **Personal or uncached page** (account area, a dashboard): switch SSR off in its layout. `ssr` can only switch it
  off; `true` never forces a render. Lynx does this for the cart, success/failure, payment review and account pages.

  ```xml
  <!-- view/frontend/layout/lynx_vendor_dashboard_index_index.xml -->
  <?xml version="1.0"?>
  <page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
      <update handle="lynx_no_cacheable"/>
      <body>
          <referenceBlock name="lynx.inertia">
              <arguments>
                  <argument name="ssr" xsi:type="boolean">false</argument>
              </arguments>
          </referenceBlock>
      </body>
  </page>
  ```

- **Your own condition**: an `after` plugin on `SsrGate::isEnabled()` in `etc/frontend/di.xml`, returning
  `$result && !…`. Example in [reference/renderer.md](reference/renderer.md#plugin-on-ssrgate).

### 2. No browser globals at module scope or during setup/render

`window`, `document`, `localStorage`, `sessionStorage`, `navigator`, `matchMedia`, `IntersectionObserver` do not exist
in the renderer. The top level of `<script setup>` runs on the server (counts as module scope). Use them only in
`onMounted` or event handlers. Start from a neutral state that the server and the first browser render agree on.

```vue
<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref } from 'vue'

// Same value on the server and in the first browser render
const isWide = ref(false)
let media: MediaQueryList | null = null
const update = () => (isWide.value = !!media?.matches)

onMounted(() => {
    media = window.matchMedia('(min-width: 64rem)')
    update()
    media.addEventListener('change', update)
})
onBeforeUnmount(() => media?.removeEventListener('change', update))
</script>
```

Also:

- `watchEffect` and `watch(…, { immediate: true })` callbacks run once during the server render: no browser APIs
  in them. `onMounted`/`onBeforeUnmount` never run on the server.
- No `setInterval`/`setTimeout`/listeners in setup or render: start them on mount and clear them on unmount.
- An npm SDK that touches `window` when imported: import it dynamically on mount
  (`const { init } = await import('some-sdk')` inside `onMounted`).
- `Teleport` to `body`: render it only after mount (`v-if="mounted"`), as the built-in Header and ProductListing do.
- Store configuration comes from `config()` of `OOCS_LynxTheme/utils/storeConfig` (the renderer gets it with the
  request); never read it from the DOM.

The theme's ESLint config reports DOM globals at module scope (`ssr-friendly/no-dom-globals-in-module-scope`). It does
not catch reads inside render functions or computed values.

### 3. Render the same markup on both sides

The first browser render must equal the server markup, or Vue reports a hydration mismatch and redraws that part.

- Never branch output on `typeof window`, the time, `Math.random()`, viewport size or `navigator.language`. Render the
  neutral state first, change it after mount.
- Customer data (login details, cart, wishlist) loads in the browser after hydration: render the guest/empty state
  first and let the customer data store update it.
- Format dates and other locale/time-zone-dependent text in the provider (PHP), not with `toLocale*()` in the
  component: the renderer's time zone and locale are not the shopper's.
- Element ids for `label`/`aria-*` pairs: `useId()` (Vue 3.5), never counters or random values.
- No per-visitor state in module-level variables: the renderer process serves every visitor. Keep it in stores
  (created per render: a Pinia instance per request).

### 4. Headless components

A component that renders nothing must produce the same placeholder on both sides.

- No `<template>` at all (an empty one or one holding only a comment shifts every later node during
  hydration). Declare the render function and do browser work on mount:

  ```vue
  <script setup lang="ts">
  import { onMounted } from 'vue'

  defineOptions({ render: () => null })

  onMounted(() => {
      // browser-only work
  })
  </script>
  ```


### 5. Data from providers, deferred props as placeholders

- The server render never calls your endpoints and renders run one at a time per process: no `fetch` or top-level
  `await` for first-paint data in a component. Everything the first paint needs comes in props from the PHP provider
  (skill provider).
- A node whose provider returns `deferred()`/`optional()` `true` is rendered with its saved settings only; the data
  props are `undefined` on the server and in the first browser render, then merged in. Always render a placeholder of
  about the final size while they are `undefined`:

  ```vue
  <div v-if="products === undefined" class="l-skeleton h-64 w-full" aria-busy="true"></div>
  ```

- Deferred data is not in the server HTML: do not defer content above the fold or important for search. Labels of a
  deferred provider arrive with its data too.
- In a controller-declared page, Inertia's `<Deferred>` renders its fallback on the server.

### 6. Keep the hydration behavior of the entry files

Server markup appears at once, but a node hydrates only when its chunk has loaded; a node that receives an update
while still waiting (initial page props, a deferred prop arriving, customer data) loses its server markup and
flickers. `renderStorefront()` (called by the theme's `main.ts`) prevents it: it calls `preloadTree()` for every
zone, and `DynamicNode` reads preview flags through `computed` so replacing page props does not re-render nodes. In a
child theme that overrides `OOCS_LynxTheme/storefront.ts` or `DynamicNode`, keep both. Use `createSSRApp` only when
the mount point has `data-server-rendered`.

## Rules

- The renderer and the browser bundle must come from the same build: rebuild the SSR bundle and restart the Node
  process (or redeploy the Worker) after every client build, or hydration will not match.
- Worker builds use browser-compatible dependency versions (`ssr.target: 'webworker'`) and one file without code
  splitting: an npm dependency that needs Node built-ins fails `npm run build:ssr:worker`.
- The renderer has no authentication (`/shutdown` stops the Node process): keep it on `127.0.0.1` or a private
  network (`LYNX_SSR_HOST`); protect a Worker URL with Cloudflare Access or a WAF rule.
- `<head>` comes from Magento's layout, not from components: the renderer's `head` array is ignored.
- Only the first full-page load is rendered; Inertia visits never are.

## Check

1. Lint (catches module-scope DOM globals) and build the client: `npm run build` in the theme's `lynx/`.
2. Build and start the renderer, point the store at it, request the page as a guest, confirm
   `data-server-rendered="true"`: [reference/testing.md](reference/testing.md).
3. Open the page in a browser with the console open: no hydration mismatch error, no content disappearing and
   reappearing after load (throttle the network to see it). Compare with JavaScript disabled to see the server output.
4. Keep a `hydrate()` case in the component's test so a regression fails `bin/magento lynx:static:test` (skill **testing**).
5. Read render errors in the Node process's stderr (source-mapped) or `npm run tail:ssr:worker`; fallbacks in
   `var/log/system.log` (`Lynx SSR: renderer did not answer` / `refused the page` / `page cannot be encoded`).
6. Restore the store's previous `lynx_ssr/general/*` values if you changed them only to test.

## Reference

- [reference/testing.md](reference/testing.md): local build, config commands, curl checks, direct `/render` test
- [reference/renderer.md](reference/renderer.md): renderer contract, runtimes, env vars, SsrGate plugin, pageComponents
- https://docs.oocscommerce.com/lynx/developer/ssr, https://docs.oocscommerce.com/lynx/guide/ssr
