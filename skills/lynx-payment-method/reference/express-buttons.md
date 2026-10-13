# Express checkout buttons

An express button (wallet: PayPal, Apple Pay, Google Pay...) sits **outside** the steps. The shopper clicks, the wallet
supplies the address and rate, and the order is placed without walking the steps. It is an ordinary PageBuilder
component (see **lynx-component**, **lynx-provider**) the merchant drops into the **Express Checkout** slot (`express`):

| Component holding the slot | Where |
|---|---|
| **Checkout** (`default_checkout`) | Above the steps, followed by "Or proceed with the standard checkout" |
| **Checkout Cart** (`default_checkout_cart`) | Summary column, under **Proceed to Checkout** |

The slot accepts only group `express_checkout`, each component once (`max_per_component` 1). Layout comes from the
holder's **Express Buttons Per Row** / **Express Buttons Gap** settings (`express_columns`, `express_gap`); the button
itself usually has no settings.

Lynx references: `OOCS_LynxPaymentBraintree::Components/Express/PayPal` (provider
`OOCS\LynxPaymentBraintree\Provider\Express\PayPal`), `OOCS_LynxPaymentStripe::Components/Express/Wallets`
(`OOCS\LynxPaymentStripe\Provider\Express\Wallets`), `OOCS_LynxPaymentPayPal::Components/Express/Shortcut`
(`OOCS\LynxPaymentPayPal\Provider\Express\Shortcut`, also on the cart).

## 1. Registry entry - global `etc/di.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="OOCS\LynxComponent\Model\Registry">
        <arguments>
            <argument name="components" xsi:type="array">
                <item name="vendor_foo_express" xsi:type="array">
                    <item name="component" xsi:type="string">Vendor_PaymentFoo::Components/Express/Foo</item>
                    <item name="group" xsi:type="string">express_checkout</item>
                    <item name="icon" xsi:type="string">ph-wallet</item>
                    <item name="provider" xsi:type="object">Vendor\PaymentFoo\Provider\Express\Foo\Proxy</item>
                    <item name="label" xsi:type="string">Foo Express</item>
                    <item name="description" xsi:type="string">Foo wallet button: the shopper pays in Foo's sheet and the order is placed without the checkout steps.</item>
                    <item name="zones" xsi:type="array">
                        <item name="content" xsi:type="string">content</item>
                    </item>
                    <item name="available_on" xsi:type="array">
                        <item name="checkout" xsi:type="string">checkout_index_index</item>
                    </item>
                    <item name="settings" xsi:type="array"/>
                </item>
            </argument>
        </arguments>
    </type>
</config>
```

- Global scope, not `etc/frontend`: the registry is read in more than one area and an area list would replace it.
- Note the `::` in `component` (registry syntax), unlike the renderer pool's `/`.
- Add `<item name="cart" xsi:type="string">checkout_cart_index</item>` to `available_on` **only** if the button works on
  the cart page (see section 5).

## 2. Provider - decides whether the button shows

```php
<?php
declare(strict_types=1);

namespace Vendor\PaymentFoo\Provider\Express;

use Magento\Checkout\Model\Session as CheckoutSession;
use Magento\Payment\Helper\Data as PaymentHelper;
use OOCS\LynxComponent\Model\AbstractComponentProvider;

class Foo extends AbstractComponentProvider
{
    /**
     * @param CheckoutSession $checkoutSession
     * @param PaymentHelper $paymentHelper
     */
    public function __construct(
        private readonly CheckoutSession $checkoutSession,
        private readonly PaymentHelper $paymentHelper,
    ) {
    }

