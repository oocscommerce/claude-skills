# Provider examples

## Labels only

A component with text but no data (built-in: `OOCS\LynxThemeComponent\Provider\Page\BackToTop`).

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Provider;

use OOCS\LynxComponent\Model\AbstractComponentProvider;

class BackToTop extends AbstractComponentProvider
{
    /**
     * Strings the component shows
     *
     * @param array $props
     * @return array
     */
    public function resolve(array $props): array
    {
        $props['labels'] = [
            'backToTop' => (string) __('Back to top'),
        ];

        return $props;
    }
}
```

Component side:

```vue
<script setup lang="ts">
const props = withDefaults(defineProps<{ labels?: Record<string, string> }>(), { labels: () => ({}) })
</script>

<template>
    <button class="l-btn l-btn-primary" :aria-label="labels.backToTop">…</button>
</template>
```

```tsx
type Props = { labels?: Record<string, string> }

export default function BackToTop({ labels = {} }: Props) {
    return <button className="l-btn l-btn-primary" aria-label={labels.backToTop}>…</button>
}
```

Values only the browser knows stay as placeholders: `__('Items %1 to %2 of %3 total')` in PHP,
`(labels.itemsOf ?? '').replace('%1', String(from))…` in the component.

## Images, links and translations

Built-in equivalent: `OOCS\LynxThemeComponent\Provider\Page\Image`.

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Provider;

use OOCS\LynxComponent\Helper\LocalizedText;
use OOCS\LynxComponent\Model\AbstractComponentProvider;
use OOCS\LynxThemeComponent\Helper\MediaUrl;
use OOCS\LynxThemeComponent\Helper\StoreLink;

class PromoTile extends AbstractComponentProvider
{
    /**
     * @param LocalizedText $localizedText
     * @param MediaUrl $mediaUrl
     * @param StoreLink $storeLink
     */
    public function __construct(
        private readonly LocalizedText $localizedText,
        private readonly MediaUrl $mediaUrl,
        private readonly StoreLink $storeLink,
    ) {
    }

    /**
     * Store-specific text, image URL and link
     *
     * @param array $props
     * @return array
     */
    public function resolve(array $props): array
    {
        $props['title'] = $this->localizedText->get($props['title']);
        $props['image'] = $this->mediaUrl->url((string) $props['image']);
        $props['url'] = $this->storeLink->path((string) $props['url']);

        return $props;
    }
}
```

## Catalog data with cache tags

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Provider;

use Magento\Catalog\Model\Product;
use Magento\Catalog\Model\ResourceModel\Product\CollectionFactory;
use OOCS\LynxComponent\Helper\LocalizedText;
use OOCS\LynxComponent\Model\AbstractComponentProvider;
use OOCS\LynxComponent\Model\PageCacheIdentities;

class FeaturedProducts extends AbstractComponentProvider
{
    /**
     * @param CollectionFactory $collectionFactory
     * @param PageCacheIdentities $pageCacheIdentities
     * @param LocalizedText $localizedText
     */
    public function __construct(
        private readonly CollectionFactory $collectionFactory,
        private readonly PageCacheIdentities $pageCacheIdentities,
        private readonly LocalizedText $localizedText,
    ) {
    }

    /**
     * Below the fold: sent after the page
     *
     * @param array $props
     * @return bool
     */
    public function deferred(array $props): bool
    {
        return !empty($props['deferred']);
    }

