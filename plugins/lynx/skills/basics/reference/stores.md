# Stores and npm dependencies

## Store templates

Stores are Pinia. Feature stores live in your module
(`view/frontend/lynx/vue/src/stores/<name>Store.ts`) and get their own chunk. Only a store every page needs
goes in `stores/global/` (bundled into the always-loaded chunk).

`view/frontend/lynx/vue/src/stores/noticeStore.ts`:

```ts
import { defineStore } from 'pinia'

export const useNoticeStore = defineStore('vendorNotice', {
    state: () => ({ dismissed: false }),
    actions: {
        dismiss(): void {
            this.dismissed = true
        },
    },
})
```

- Store ids must be unique across modules: prefix with your vendor (`vendorNotice`).
- The browser has one Pinia per tab; the SSR renderer creates one per request. Never keep per-visitor
  state in a module-level variable.

## Using the global stores

```ts
import { useMessagesStore } from 'OOCS_LynxTheme/stores/global/messagesStore'
const messages = useMessagesStore()
messages.add('error', response.message)
```

`customerDataStore` holds Magento customer data sections (IndexedDB `lynx_customer_data`); after your own POST call
`invalidate([...sections])` with the sections the action invalidates. `formKeyStore.ensure()` before a POST.
`analyticsStore.track(event, params)` emits on the analytics bus; trackers subscribe with `addTracker(fn)`.

## npm dependencies of a module

List them in a `package.json` next to `src/` (`view/frontend/lynx/vue/package.json`):

```json
{
  "name": "vendor-module-vue",
  "version": "1.0.0",
  "private": true,
  "dependencies": {
    "some-sdk": "4.2.0"
  }
}
```

- `lynx:theme:setup` merges them into each theme's `lynx/.lynx/package.json` and installs them into the theme's
  `node_modules`; module code imports them from there. Never run `npm install` in the module.
- Pin exact versions. Two modules, or a module and the theme, pinning one package to different versions stop setup.
- Never add Vue, Inertia.js or Pinia: they come from the theme.
- A dependency that touches `window` when imported must be imported dynamically inside `onMounted`
  (skill ssr). For the Cloudflare Worker SSR build, it must not need Node built-ins.