    /**
     * Whether the button shows, and what it needs
     *
     * @param array $props
     * @return array
     */
    public function resolve(array $props): array
    {
        $quote = $this->checkoutSession->getQuote();
        // Same question the Payment step asks: active flag, country, currency, min/max total
        $props['enabled'] = $quote->getId()
            && !$quote->isVirtual()
            && $this->paymentHelper->getMethodInstance('vendor_foo')->isAvailable($quote);

        if (!$props['enabled']) {
            return $props;
        }

        return $props + [
            'labels' => [
                'acceptTerms' => (string) __('Please agree to all Terms and Conditions before placing the order.'),
            ],
        ];
    }
}
```

- Base `enabled` on the method's own `isAvailable()`, not only a button-specific flag, so the button and the Payment
  step never disagree. Add the button's own config switches on top.
- Drop the `isVirtual()` check only if the wallet flow handles carts without shipping.
- No gateway calls here (client tokens etc.): the component fetches them after mount.

## 3. Component - drives the quote through the checkout store

No `placeOrder` / `validate` props. Use the store from `OOCS_LynxCheckout/stores/checkoutStore`:

| Task | Use |
|---|---|
| Quote REST call (`estimate-shipping-methods`, `shipping-information`, `payment-information`) | `checkout.rest(path, body)` - prefixes `guest-carts/<masked id>` or `carts/mine` |
| Wallet state code ("CA") to `region_id` | `checkout.regionIdByCode(countryId, code)` |
| Amount to open the wallet with | `checkout.grandTotal?.value` (do not open before it is known) |
| After the order is placed | `await checkout.orderPlaced()` then `window.location.assign(checkout.successUrl)` |
| Show an error | `useMessagesStore().add('error', message)` from `OOCS_LynxTheme/stores/global/messagesStore` |
| Terms and Conditions | `checkout.termsAccepted`, `checkout.agreementIds` |
| SDK that injects inline scripts | `checkout.cspNonce` |

Sequence (as in Braintree's express PayPal):

1. **Open the wallet** on the current grand total; the wallet asks for the address.
2. **Address or rate changed**: POST `estimate-shipping-methods` with the partial address the wallet shares (only
   `country_id` is required), save the chosen rate with `shipping-information`, update the wallet amount from the
   `totals` it returns.
3. **Shopper approves**: save the full address with `shipping-information`, POST `payment-information` with the token
   and agreement ids, `await checkout.orderPlaced()`, go to `checkout.successUrl`.

```ts
await checkout.rest('payment-information', {
    paymentMethod: {
        method: 'vendor_foo',
        additional_data: { foo_token: token },
        ...(checkout.agreementIds.length ? { extension_attributes: { agreement_ids: checkout.agreementIds } } : {}),
    },
    billingAddress,
})
```

Skeleton (Vue; React is the same with `useCheckoutStore()` / `useMessagesStore()` hooks at the top of the function and
`return null` when not shown):

```vue
<script setup lang="ts">
import { onMounted, ref } from 'vue'
import { useCheckoutStore } from 'OOCS_LynxCheckout/stores/checkoutStore'
import { useMessagesStore } from 'OOCS_LynxTheme/stores/global/messagesStore'

const props = defineProps<{ enabled?: boolean; labels?: Record<string, string> }>()

const checkout = useCheckoutStore()
const messages = useMessagesStore()
// The wallet may report that this browser is not supported
const eligible = ref(false)

onMounted(async () => {
    if (!props.enabled) return
    eligible.value = await window.FooWallet.isSupported()
})

async function pay() {
    // Second guard: the card already disables the buttons until terms are ticked
    if (!checkout.termsAccepted) {
        messages.add('error', props.labels?.acceptTerms ?? '')
        return
    }
    if (!checkout.grandTotal) return
    try {
        // open the wallet with checkout.grandTotal.value, handle steps 2-3 above
        await checkout.orderPlaced()
        window.location.assign(checkout.successUrl)
    } catch (error) {
        messages.add('error', (error as Error).message)
    }
}
</script>

<template>
    <div v-if="enabled && eligible">
        <!-- Usually the wallet SDK draws its own branded button into a container instead -->
        <button type="button" class="l-btn l-btn-primary w-full" @click="pay">{{ labels?.payWithFoo }}</button>
    </div>
</template>
```

Add `payWithFoo` to the provider's `labels`; never hardcode UI text.

- When the button cannot work (`enabled` false, wallet says ineligible) render **nothing** and show no error. The
  Express Checkout card hides itself when its `[data-express-buttons]` container ends up empty.
- Show a `l-skeleton` the button's height while the SDK draws.

## 4. Terms and Conditions

With terms enabled, the Express Checkout card shows them above the buttons and disables the buttons (`opacity-50
pointer-events-none`) until every manual term is ticked, because a wallet sheet must open within the click. The ticks
are shared with the Payment step. A button that places the order through `payment-information` must send
`agreement_ids` (Magento validates them) and should check `checkout.termsAccepted` in its click handler. A flow that
places the order elsewhere (a review page, the gateway module's own endpoint) does not go through core's terms check.

## 5. On the cart page

The checkout store is initialised by the **Checkout** component, so on the cart page `checkout.rest()` has no cart URL
and `grandTotal` is `null`. A button that drives the quote through the store works on the checkout page only. A button
that only **navigates** (to the gateway, then to a review page) works on both; that is why Lynx's PayPal Express and
PayPal Credit buttons are available on the cart and the Braintree / Stripe express buttons are not.
