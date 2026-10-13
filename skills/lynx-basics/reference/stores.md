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

React (jotai), `view/frontend/lynx/react/src/stores/noticeStore.ts`: an atom for the state, actions bound to a jotai
store (usable outside React too), and a hook for components:

```ts
import { atom, useAtomValue, useStore } from 'jotai'
import { useMemo } from 'react'
import type { JotaiStore } from 'OOCS_LynxTheme/types/store'

type NoticeState = { dismissed: boolean }

export const noticeAtom = atom<NoticeState>({ dismissed: false })

export function noticeStore(store: JotaiStore) {
    return {
        get dismissed(): boolean {
            return store.get(noticeAtom).dismissed
        },
        dismiss(): void {
            store.set(noticeAtom, { dismissed: true })
        },
    }
}

export function useNoticeStore() {
    const store = useStore()
    // Subscribed, so the component renders again when the state changes
    useAtomValue(noticeAtom)
    return useMemo(() => noticeStore(store), [store])
}
```

- Pinia store ids must be unique across modules: prefix with your vendor (`vendorNotice`).
- The browser has one Pinia / jotai store per tab; the SSR renderer creates one per request. Never keep per-visitor
  state in a module-level variable.
- React: in callbacks after `await`, read fresh state through store actions (bound to the jotai store), not from the
  render closure.

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
- Never add Vue, React, Inertia.js, Pinia or jotai: they come from the theme.
- A dependency that touches `window` when imported must be imported dynamically inside `onMounted` / `useEffect`
  (skill lynx-ssr). For the Cloudflare Worker SSR build, it must not need Node built-ins.
