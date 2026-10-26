# Analytics: bus, events, tracker component

## Store API

`OOCS_LynxTheme/stores/global/analyticsStore` (`useAnalyticsStore()`):

| Member | Meaning |
|---|---|
| `track(event, params = {})` | Sends to every registered tracker and keeps it in the current page's buffer (last 20; a `page_view` starts a new page) |
| `trackOnce(key, event, params = {})` | Like `track()`, at most once per browser session per `key` (`sessionStorage` key `lynx_analytics_<key>`) |
| `addTracker(tracker)` | Registers `tracker(event, params)` and replays the current page's buffer to it. A tracker that throws is logged and skipped, never breaking the caller |
| `removeTracker(tracker)` | Unregisters |

Type: `AnalyticsTracker = (event: string, params: Record<string, unknown>) => void`.

## Events Lynx tracks

Event names and parameters follow the GA4 ecommerce format. `currency` is the store's currency code. An item is `{ item_id, item_name, price, quantity }`, where `item_id` is the SKU.

| Event | Tracked by | When | Parameters |
|---|---|---|---|
| `page_view` | The theme's entry point | Every page load and every page change | `page_location`, `page_title` |
| `view_item_list` | Product Listing | The list is shown, and again when it changes (pagination, filters, search) | `item_list_name` (the list title), `items` with `index` and without `quantity` |
| `view_item` | Product page (simple, configurable, bundle, grouped, downloadable) | The product page is shown | `currency`, `value` (price), `items` (one item, without `quantity`) |
| `add_to_cart` | Product page, Product Listing quick add, Product Carousel, Checkout Cart | The server accepted the add; on the cart, a quantity increase | `currency`, `value`, `items`. On the product page the price includes the chosen options; grouped products send one item per product with a quantity |
| `remove_from_cart` | Checkout Cart | A quantity decrease or a removed item | `currency`, `value`, `items` (`quantity` is the amount removed) |
| `view_cart` | Checkout Cart | The cart page is shown with items | `currency`, `value` (sum of row totals), `items` |
| `begin_checkout` | Checkout | The checkout is shown with items | `currency`, `value` (subtotal), `coupon` (when applied), `items` |
| `add_shipping_info` | Checkout Shipping Method step | **Continue** on the step | `shipping_tier` (`<carrier>_<method>`) |
| `add_payment_info` | Checkout Payment step | **Place Order**, after validation | `payment_type` (the payment method code) |
| `purchase` | Checkout Success page | The success page is shown; once per order (`trackOnce`) | `transaction_id` (order number), `value` (grand total), `tax`, `shipping`, `currency`, `coupon` (when used), `items` |

The `purchase` parameters are built on the server from the order (`OOCS\LynxThemeComponent\Provider\Checkout\Success`), the same values Magento's own Google Tag module reports on the native success page. They are passed to the page as the `purchase` prop of the Checkout Success component, so a tracker gets them from the bus without reading the order itself.

The GA4 component (`OOCS_LynxGoogleTag`) forwards `page_view`, `purchase`, `view_item_list`, `view_item`,
`add_to_cart`, `view_cart`, `begin_checkout`, `add_shipping_info`, `add_payment_info`, each behind an `event_<name>`
toggle. Anything else on the bus (including `remove_from_cart` and module events) is for other trackers.

## Tracker component

A headless component placed in the **Backstage** zone; the merchant adds it per design, configures it, removes it
without a deployment. The steps:

1. Register a component with `zones` = `backstage`, settings for per-event toggles and load options, and a provider that
   passes the account id.
2. In the component, register a tracker with `addTracker()` on mount, map Lynx events to the vendor's names, remove it
   on unmount.
3. Load the vendor script lazily, and only once the consent cookies the merchant listed exist.
4. Whitelist the vendor's domains in `etc/csp_whitelist.xml`.
5. Do nothing in the PageBuilder preview.

### Example: `Vendor_FooPixel`