    /**
     * Product cards for the configured SKUs
     *
     * @param array $props
     * @return array
     */
    public function resolve(array $props): array
    {
        $skus = array_filter(array_map('trim', explode(',', (string) $props['skus'])));
        $collection = $this->collectionFactory->create()
            ->addAttributeToSelect(['name', 'price', 'small_image', 'url_key'])
            ->addAttributeToFilter('sku', ['in' => $skus ?: ['']])
            ->setPageSize((int) $props['limit']);

        $identities = [Product::CACHE_TAG];
        $props['products'] = [];
        foreach ($collection as $product) {
            $identities[] = Product::CACHE_TAG . '_' . $product->getId();
            $props['products'][] = [
                'id' => (int) $product->getId(),
                'name' => $this->toPlainText((string) $product->getName()),
                'url' => (string) $product->getProductUrl(),
            ];
        }
        $this->pageCacheIdentities->add($identities);

        $props['title'] = $this->localizedText->get($props['title']);
        $props['labels'] = ['addToCart' => (string) __('Add to Cart')];

        return $props;
    }
}
```

(Visibility, status, stock and price formatting are omitted; copy the logic of the matching core block or of
`OOCS\LynxCatalog\Provider\Product\Carousel` for a real product list.)

## Expensive shared data: `ComponentCache`

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Provider;

use Magento\Framework\Serialize\SerializerInterface;
use Magento\Store\Model\Store;
use Magento\Store\Model\StoreManagerInterface;
use OOCS\LynxComponent\Model\AbstractComponentProvider;
use OOCS\LynxComponent\Model\Cache\Type\ComponentCache;

class StoreList extends AbstractComponentProvider
{
    private const CACHE_LIFETIME = 86400;

    /**
     * @param StoreManagerInterface $storeManager
     * @param ComponentCache $cache
     * @param SerializerInterface $serializer
     */
    public function __construct(
        private readonly StoreManagerInterface $storeManager,
        private readonly ComponentCache $cache,
        private readonly SerializerInterface $serializer,
    ) {
    }

    /**
     * Store list, cached per store view
     *
     * @param array $props
     * @return array
     */
    public function resolve(array $props): array
    {
        $key = 'vendor_store_list_' . $this->storeManager->getStore()->getCode();
        $cached = $this->cache->load($key);
        if ($cached !== false) {
            $props['stores'] = $this->serializer->unserialize($cached);

            return $props;
        }

        $props['stores'] = $this->buildList();
        $this->cache->save($this->serializer->serialize($props['stores']), $key, [Store::CACHE_TAG], self::CACHE_LIFETIME);

        return $props;
    }

    /**
     * The list itself
     *
     * @return array
     */
    private function buildList(): array
    {
        return [];
    }
}
```

`ComponentCache` is the `lynx_component` cache type ("Lynx Component Data"), a tag-scoped frontend: merchants can
clean it in System > Cache Management, and entity tags clear entries when the entity changes. Key by everything the
data depends on (store, currency, customer group); never cache per-visitor data.

## Sample-data preview

Built-in: `OOCS\LynxThemeComponent\Provider\Checkout\Success` (order success page).

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Provider;

use Magento\Checkout\Model\Session as CheckoutSession;
use Magento\Framework\UrlInterface;
use OOCS\LynxComponent\Api\SampleProviderInterface;
use OOCS\LynxComponent\Model\AbstractComponentProvider;

class OrderNote extends AbstractComponentProvider implements SampleProviderInterface
{
    /**
     * @param CheckoutSession $checkoutSession
     * @param UrlInterface $urlBuilder
     */
    public function __construct(
        private readonly CheckoutSession $checkoutSession,
        private readonly UrlInterface $urlBuilder,
    ) {
    }

    /**
     * The order just placed
     *
     * @param array $props
     * @return array
     */
    public function resolve(array $props): array
    {
        $order = $this->checkoutSession->getLastRealOrder();
        $props['order_number'] = (string) $order->getIncrementId();
        $props['urls'] = [
            'order' => $this->urlBuilder->getUrl('sales/order/view', ['order_id' => $order->getEntityId()]),
        ];
        $props['labels'] = ['orderNumberIs' => (string) __('Your order number is: %1.')];

        return $props;
    }

    /**
     * A sample order for the editor preview
     *
     * @param array $resolved
     * @param array $props
     * @return array
     */
    public function resolveSample(array $resolved, array $props): array
    {
        $resolved['order_number'] = 'SAMPLE-000000123';
        $resolved['urls']['order'] = '#';

        return $resolved;
    }
}
```

When it runs: the request is a PageBuilder preview of a page declared with `sample_data` `true`:

```xml
<type name="OOCS\LynxPageBuilder\Model\PreviewPagebuilder">
    <arguments>
        <argument name="pages" xsi:type="array">
            <item name="checkout_success" xsi:type="array">
                <item name="sort_order" xsi:type="number">82</item>
                <item name="label" xsi:type="string">Order success</item>
                <item name="group" xsi:type="string">Checkout</item>
                <item name="path" xsi:type="string">checkout_onepage_success</item>
                <item name="route" xsi:type="string">checkout/onepage/success</item>
                <item name="sample_data" xsi:type="boolean">true</item>
            </item>
        </argument>
    </arguments>
</type>
```

(`OOCS_LynxCheckout` declares this one; a module adding its own page declares it the same way in its global
`etc/di.xml`.) `OOCS\LynxComponent\Helper\PreviewSample::isActive()` is what `LayoutResolver` checks; elsewhere the
preview uses real data.
