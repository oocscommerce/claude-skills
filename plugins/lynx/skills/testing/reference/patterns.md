# Test patterns

## A form posting through fetch

```ts
import { mountNode } from '@oocs/module-lynx-theme/test'
import { flushPromises } from '@vue/test-utils'
import WishlistButton from 'Vendor_Module/Components/WishlistButton.vue'
import { describe, expect, it, vi } from 'vitest'

const props = { product_id: 42, urls: { add: '/wishlist/index/add/' }, labels: { add: 'Add to Wish List' } }

describe('WishlistButton', () => {
    it('posts the product with the form key', async () => {
        document.cookie = 'form_key=test-form-key; path=/'
        const fetch = vi.fn(async () => ({ ok: true, json: async () => ({ success: true }) }))
        vi.stubGlobal('fetch', fetch)
        const wrapper = mountNode(WishlistButton, { props })

        await wrapper.find('button').trigger('click')
        await flushPromises()

        const [url, init] = fetch.mock.calls[0]
        expect(url).toBe('/wishlist/index/add/')
        expect(new URLSearchParams(String(init.body)).get('product')).toBe('42')
        expect(new URLSearchParams(String(init.body)).get('form_key')).toBe('test-form-key')
    })
})
```

## An Inertia visit

```ts
import { router } from '@inertiajs/vue3'

const visit = vi.spyOn(router, 'visit').mockImplementation(() => {})
// … click a sort option …
expect(String(visit.mock.calls[0][0])).toContain('product_list_order=price')
```

For `router.post(url, data, options)`, call `options.onSuccess?.()` / `options.onError?.()` and
`options.onFinish?.()` in the mock to drive the component through each outcome:

```ts
vi.spyOn(router, 'post').mockImplementation((_url, _data, options) => {
    options?.onError?.({ qty: 'Invalid' })
    options?.onFinish?.()
})
```

## Shared stores

Read the store the mounted component uses:

```ts
// The helpers make the page's pinia active
const wrapper = mountNode(MiniCart, { props })
useCustomerDataStore().sections = { cart: { summary_count: 2 } }
await nextTick()
```

Store config the store reads goes through `setStoreConfig({ … })` from `OOCS_LynxTheme/utils/storeConfig` when testing
a store without mounting.

## Timers

```ts
vi.useFakeTimers()
const wrapper = mountNode(AnnouncementBar, { props })
await vi.advanceTimersByTimeAsync(4000)
```

Fake timers are reset after each test.

## Module-level state

Code that keeps state at module level (a script loaded once per page, a counter of instances) carries it from one test
to the next. Load fresh copies in each test, helpers included, and unmount what you mounted yourself:

```ts
const mounted: { unmount: () => void }[] = []

async function load() {
    vi.resetModules()
    const { mountNode } = await import('@oocs/module-lynx-theme/test')
    const { default: Widget } = await import('Vendor_Module/Components/Widget.vue')
    return { mountNode, Widget }
}

afterEach(() => {
    for (const wrapper of mounted.splice(0)) wrapper.unmount()
})
```

Copies loaded this way are not the ones the shared cleanup knows, hence the local `afterEach`.

## Nodes rendered through DynamicNode

Components resolved from the registry load asynchronously. New nodes are held back up to 300 ms when the page has no
server markup (always the case in a test): wait with `await vi.waitFor(() => expect(…))`, not `flushPromises`.

## Test-environment gotchas

| Symptom | Cause and fix |
|---|---|
| `window.location` is `/` right after mounting | Inertia's App writes the page URL on mount. Set the URL you need with `history.replaceState` after `await new Promise(r => setTimeout(r))`. |
| `confirm is not a function` | The test DOM has no `window.confirm`: `vi.stubGlobal('confirm', vi.fn(() => true))`. |
| `IntersectionObserver is not defined` | Stub it: `vi.stubGlobal('IntersectionObserver', class { observe() {} unobserve() {} disconnect() {} })`. |
| Sizes, positions, scrolling are all 0 | No layout engine: do not assert geometry; stub `scrollIntoView`/`scrollBy` if the code calls them. |
| An iframe or external script "failed to load" on stderr | The test DOM loads no frames or remote scripts: assert the `src` attribute, never the load. |
| A Vue slot's inputs reset on every render | Props built inline in a render function are new on each render: build the props object once. |
| A missing Boolean prop is `false`, not `undefined` | Vue turns a missing Boolean prop into `false`: test the missing case with that in mind. |
