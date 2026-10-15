# Stores and npm dependencies

## Store templates

Same file name and exports on both frameworks. Feature stores live in your module
(`view/frontend/lynx/<vue|react>/src/stores/<name>Store.ts`) and get their own chunk. Only a store every page needs
goes in `stores/global/` (bundled into the always-loaded chunk).

Vue (Pinia), `view/frontend/lynx/vue/src/stores/noticeStore.ts`:

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

React, `view/frontend/lynx/react/src/stores/noticeStore.ts`: `defineStore` from `OOCS_LynxTheme/stores/defineStore`
(Zustand underneath) takes Pinia's options shape; an action reads `this` and writes `this.$set({ … })`:

```ts
import { defineStore } from 'OOCS_LynxTheme/stores/defineStore'

export const useNoticeStore = defineStore('vendorNotice', {
    state: () => ({ dismissed: false }),
    actions: {
        dismiss(): void {
            this.$set({ dismissed: true })
        },
    },
})

// component: const notice = useNoticeStore()          (state + actions)
//            const dismissed = useNoticeStore(s => s.dismissed)   (renders only when it changes)
// outside React: useNoticeStore.of(registry).getState().dismiss()
```

- Store ids must be unique across modules, Pinia and React alike: prefix with your vendor (`vendorNotice`).
- The browser has one Pinia / store registry per tab; the SSR renderer creates one per request. Never keep per-visitor
  state in a module-level variable.
- React: `actions` holds functions only; constants and objects changed in place (a Map) go in `state`. `this` in an
  action is the state as it is now, also after `await`; the object `useX()` returned is the render's snapshot, so a
  later callback (effect cleanup, timer) reads `useX.of(useRegistry()).getState()` or calls an action.
- React getters: compute them in a hook around the store (as `useCustomerDataStore` does); a store nobody renders
  from picks its actions with `useShallow`, so events do not re-render.

## Using the global stores

```ts
// Vue
import { useMessagesStore } from 'OOCS_LynxTheme/stores/global/messagesStore'
const messages = useMessagesStore()
messages.add('error', response.message)

// React
import { useMessagesStore } from 'OOCS_LynxTheme/stores/global/messagesStore'
const messages = useMessagesStore()
messages.add('error', response.message)
```

`customerDataStore` holds Magento customer data sections (IndexedDB `lynx_customer_data`); after your own POST call
`invalidate([...sections])` with the sections the action invalidates. `formKeyStore.ensure()` before a POST.
`analyticsStore.track(event, params)` emits on the analytics bus; trackers subscribe with `addTracker(fn)`.

## npm dependencies of a module

Ship a `package.json` next to `src/` (`view/frontend/lynx/vue/package.json`, `…/react/package.json`):

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

- `lynx:workspace:setup` adds the folder as an npm workspace; one `npm install` at the root serves every theme.
- `name` must be unique across all workspaces. Pin exact versions.
- Never add Vue, React, Inertia.js, Pinia or Zustand: they come from the theme.
- A dependency that touches `window` when imported must be imported dynamically inside `onMounted` / `useEffect`
  (skill ssr). For the Cloudflare Worker SSR build, it must not need Node built-ins.
