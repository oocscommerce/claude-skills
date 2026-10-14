---
name: testing
description: Writes and runs Vitest tests for Lynx storefront code - Vue and React components, stores and utils of a module or theme override - in the `test/` folder beside `src/`, with the `@oocs/lynx-vue/test` / `@oocs/lynx-react/test` helpers (`mountNode`, `renderSsr`, `hydrate`), realistic props from the PHP provider, network and Inertia stubbed, an SSR + hydration check for components on server-rendered pages, and `bin/magento lynx:static:test`. Use when a developer says "write tests for my Lynx component", "test this store", "add a hydration test", "my Lynx test fails", "run the Lynx tests", or "test my theme override". Use with the basics skill.
---
# Testing Lynx frontend code

Produces test files for the Vue and React editions of a module's (or a theme override's) components, stores and
utils, and runs them per theme.

## Before you start

- Read **basics**. Tests run on the theme's own Vite build: imports resolve through theme fallback exactly as in the
  storefront, so a test runs against the file that theme ships, its override included.
- Nothing to install: Vitest, happy-dom, Vue Test Utils and Testing Library come with the build tools
  (`@oocs/lynx-vue`, `@oocs/lynx-react`) that `bin/magento lynx:workspace:setup` installs.
- Read the source you test, and for a component also its PHP provider (`resolve()`, `resolveSample()`) and its
  settings in `etc/di.xml`: they give the real prop shapes, labels and URLs.

## Steps

### 1. Place the test file

Mirror the source under `test/` beside `src/`, named `<File>.test.ts` (React: `.test.tsx` only if it holds JSX;
`createElement` in a `.test.ts` works too).

| Source | Test |
|---|---|
| `view/frontend/lynx/vue/src/Components/WishlistButton.vue` | `view/frontend/lynx/vue/test/Components/WishlistButton.test.ts` |
| `view/frontend/lynx/react/src/stores/wishlistStore.ts` | `view/frontend/lynx/react/test/stores/wishlistStore.test.ts` |
| `<theme>/Vendor_Module/lynx/vue/src/Components/WishlistButton.vue` (override) | `<theme>/Vendor_Module/lynx/vue/test/Components/WishlistButton.test.ts` |
| `<theme>/lynx/src/...` | `<theme>/lynx/test/...` |

A test runs only in themes that build the file it sits beside: once a theme overrides a file, the module's test for it
no longer runs there, and the theme's own test (if any) does. Files under `test/` never reach the bundle or `pub/static`.

Import what you test by module name, never by relative path:

```ts
import WishlistButton from 'Vendor_Module/Components/WishlistButton.vue'   // React: 'Vendor_Module/Components/WishlistButton'
```

### 2. Mount the way a page does

```ts
import { hydrate, mountNode, renderSsr } from '@oocs/lynx-vue/test'   // React: '@oocs/lynx-react/test'
```

| Helper | Returns |
|---|---|
| `mountNode(Component, { props, slots/children, page, config })` | Vue Test Utils wrapper (Vue), Testing Library result (React) |
| `renderSsr(Component, options)` | Server HTML, rendered with `window`, `document` and every browser-only global removed |
| `hydrate(Component, options)` | `{ html, el, warnings }`: server HTML hydrated in the browser, every warning/error logged |

- `props`: what the provider returns for the node (settings + data, labels included).
- `page`: shared Inertia page props as `usePage().props` reads them (`auth`, `preview`…).
- `config`: the store config read by `config()` of `OOCS_LynxTheme/utils/storeConfig` (`priceFormat`,
  `currencyCode`, `cookie`, `sections`…). Set every key the code reads; never rely on defaults by accident.
- `slots` (Vue) / `children` (React) for container components.

After each test, everything the helpers mounted is unmounted and cookies, storage, the document, mocks
(`vi.restoreAllMocks`), stubbed globals and fake timers are reset. Do not add your own `afterEach` for that.

### 3. Test behaviour, the same cases in both editions

Write one `describe` per file and the **same `it` names** in the Vue and React files. Cover what the code does:

- rendering from props, and the empty or missing-data state;
- each interaction: click, input, keyboard, submit, with what it sends (payload, `form_key`, URL);
- the store updates and Inertia visits that follow;
- the editor preview (`page: { preview: true }`) when the component behaves differently there.

Assert on text, attributes, `aria-*`, payloads and calls, never on CSS class strings. Stub the outside world:

```ts
const fetch = vi.fn(async () => ({ ok: true, json: async () => ({ success: true }) }))
vi.stubGlobal('fetch', fetch)
const post = vi.spyOn(router, 'post').mockImplementation(() => {})   // router from '@inertiajs/vue3' / '@inertiajs/react'
```

Patterns for both editions (forms, stores, timers, module state): [reference/patterns.md](reference/patterns.md).

### 4. Add the SSR case, for components on server-rendered pages

```ts
it('server-renders and hydrates the same markup', async () => {
    const { html, warnings } = await hydrate(WishlistButton, { props })

    expect(html).toContain('Add to Wish List')
    expect(warnings).toEqual([])
})
```

It catches browser globals used during render and markup that differs between server and browser (skill **ssr**).
Leave it out when the component only appears on pages Lynx never renders on the server: cart, checkout, checkout
success/failure, customer account and login pages, PageBuilder preview only (`PreviewBridge`-like) components.

### 5. Test stores and utils directly

```ts
// Vue
beforeEach(() => setActivePinia(createPinia()))
const store = useWishlistStore()

// React
const store = wishlistStore(createStore())   // createStore from 'jotai'
```

For a React hook, `renderHook(() => useWishlistStore(), { wrapper })` with a jotai `Provider` wrapper.

## Rules

- Never change `src/` to make a test pass. When a test exposes a bug (wrong payload, crash, hydration mismatch, Vue
  and React behaving differently), report it with the file and line and leave that case out until it is fixed: no
  failing or skipped tests committed.
- Each test file is self-contained: no shared helper files; small helpers inside the file are fine.
- Labels and messages come from props or config in the test, as in production; never assert on text the component
  should not hardcode.
- No `eslint-disable`, `@ts-ignore`, snapshots. Format with Prettier (`npm run format` at the project root).

## Check

1. `bin/magento lynx:static:test` (every theme set up), or `bin/magento lynx:static:test OOCS_LynxVue` for one.
   Inside a theme's `lynx/`: `npx vitest run <part of the path>` for one file, `npx vitest` to watch.
2. Every new file passes in each theme you ship, with no `stderr` output from your tests (Vue warnings,
   React `act(...)` warnings): fix the test or report the source.
3. `npm run format`, and `git status` shows only files under `test/`.

## Reference

- [reference/patterns.md](reference/patterns.md): Vue and React test patterns and test-environment gotchas
- https://docs.oocscommerce.com/lynx/developer/testing