```
app/code/Vendor/FooPixel/
├── etc/
│   ├── adminhtml/system.xml          ← "Foo Pixel ID" field (standard Magento, not shown)
│   ├── components/FooPixel.graphqls
│   ├── csp_whitelist.xml
│   └── module.xml
├── Provider/FooPixel.php
└── view/frontend/lynx/
    └── vue/src/Components/FooPixel.vue
```

#### Declare the component

`etc/components/FooPixel.graphqls`. Settings named `event_<name>` follow the GA4 component's convention. Setting types:
skill **component-settings**.

```graphql
"Sends the events switched on below to Foo. Renders nothing; place it once in Backstage."
type VendorFooPixel @component(
  code: "vendor_foo_pixel"
  template: "Vendor_FooPixel::Components/FooPixel"
  provider: "Vendor\\FooPixel\\Provider\\FooPixel"
  label: "Foo Pixel"
  group: "utility"
  icon: "ph-chart-bar"
  zones: ["backstage"]
) {
  "One cookie name per line. The Foo script loads only while every one of them exists with a value, e.g. cookie_consent from the Cookie Consent component. Empty: no condition."
  consent_cookies: Textarea! @setting(label: "Require cookies") @optional
  event_page_view: Boolean! @setting(label: "Send page views", default: true)
  event_add_to_cart: Boolean! @setting(label: "Send add to cart", default: true)
  event_purchase: Boolean! @setting(label: "Send purchases", default: true)
  "From store configuration; empty when tracking is off"
  pixel_id: String!
}
```

#### The provider

The provider passes the account id from store configuration, or an empty string when tracking is off, so the component does nothing. Provider contract: skill **provider**.

```php
<?php
declare(strict_types=1);

namespace Vendor\FooPixel\Provider;

use Magento\Framework\App\Config\ScopeConfigInterface;
use Magento\Store\Model\ScopeInterface;
use OOCS\LynxComponent\Model\AbstractComponentProvider;

class FooPixel extends AbstractComponentProvider
{
    public function __construct(
        private readonly ScopeConfigInterface $scopeConfig,
    ) {
    }

    public function resolve(array $props): array
    {
        $props['pixel_id'] = $this->scopeConfig->isSetFlag('vendor_foo_pixel/general/active', ScopeInterface::SCOPE_STORE)
            ? (string) $this->scopeConfig->getValue('vendor_foo_pixel/general/pixel_id', ScopeInterface::SCOPE_STORE)
            : '';

        return $props;
    }
}
```

#### Content Security Policy

Allow the vendor's script and the hosts it sends data to, as for any Magento module. The Lynx checkout enforces its policy, so a missing entry stops tracking exactly where `purchase` matters.

```xml
<?xml version="1.0"?>
<csp_whitelist xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
               xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Csp:etc/csp_whitelist.xsd">
    <policies>
        <policy id="script-src">
            <values>
                <value id="foo_pixel_script" type="host">https://cdn.foo.example</value>
            </values>
        </policy>
        <policy id="connect-src">
            <values>
                <value id="foo_pixel_collect" type="host">https://collect.foo.example</value>
            </values>
        </policy>
        <policy id="img-src">
            <values>
                <value id="foo_pixel_img" type="host">https://collect.foo.example</value>
            </values>
        </policy>
    </policies>
</csp_whitelist>
```

If the store runs `bin/magento lynx:compile`, run it again after installing the module so the compiled configuration includes the new whitelist.

#### The component

The component keeps one runtime per page even if a design holds the node twice, and builds the vendor's queue function before the script loads so no event is lost while waiting. The script itself loads on the shopper's first interaction, and only once the consent cookies exist; each interaction checks again until it has loaded.

