# Props, partial reloads and navigation

All PHP below uses `OOCS\LynxInertia\Service\InertiaService` (a shared instance per request). Methods that exist:
`setPage($component, $props, $url)`, `share($key, $value)`, `defer(callable, $group = 'default')`,
`optional(callable)`, `encryptHistory()`, `clearHistory()`, `hasPage()`, `resolvePage(?array $only, ?string $partialComponent)`,
`setVersion()` / `getVersion()`.

## Shared props

Merged into every page of the request, layout-driven or not. Call `share()` before the page is resolved: a `before`
plugin on `resolvePage()` always runs in time.

`etc/frontend/di.xml`:

```xml
<type name="OOCS\LynxInertia\Service\InertiaService">
    <plugin name="vendor_module_share_store_hours" type="Vendor\Module\Plugin\ShareStoreHours"/>
</type>
```

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin;

use OOCS\LynxInertia\Service\InertiaService;
use Vendor\Module\Model\StoreHours;

class ShareStoreHours
{
    /**
     * @param StoreHours $storeHours
     */
    public function __construct(
        private readonly StoreHours $storeHours,
    ) {
    }

    /**
     * Share today's opening hours with every page
     *
     * @param InertiaService $subject
     * @return null
     */
    public function beforeResolvePage(InertiaService $subject): null
    {
        $subject->share('storeHours', $this->storeHours->getToday());

        return null;
    }
}
```

Lynx shares: `auth.isLoggedIn` (from the HTTP context FPC varies on), and on Inertia responses `title` (applied to
`document.title`) and `bodyClass` (applied to `#html-body`). Shared values are cached with the page: same for every
visitor in the cache variation only.

Read them:

```ts
// Vue
import { usePage } from '@inertiajs/vue3'
const page = usePage<{ storeHours?: string }>()
// page.props.storeHours
```

```tsx
// React
import { usePage } from '@inertiajs/react'
const { storeHours } = usePage<{ storeHours?: string }>().props
```

## Deferred and optional props (controller-declared page)

```php
$this->inertiaService->setPage('Vendor_Faq::Ask', [
    'heading' => __('Ask a question'),
    // fetched together right after the page appears
    'popular' => $this->inertiaService->defer(fn () => $this->getPopularQuestions()),
    'recent'  => $this->inertiaService->defer(fn () => $this->getRecentQuestions()),
    // fetched in a request of its own
    'stats'   => $this->inertiaService->defer(fn () => $this->getStats(), 'secondary'),
    // never sent until a partial reload names it
    'archive' => $this->inertiaService->optional(fn () => $this->getArchive()),
], $this->request->getRequestUri());
```

Callbacks run only when their prop is resolved. Deferred prop requests carry `_lynx_only` so FPC caches them apart.

```vue
<script setup lang="ts">
import { Deferred } from '@inertiajs/vue3'

defineProps<{ popular?: string[] }>()
</script>

<template>
    <Deferred data="popular">
        <template #fallback>
            <div class="l-skeleton h-24 rounded-box" />
        </template>
        <ul>
            <li v-for="question in popular" :key="question">{{ question }}</li>
        </ul>
    </Deferred>
</template>
```

```tsx
import { Deferred } from '@inertiajs/react'

export default function Popular({ popular }: { popular?: string[] }) {
    return (
        <Deferred data="popular" fallback={<div className="l-skeleton h-24 rounded-box" />}>
            <ul>
                {popular?.map(question => (
                    <li key={question}>{question}</li>
                ))}
            </ul>
        </Deferred>
    )
}
```

On layout-driven pages, defer through the component's provider instead (`deferred()` / `deferredGroup()` /
`optional()` of `AbstractComponentProvider`, skill **lynx-provider**); the component then gets a `_key` prop.

## Partial reload

```ts
import { router } from '@inertiajs/vue3' // or '@inertiajs/react'

router.reload({ only: ['archive'] })      // controller-declared page prop
router.reload({ only: [props._key] })     // provider-deferred component data on a layout-driven page
```

The server resolves only the named props, optional ones included.

## Navigation

```vue
<script setup lang="ts">
import { Link, router } from '@inertiajs/vue3'

const search = (q: string) => router.get('/catalogsearch/result/', { q })
</script>

<template>
    <Link :href="url">{{ label }}</Link>
</template>
```

```tsx
import { Link, router } from '@inertiajs/react'

const search = (q: string) => router.get('/catalogsearch/result/', { q })

export default function FaqLink({ url, label }: { url: string; label: string }) {
    return <Link href={url}>{label}</Link>
}
```

What Lynx adds on the client:

- Every Inertia GET gets `_lynx=<base64 of path>` (and `_lynx_only` for partial reloads) so FPC keys JSON apart from
  HTML; the server strips them from the URL it returns.
- `Link`/`router` to another origin, `mailto:`, `tel:` is handed to the browser.
- Checkout and pages handed to a standard theme (Theme Fallback) answer an Inertia visit with `409` +
  `X-Inertia-Location` and load in full; use a plain `<a>` for those links.
- After a visit to a different path, focus moves to `<main id="page-content">`; query-only changes keep focus.
- On every navigation the client applies `title` and `bodyClass`, shows server flash messages and tracks `page_view`.

## Asset version

The Inertia version is Magento's static deployment version. A stale tab gets `409` on its next visit and reloads.
`InertiaService::setVersion()` overrides it if a module needs its own scheme.

## Forms on the page

Posting from a page (Inertia `router.post` with `forceFormData` + `form_key`, native posts, reCAPTCHA, messages):
see skill **lynx-forms-analytics**.
