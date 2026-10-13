---
name: lynx-provider
description: Writes the PHP provider behind a Lynx PageBuilder component - a class extending OOCS\LynxComponent\Model\AbstractComponentProvider whose resolve() turns a node's saved settings into the props the Vue/React component renders (Magento data, translated `labels`, `_i18n`/image/url settings), with deferred()/deferredGroup()/optional() for loading after the page or on demand (the `lazy` toggle convention), full page cache tags through PageCacheIdentities, the Lynx Component Data cache, and SampleProviderInterface::resolveSample() for sample-data preview pages. Use when a developer says "load products/data into my Lynx component", "component provider", "translate labels for a Lynx component", "defer / lazy load a block", "page cache not refreshing for my block", or "sample data in the PageBuilder preview". Use with lynx-basics and lynx-component.
---
# Lynx component provider

Produces a provider class and its registration on a component entry. The provider runs on the server for every
page (or partial request) that renders a node of the component; what `resolve()` returns is exactly what the
component receives as props.

## Before you start

- The component entry exists in global `etc/di.xml` (skill **lynx-component**); settings are declared (skill
  **lynx-component-settings**).
- A component needs a provider when it shows Magento data, has `_i18n` / `image` / `url` settings, or shows any text
  of its own (button labels, `aria-label`, messages).
- Find a similar built-in provider to imitate, under `vendor/oocscommerce/module-lynx-*/Provider/`:
  `OOCS\LynxThemeComponent\Provider\Page\BackToTop` (labels only),
  `…\Page\Image` (i18n + image + url), `OOCS\LynxCatalog\Provider\Product\Specs` (current product, `lazy`, cache tags),
  `OOCS\LynxThemeComponent\Provider\Checkout\Success` (sample data).

## Steps

### 1. The class - `Provider/Highlight.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Provider;

use OOCS\LynxComponent\Helper\LocalizedText;
use OOCS\LynxComponent\Model\AbstractComponentProvider;

class Highlight extends AbstractComponentProvider
{
    /**
     * @param LocalizedText $localizedText
     */
    public function __construct(
        private readonly LocalizedText $localizedText,
    ) {
    }

    /**
     * Props the component receives
     *
     * @param array $props the node's settings, every declared default filled in
     * @return array
     */
    public function resolve(array $props): array
    {
        $props['title'] = $this->localizedText->get($props['title']);
        $props['labels'] = [
            'close' => (string) __('Close'),
        ];

        return $props;
    }
}
```

Contract (`OOCS\LynxComponent\Api\ComponentProviderInterface`; extend the abstract class and override only what you
need):

| Method | Base class default | Decides |
|---|---|---|
| `resolve(array $props): array` | abstract | The props the component receives |
| `deferred(array $props): bool` | `false` | Leave the props out of the page; the browser fetches them right after load |
| `deferredGroup(array $props): string` | `'default'` | Deferred providers of one group come back in one request |
| `optional(array $props): bool` | `false` | Leave the props out until the component asks (`router.reload({ only: [_key] })`) |
| `toPlainText(string $value): string` (protected) | - | Strip tags, decode entities, trim: for catalog text (names, attribute labels) sent as plain text |

All methods get the node's props (saved settings + defaults), so each node can decide from a setting.

### 2. Register it as a `\Proxy`

```xml
<item name="vendor_highlight" xsi:type="array">
    <item name="component" xsi:type="string">Vendor_Module::Components/Highlight</item>
    <item name="provider" xsi:type="object">Vendor\Module\Provider\Highlight\Proxy</item>
    …