```vue
<script setup lang="ts">
import { computed, onMounted, onUnmounted, useAttrs } from 'vue'
import { useAnalyticsStore } from 'OOCS_LynxTheme/stores/global/analyticsStore'

// Node attributes go on the preview placeholder, the only element this component renders
defineOptions({ inheritAttrs: false })
const attrs = useAttrs()

const props = withDefaults(defineProps<Lynx.VendorFooPixel>(), { consent_cookies: '' })

declare global {
    interface Window {
        foo?: ((...args: unknown[]) => void) & { queue?: unknown[] }
    }
}

const INTERACTIONS = ['pointerdown', 'keydown', 'touchstart', 'scroll'] as const
// PageBuilder previews the storefront in an iframe
const inPreview = () => window.location.pathname.includes('/lynx_pagebuilder/') || window.parent !== window

// Lynx event name → the vendor's name and parameters
const MAP: Record<string, (p: Record<string, any>) => [string, Record<string, unknown>]> = {
    page_view: () => ['PageView', {}],
    add_to_cart: p => ['AddToCart', { value: p.value, currency: p.currency, contents: p.items }],
    purchase: p => ['Purchase', { order_id: p.transaction_id, value: p.value, currency: p.currency, contents: p.items }],
}

const analytics = useAnalyticsStore()
let instances = 0
let stop: (() => void) | null = null
let requested = false

const cookieNames = computed(() =>
    props.consent_cookies.split(/\r?\n/).map(name => name.trim()).filter(Boolean)
)

function consentGiven(): boolean {
    const present = new Set(
        document.cookie
            .split(';')
            .map(part => part.trim())
            .filter(part => part.includes('=') && !part.endsWith('='))
            .map(part => part.slice(0, part.indexOf('=')))
    )
    return cookieNames.value.every(name => present.has(name))
}

function loadScript() {
    if (requested || !consentGiven()) return
    requested = true
    const script = document.createElement('script')
    script.src = 'https://cdn.foo.example/pixel.js'
    script.async = true
    document.head.appendChild(script)
    for (const name of INTERACTIONS) document.removeEventListener(name, loadScript)
}

function send(event: string, params: Record<string, unknown>) {
    if (!MAP[event] || !props[`event_${event}` as keyof typeof props]) return
    window.foo!('track', ...MAP[event](params))
}

onMounted(() => {
    if (inPreview() || !props.pixel_id || instances++ > 0) return
    // The vendor's stub: calls queue up until the script replaces it
    window.foo ||= Object.assign((...args: unknown[]) => window.foo!.queue!.push(args), { queue: [] })
    window.foo('init', props.pixel_id)

    analytics.addTracker(send)
    for (const name of INTERACTIONS) document.addEventListener(name, loadScript, { passive: true })
    stop = () => {
        analytics.removeTracker(send)
        for (const name of INTERACTIONS) document.removeEventListener(name, loadScript)
    }
})

onUnmounted(() => {
    if (inPreview() || !props.pixel_id || --instances > 0) return
    stop?.()
    stop = null
})
</script>

<template>
    <div
        v-if="inPreview()"
        :data-lynx-id="attrs['data-lynx-id']"
        :data-lynx-label="attrs['data-lynx-label']"
        class="rounded-box border-(length:--border) border-dashed border-base-300 px-3 py-2 text-sm text-base-content/70"
    >
        Foo Pixel · {{ pixel_id }}
    </div>
</template>
```

Points to keep from this example:

- **Load lazily.** The interaction trigger keeps the vendor script out of the first page load. Add a timed fallback, as the GA4 component's **Delay** setting does, if visitors who never interact must be counted too.
- **Consent.** The `consent_cookies` setting lets the merchant tie loading to the **Cookie Consent** component (cookie `cookie_consent`) or a consent tool's own cookie. Events keep queuing in the stub while consent is pending and are sent once the script loads.
- **One runtime per page.** Count instances instead of using a flag, so a node unmounting while another mounts (entering the checkout, for example) does not remove the tracker.
- **Skip the preview.** PageBuilder shows the storefront in an iframe; tracking there would send the merchant's editing as visits. Show a small placeholder carrying the node's `data-lynx-id` and `data-lynx-label` attributes so the merchant can select the node in the preview.
- **Text.** The placeholder is shown to the merchant in the editor only. Anything shown to shoppers comes from server-provided labels.

After installing, the merchant opens **PageBuilder**, adds **Foo Pixel** to the **Backstage** zone of each design that should track, and saves.
