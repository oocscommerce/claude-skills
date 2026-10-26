# Deferred, optional and lazy providers

## Flow

```
Page response:
  layout.content[n] = { name: 'vendor_recommendations', props: { limit: 8, _key: 'vendor_recommendations' } }
  deferred groups announce 'vendor_recommendations'        <- deferred only; optional is not announced

Follow-up partial request (Inertia, only: ['vendor_recommendations']):
  provider->resolve({ limit: 8 })  (+ resolveSample() on a sample preview page)
  -> page.props.vendor_recommendations = { limit: 8, products: [...] }

DynamicNode: props = { ...node.props, ...page.props[node.name] }
```

- Registered by `OOCS\LynxComponent\Model\LayoutResolver::resolveProps()` through the page's defer registrar;
  `optional()` is asked first and wins.
- The key is the component's `code`, also given to the node as `_key`.
- The partial request goes through the full page cache on its own (Lynx keeps it apart from the page with its
  `_lynx` / `_lynx_only` parameters), so tags must be added inside `resolve()`.

## Merchant toggle: `deferred` (fetched right after load)

```graphql
"On: fetched right after the page loads, so the page is sent sooner. Off: sent with the page and visible to search engines"
deferred: Boolean! @setting(label: "Load after the page", default: true)
products: [VendorRecommendationsProduct!]! @optional
```

```php
public function deferred(array $props): bool
{
    return !empty($props['deferred']);
}
```

The component needs no code for this: it renders a placeholder while its data prop (`@optional` in the `.graphqls`) is
`undefined`.

```vue
<div v-if="products === undefined" class="l-skeleton h-64 w-full" aria-busy="true"></div>
<ul v-else>…</ul>
```

Built-in: Related / Up-sell / Cross-sell Products (`OOCS\LynxCatalog\Provider\Product\Related`, `Upsell`,
`Crosssell`).

## The `lazy` convention (fetched when shown)

1. Toggle setting and the data, in `etc/components/Specs.graphqls`:

```graphql
type VendorSpecs @component(
  code: "vendor_specs"
  template: "Vendor_Module::Components/Specs"
  provider: "Vendor\\Module\\Provider\\Specs"
  label: "Specifications"
  zones: ["content"]
  availableOn: ["catalog_product_view*"]
) {
  "Off: sent with the page and visible to search engines. On: fetched only when the block appears, for example when its tab is opened"
  lazy: Boolean! @setting(label: "Load when shown", default: false)
  rows: [VendorSpecsRow!]! @optional
}

type VendorSpecsRow {
  label: String!
  value: String!
}
```

2. Provider (`OOCS\LynxCatalog\Provider\Product\Specs` is the reference implementation):

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Provider;

use Magento\Catalog\Model\Product;
use Magento\Framework\Registry;
use OOCS\LynxComponent\Model\AbstractComponentProvider;
use OOCS\LynxComponent\Model\PageCacheIdentities;

class Specs extends AbstractComponentProvider
{
    /**
     * @param Registry $registry
     * @param PageCacheIdentities $pageCacheIdentities
     */
    public function __construct(
        private readonly Registry $registry,
        private readonly PageCacheIdentities $pageCacheIdentities,
    ) {
    }

    /**
     * Left out of the page while the block is lazy
     *
     * @param array $props
     * @return bool
     */
    public function optional(array $props): bool
    {
        return !empty($props['lazy']);
    }

    /**
     * Rows of the viewed product
     *
     * @param array $props
     * @return array
     */
    public function resolve(array $props): array
    {
        /** @var Product|null $product */
        $product = $this->registry->registry('current_product');
        if (!$product) {
            return $props;
        }

        // The lazy request is cached on its own, so it carries the product tags itself
        if (!empty($props['lazy'])) {
            $this->pageCacheIdentities->add([Product::CACHE_TAG, Product::CACHE_TAG . '_' . $product->getId()]);
        }

        $rows = [];
        foreach ($product->getAttributes() as $attribute) {
            if (!$attribute->getIsVisibleOnFront()) {
                continue;
            }
            $value = $attribute->getFrontend()->getValue($product);
            if ($value === null || $value === '' || $value === false) {
                continue;
            }
            $rows[] = [
                'label' => $this->toPlainText((string) $attribute->getStoreLabel()),
                'value' => $this->toPlainText(\is_array($value) ? implode(', ', $value) : (string) $value),
            ];
        }
        $props['rows'] = $rows;

        return $props;
    }
}
```

3. Vue (`view/frontend/lynx/vue/src/Components/Specs.vue`):

```vue
<script setup lang="ts">
import { onMounted } from 'vue'
import { router } from '@inertiajs/vue3'

const props = withDefaults(defineProps<Lynx.VendorSpecs>(), {
    rows: undefined,
    _key: '',
})

onMounted(() => {
    if (props.lazy && props.rows === undefined && props._key) {
        router.reload({ only: [props._key], preserveUrl: true })
    }
})
</script>

<template>
    <div v-if="rows === undefined" class="grid gap-3" aria-busy="true">
        <span v-for="n in 3" :key="n" class="l-skeleton block h-4 w-full"></span>
    </div>
    <dl v-else-if="rows.length" class="divide-y divide-base-300">
        <div v-for="row in rows" :key="row.label" class="flex gap-6 py-3.5">
            <dt class="w-32 shrink-0 font-semibold text-base-content/70">{{ row.label }}</dt>
            <dd class="text-base-content">{{ row.value }}</dd>
        </div>
    </dl>
</template>
```

Test a prop that `resolve()` always sets (`rows`), not a setting. A block that may have nothing to show returns
`enabled: false` and is not optional in that case, so Tabs hides its tab from the first paint.

## Limits

- One key per component code: a component placed twice on a page with deferred/optional data shares one result.
- Not in SSR HTML; one extra request per group / reload.
- `labels` of a deferred/optional provider arrive with the data.