</item>
```

The pool is built on every request; the proxy means the provider and its dependencies are created only on pages that
contain the component. A provider that does not implement `ComponentProviderInterface` is silently ignored.

### 3. Write `resolve()`

- **Return the props you received plus your data.** Keys you drop are gone from the component.
- **Read the page context like a block would**: `current_product` / `current_category` from
  `Magento\Framework\Registry`, the store from `StoreManagerInterface`, sessions, config.

```php
/** @var \Magento\Catalog\Model\Product|null $product */
$product = $this->registry->registry('current_product');
if (!$product) {
    return $props;
}
```

- **Resolve portable settings**: `LocalizedText::get()` for `_i18n`, `OOCS\LynxThemeComponent\Helper\MediaUrl::url()`
  for `image`, `OOCS\LynxThemeComponent\Helper\StoreLink::path()` for `url` (also inside `array` items, with
  `?? ''` because the server does not fill sub-field defaults). Add `OOCS_LynxThemeComponent` to `<sequence>` /
  `composer.json` when using its helpers.
- **Text the component shows goes in `labels`**, translated with `__()`. Reuse Magento's own phrases ("Add to Cart",
  "Sign In", "Items %1 to %2 of %3 total") so language packs translate them; add new phrases to the module's
  `i18n/en_US.csv`. Keep `%1`, `%2` for values only the browser knows and `replace()` them in the component. Name
  keys by role (`submit`, `empty`). A component with text but no data still gets a provider that only returns
  `labels`.
- **No per-visitor data in props** (cart, wishlist, customer name): page props go through the full page cache. Read it
  in the browser from customer data.
- `resolve()` is memoized per request by component name + props: two identical nodes on one page resolve once.
- `LayoutResolver` calls `resolve()` from outside, so other modules can extend a provider with an `after` plugin on
  `resolve`.

### 4. Tag the page for the full page cache

When the props show entity data, add the entity's cache tags so saving the entity refreshes the pages:

```php
use Magento\Catalog\Model\Product;
use OOCS\LynxComponent\Model\PageCacheIdentities;

$identities = [Product::CACHE_TAG];
foreach ($products as $product) {
    $identities[] = Product::CACHE_TAG . '_' . $product->getId();
}
$this->pageCacheIdentities->add($identities);
```

Add them inside `resolve()`: a deferred/optional node is fetched in its own cached request, and `resolve()` runs
there. A product-page block that only shows the current product needs to tag it only when it is lazy (the page
response already carries the product tags) - see `OOCS\LynxCatalog\Provider\Product\Specs`.

For data that is expensive and shared by many pages (store lists, menus, aggregates), cache it in
`OOCS\LynxComponent\Model\Cache\Type\ComponentCache` ("Lynx Component Data" in Cache Management) with entity tags;
include store, currency, customer group… in the key. Example in `reference/provider-examples.md`.

### 5. Optional: defer or load on demand

| Situation | Use |
|---|---|
| Above the fold or important for search (product detail, hero, category grid) | Default: resolve with the page |
| Below the fold, expensive, not needed for search (related products, recommendations) | `deferred()` returns `true` |
| Several such blocks on one page | `deferred()` + the same `deferredGroup()` |
| Only some shoppers look at it (closed tab, modal, later checkout step) | `optional()` returns `true`; the component asks |
| Merchant should choose | A toggle setting read by `deferred()` (`deferred` setting) or the `lazy` convention |
| Depends on the shopper | Neither: customer data in the browser |

```php
public function deferred(array $props): bool
{
    return !empty($props['deferred']);   // toggle setting on the node
}

public function deferredGroup(array $props): string
{
    return 'recommendations';
}
```

How it works: `LayoutResolver` does not call `resolve()`; it registers it with Inertia under the **component's
registered name** and sends the node with its stored props plus `_key` (= that name). `deferred` is announced and
fetched by Inertia after mount, one request per group; `optional` wins over `deferred` and is never fetched unless
the component calls `router.reload({ only: [_key], preserveUrl: true })`. DynamicNode merges the data over the stored
props when it arrives, so the component just sees props that are `undefined` at first: render an `l-skeleton`
placeholder of about the final size meanwhile.

The `lazy` convention ("Load when shown"): a toggle setting `lazy`, `optional()` returns `!empty($props['lazy'])`,
and the component requests its data on mount while a prop `resolve()` always sets is `undefined`. Inside a Tab,
"mount" is when the tab opens. Full Vue/React code in `reference/deferred-and-lazy.md`.

### 6. Optional: sample data for preview pages

Pages that cannot be previewed with real data (order success, account pages…) are declared with `sample_data` `true`
in the `pages` argument of `OOCS\LynxPageBuilder\Model\PreviewPagebuilder`; the editor lists them as
"<page> (Sample)" (declaring a page: skill **lynx-page**). On those pages only, a provider implementing `OOCS\LynxComponent\Api\SampleProviderInterface` gets
`resolveSample()` after `resolve()`:

```php
use OOCS\LynxComponent\Api\SampleProviderInterface;

