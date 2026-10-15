# Renderer internals

## Contract (both runtimes)

| Request | Response |
|---|---|
| `POST /render` with the page as JSON | `200 { "head": string[], "body": string }` |
| `POST /render` that fails | `500 { "error", "component", "url", "stack" }` |
| `GET /health` | `200 { "status": "OK" }` |
| Any other path | `200 { "status": "NOT_FOUND" }` (Magento treats it as a failure) |

Request body: the Inertia page object plus `storeConfig` (store-scope config the components read with `config()`):

```json
{
  "component": "OOCS_LynxTheme/Pages/Default",
  "props": { "layout": { "header": [], "content": [], "footer": [], "backstage": [] } },
  "url": "/women/tops.html",
  "version": "…",
  "storeConfig": { "storeCode": "default", "currencyCode": "USD", "pageComponents": [] }
}
```

`body` is the page script plus `<div id="lynx" data-server-rendered…>…</div>`; Magento prints it in the
`lynx.inertia` block (`OOCS_LynxInertia::inertia.phtml`) instead of the empty mount point. `head` is ignored.

## `ssrRender.ts` (module `OOCS_LynxTheme`)

`view/frontend/lynx/vue/src/ssrRender.ts` and `view/frontend/lynx/react/src/ssrRender.ts`, imported as
`OOCS_LynxTheme/ssrRender`, export `renderEnvelope(envelope)`:

- builds the Inertia app once per process with the same page map (`virtual:lynx-pages`) and default layout
  (`OOCS_LynxTheme/Layouts/Default`) as the browser entry;
- per render: `setStoreConfig(storeConfig)`, a fresh Pinia (`createPinia()`) or store registry (`createRegistry()`);
- renders one page at a time per process (a promise queue), because the store config is process-wide during a render;
- React only: `await preloadComponents(pageComponents(page))` before `renderToString`.

It uses neither Node nor browser APIs, so the same code runs in both runtimes.

| Runtime | Entry (`<theme>/lynx/src/`) | Build | Output |
|---|---|---|---|
| Node.js | `ssr.ts` (`createServer` of `@inertiajs/vue3/server` / `@inertiajs/react/server`) | `npm run build:ssr` | `dist/ssr/nodejs/server.mjs` (target node22, all deps bundled, source maps) |
| Cloudflare Worker | `ssr-worker.ts` (`export default { fetch }`) | `npm run build:ssr:worker` | `dist/ssr/worker/worker.mjs` (one file, `ssr.target: 'webworker'`) |

The Node server also answers `/shutdown`. Workers: `ssr` (Vue theme), `lynx-ssr-react` (React theme) in
`wrangler.toml`.

## When a request is rendered

`OOCS\LynxInertia\Service\SsrGate::isEnabled()`:

| Condition | Source |
|---|---|
| SSR enabled for the store view | `lynx_ssr/general/enabled` |
| `GET`, no `X-Inertia` header | request |
| Full action name matches a line of Routes (`fnmatch`) | `lynx_ssr/general/routes` |
| Customer group selected (from the HTTP context the FPC varies on) | `lynx_ssr/general/customer_groups` |
| Not a PageBuilder preview | plugin `OOCS\LynxPageBuilder\Plugin\DisableSsrInPreview` |

Then `OOCS\LynxInertia\Block\Inertia` skips the renderer when its `ssr` argument is set and false. All settings are
store-view scope, so a Vue store view and a React store view can point at their own renderers.

## Plugin on SsrGate

`etc/frontend/di.xml`:

```xml
<type name="OOCS\LynxInertia\Service\SsrGate">
    <plugin name="vendor_module_disable_ssr" type="Vendor\Module\Plugin\DisableSsrForCampaign"/>
</type>
```

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin;

use Magento\Framework\App\RequestInterface;
use OOCS\LynxInertia\Service\SsrGate;

class DisableSsrForCampaign
{
    /**
     * @param RequestInterface $request
     */
    public function __construct(
        private readonly RequestInterface $request,
    ) {
    }

    /**
     * Keep campaign landing requests client-rendered
     *
     * @param SsrGate $subject
     * @param bool $result
     * @return bool
     */
    public function afterIsEnabled(SsrGate $subject, bool $result): bool
    {
        return $result && !$this->request->getParam('utm_campaign');
    }
}
```

Mind the FPC: the rendered HTML is cached; a condition the full page cache does not vary on caches whichever variant
came first.

## pageComponents

Components Lynx mounts on every page next to the layout (and that the React renderer preloads). Unconditional, global
`etc/di.xml`:

```xml
<type name="OOCS\LynxTheme\ViewModel\StoreConfig">
    <arguments>
        <argument name="pageComponents" xsi:type="array">
            <item name="vendor_module" xsi:type="string">Vendor_Module/Components/ChatLauncher</item>
        </argument>
    </arguments>
</type>
```

Conditional, an `afterBuild` plugin on `OOCS\LynxTheme\ViewModel\StoreConfig` (`build(StoreInterface $store)`):

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin;

use OOCS\LynxTheme\ViewModel\StoreConfig;

class AddPageComponent
{
    /**
     * Mount the chat launcher on every page
     *
     * @param StoreConfig $subject
     * @param array $result
     * @return array
     */
    public function afterBuild(StoreConfig $subject, array $result): array
    {
        $result['pageComponents'][] = 'Vendor_Module/Components/ChatLauncher';

        return $result;
    }
}
```

## Hydration in the browser entries

- `main.tsx` (React): if the mount point has `data-server-rendered`, `await preloadComponents(pageComponents(props.initialPage))`
  then `hydrateRoot`; otherwise `createRoot().render()`. Loaded components resolve directly, not through `lazy`, so the
  page hydrates in one pass.
- `main.ts` (Vue): `createSSRApp` when `data-server-rendered`, else `createApp`; `preloadTree(zone)` for every layout
  zone requests all chunks at startup; each node hydrates as its chunk arrives. `DynamicNode` reads `preview` /
  `preview_outlines` through `computed`, so replacing page props does not re-render nodes; only nodes with deferred
  data update, and those were rendered as placeholders on the server.
- The 300 ms reveal window that batches chunk mounts on client-rendered pages is off on the server and when hydrating
  server markup.

## Operating

- Rebuild and restart the Node renderer (or redeploy the Worker) with every client build.
- Run Node under a process manager, e.g. a systemd unit with `WorkingDirectory=<theme>/lynx`,
  `Environment=LYNX_SSR_HOST=127.0.0.1`, `Environment=LYNX_SSR_CLUSTER=1`,
  `ExecStart=/usr/bin/node --enable-source-maps ./dist/ssr/nodejs/server.mjs`, `Restart=always`.
- Only routes crawlers read are worth rendering; each route costs one render per FPC miss.
