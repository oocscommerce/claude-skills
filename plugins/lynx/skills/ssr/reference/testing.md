# Testing SSR locally

Run these where Node.js 22.12+ is available (the project's build container if PHP and Node live in containers).
`<theme>/lynx` is the theme's workspace, e.g. `vendor/oocscommerce/theme-frontend-lynx-vue/lynx`,
`vendor/oocscommerce/theme-frontend-lynx-react/lynx` or your child theme's `app/design/frontend/Vendor/storefront/lynx`.

## 1. Build both bundles from the same sources

```bash
bin/magento lynx:static:deploy OOCS_LynxVue --ssr        # client + Node SSR bundle
# or, in <theme>/lynx:
npm run build && npm run build:ssr                       # → dist/ssr/nodejs/server.mjs
npm run build:ssr:worker                                 # Worker bundle → dist/ssr/worker/worker.mjs (catches Node-only deps)
```

## 2. Start the Node renderer

```bash
cd <theme>/lynx
LYNX_SSR_HOST=127.0.0.1 npm run ssr        # node --enable-source-maps ./dist/ssr/nodejs/server.mjs
curl -s http://127.0.0.1:13714/health      # {"status":"OK",...}   (React theme default port: 13715)
```

| Variable | Default | Purpose |
|---|---|---|
| `LYNX_SSR_PORT` | `13714` Vue, `13715` React | Listening port |
| `LYNX_SSR_HOST` | `0.0.0.0` | Listening address; `127.0.0.1` when PHP is on the same host |
| `LYNX_SSR_CLUSTER` | unset | `1` forks one worker per CPU core |

If PHP runs in another container, use an address PHP can reach and keep the port off public interfaces.

## 3. Point the store view at it

Store-view scope (`default` is the store code); note the current values first to restore them afterwards:

```bash
bin/magento config:show lynx_ssr/general
bin/magento config:set --scope=stores --scope-code=default lynx_ssr/general/enabled 1
bin/magento config:set --scope=stores --scope-code=default lynx_ssr/general/url http://127.0.0.1:13714
bin/magento config:set --scope=stores --scope-code=default lynx_ssr/general/routes \
  "$(printf 'cms_index_index\ncatalog_category_view\ncatalog_product_view\nvendor_faq_index_index')"
bin/magento config:set --scope=stores --scope-code=default lynx_ssr/general/customer_groups 0
bin/magento cache:clean config full_page
```

**Test Connection** in **Stores › Configuration › Lynx › SSR** calls `/health`. Renderer URL is scheme, host and port
only; `/render` and `/health` are appended.

## 4. Check the output

```bash
curl -s https://store.test/faq | grep -o 'data-server-rendered="true"'
```

No match means the browser drew the page. Check, in order: route in **Routes** (full action name, `fnmatch`), a cached
page from before SSR was on (clean `full_page`), guest group selected, the layout does not set `ssr` false,
`var/log/system.log` for `Lynx SSR:` warnings, the renderer's stderr.

| Log message | Cause |
|---|---|
| `Lynx SSR: renderer did not answer` | Connection refused, DNS, or timeout (**Timeout (ms)**; connect capped at 1 s) |
| `Lynx SSR: renderer refused the page` | Non-200, body not JSON, or `NOT_FOUND` (Renderer URL with a path); `error` carries the renderer's message |
| `Lynx SSR: page cannot be encoded` | Page props not JSON-encodable |

## 5. Render one page directly (debugging a component)

Take the page object and store config from a client-rendered response (SSR off for that request, e.g. a route not in
Routes or a signed-in session) and post them to the renderer; a failed render answers `500` with
`{ error, component, url, stack }`:

```bash
curl -s https://store.test/faq > /tmp/page.html
python3 - /tmp/page.html > /tmp/envelope.json <<'EOF'
import html, json, re, sys
src = open(sys.argv[1], encoding='utf-8').read()
page = re.search(r'<script type="application/json" data-page="lynx">(.*?)</script>', src, re.S).group(1)
conf = re.search(r'<script type="application/json" id="lynx-store-config">(.*?)</script>', src, re.S).group(1)
envelope = json.loads(page)
envelope['storeConfig'] = json.loads(conf)
print(json.dumps(envelope))
EOF
curl -s -X POST -H 'Content-Type: application/json' --data @/tmp/envelope.json http://127.0.0.1:13714/render | head -c 2000
```

The renderer removes `storeConfig` before rendering; components read it through `config()`.

## 6. Hydration

- Open the page in a browser with the console open. A mismatch shows as a hydration error (React) or "Hydration
  completed but contains mismatches" (Vue production build). Find the node by comparing the server HTML
  (JavaScript disabled, or `view-source:`) with the DOM after load.
- Throttle the network (DevTools) with a cold cache: content that disappears and reappears after load means a node
  re-rendered before its chunk arrived (entry-file behavior, SKILL step 7).

## 7. Worker (optional)

```bash
cd <theme>/lynx
npm run build:ssr:worker
npx wrangler login                 # once
npm run deploy:ssr:worker          # prints https://<name>.<subdomain>.workers.dev
npm run tail:ssr:worker            # live logs
```

Set Renderer URL to the Worker origin (no `/render`) and raise Timeout (3000–4000 ms) for the extra network hop.
A render needs more CPU than Workers Free allows. In a child theme, rename the Worker in `wrangler.toml` (`name`).

## 8. Clean up

Restore the `lynx_ssr/general/*` values you noted, `bin/magento cache:clean config full_page`, stop the renderer.