class OrderNote extends AbstractComponentProvider implements SampleProviderInterface
{
    // resolve() as usual …

    /**
     * Sample values for what the session cannot give in the preview
     *
     * @param array $resolved what resolve() returned, plugins included
     * @param array $props the node's settings, for any resolve() dropped
     * @return array
     */
    public function resolveSample(array $resolved, array $props): array
    {
        $resolved['order_number'] = 'SAMPLE-000000123';
        // Endpoints are '#' in a sample preview; the component sends nothing to '#'
        $resolved['urls'] = array_map(static fn (): string => '#', $resolved['urls'] ?? []);
        // No side effects (analytics events) from a preview
        unset($resolved['purchase']);

        return $resolved;
    }
}
```

Overwrite only what the session or database cannot provide; keep real labels and settings. Providers without the
interface resolve normally with the admin browser's real session. Do not call `$this->resolve()` inside
`resolveSample()` (that would bypass plugins; the result is already passed in).

## Rules

- **Always `\Proxy`** for `provider` (and for any option `source`).
- **Settings arrive complete**: read `$props['key']` directly for top-level settings (defaults are filled before the
  provider runs); use fallbacks only for `array` sub-fields and props that may be absent.
- **Deferred/optional nodes get their stored props first**: until the data arrives the component sees raw setting
  values (an `_i18n` object, a media path) and `undefined` data props. Type such props loosely or check the
  data prop before using them (`ProductCarousel` types `title?: string | Record<string, string>`).
- **Labels of a deferred/optional provider arrive with its data.** Text needed on the first paint belongs to a
  provider that is not deferred.
- **One key per component**: deferred data is keyed by the registered name, so two nodes of the same component on a
  page share it and the last one resolved wins. Do not defer a component that can appear several times on a page.
- **Deferred data is not in the SSR HTML** and each group / optional reload is one more (FPC-cached) request: do not
  split a page into many groups, and never defer what search engines must see.
- A block that can have nothing to show (feature off) returns `enabled: false` and must **not** be optional in that
  case, so a parent Tabs can hide the tab on first paint.
- **Never put per-visitor data in props**; never cache per-visitor data in `ComponentCache`.
- The `provider` key never reaches the browser; do not expose secrets through returned props either.

## Check

1. `bin/magento cache:clean` after di.xml changes; `setup:di:compile` in production mode (proxies are generated).
2. Storefront: the component receives the resolved props (Vue/React devtools, or the Inertia page JSON in the HTML / the `X-Inertia` response of a visit).
3. Store view in another locale: `_i18n` settings and `labels` come out translated.
4. Save an entity the block shows in the admin: the page refreshes (FPC tags work); check the deferred request too.
5. Deferred: page HTML has the node without data, one follow-up request per group fills it; placeholder has no
   layout shift. Optional/lazy: no request until the component asks (open the tab).
6. Editor preview: the node renders; on a "(Sample)" page `resolveSample()` values show and no side effects fire.
7. Enable SSR if the store uses it and confirm the non-deferred content is in the server HTML.

## Reference

- `reference/deferred-and-lazy.md` - full `lazy` example (provider, di.xml toggle, Vue and React), deferred flow.
- `reference/provider-examples.md` - images/links/translations provider, catalog provider with cache tags,
  `ComponentCache` example, sample provider.
- Skills: **lynx-basics**, **lynx-component**, **lynx-component-settings**.
- https://docs.oocscommerce.com/lynx/developer/providers
