---
name: lynx-payment-method
description: Adds a payment method to the Lynx one-page checkout (OOCS_LynxCheckout) - the client-side renderer (Vue and React) mounted under the method's radio in the Payment step, its `paymentMethodRenderers` entry in etc/frontend/di.xml, the PHP config class that feeds it public keys and translated labels, and optionally an express checkout button for the `express` slot of the Checkout and Checkout Cart components. Covers the form, wallet button and redirect flow shapes (`preparePayment`, `afterPlaceOrder`, `ownsAction`, `loaded`, `placeOrder`, `validate`), offline vs online visibility, the reCAPTCHA key per method, the CSP nonce for gateway SDKs and an offline method with its own field. Use when a developer says "payment renderer for Lynx checkout", "make my payment gateway work on Lynx", "my payment method does not show in the Lynx checkout", "add a PO/reference field to an offline method", "Apple Pay / wallet / express button on Lynx". Use with lynx-basics.
---

# Lynx payment method renderer

Produces the Lynx side of a Magento payment method: a renderer component (Vue + React) under the method's radio in the
Payment step, its pool entry, a config class, and optionally an express checkout button. The server side (payment
model, gateway commands, `system.xml`, data-assign observer) stays a standard Magento payment method.

## Before you start

- Read **lynx-basics** (module layout, `view/frontend/lynx/<vue|react>/src`, import aliases, build commands, styling
  rules). Express buttons are ordinary components: their registry entry and provider follow **lynx-component** and
  **lynx-provider**. reCAPTCHA keys and analytics events are covered in **lynx-forms-analytics**.
- Find the payment method code (`payment/<code>/...` in `config.xml`) and whether the method is offline
  (`MethodInterface::isOffline()`: `payment/<code>/is_offline` for an `Adapter` (gateway) method, `$_isOffline` for an
  `AbstractMethod` subclass).
- Decide the flow shape (see Step 3). Imitate the closest Lynx renderer in `vendor/oocscommerce/`:

  | Shape | Reference renderer (`<Module>/<path under src>`) |
  |---|---|
  | Form (card fields, the shared Place Order submits) | `OOCS_LynxPaymentBraintree/Components/Cc`, `OOCS_LynxPaymentStripe/Components/PaymentElement`, `OOCS_LynxCheckout/Components/Checkout/Payment/PurchaseOrder` |
  | Button (wallet places the order itself) | `OOCS_LynxPaymentBraintree/Components/PayPal`, `.../GooglePay`, `.../ApplePay`, `.../Venmo` |
  | Redirect (order first, then the hosted page) | `OOCS_LynxPaymentStripe/Components/StripeCheckout` |
  | Express button | `OOCS_LynxPaymentBraintree::Components/Express/PayPal`, `OOCS_LynxPaymentStripe::Components/Express/Wallets`, `OOCS_LynxPaymentPayPal::Components/Express/Shortcut` |

- The Payment step code to read when in doubt: `OOCS\LynxCheckout\Provider\Checkout\Step\Payment` and
  `OOCS_LynxCheckout/Components/Checkout/Step/Payment.{vue,tsx}`.

### Which methods show up

The Payment step lists `PaymentMethodManagementInterface::getList()` for the quote, filtered:

| Method | No pool entry | Pool entry |
|---|---|---|
| Offline (checkmo, cashondelivery, banktransfer, free...) | Shown, order placed with `{ method }` only | Shown with the renderer |
| Online (gateway) | **Hidden** | Shown with the renderer |

An online method that needs nothing from the browser gets an entry **without** `component`; it is then shown and placed
with `{ method }`:

```xml
<item name="vendor_invoice" xsi:type="array"/>
```

Offline methods already print their **Instructions** (and Check / Money Order's **Make Check payable to** / **Send Check
to**) without a renderer.

## Steps

### 1. Module wiring

`etc/module.xml` sequences `OOCS_LynxCheckout` (and the gateway module if it is a separate one);
`composer.json` requires `oocscommerce/module-lynx-checkout`.

```xml
<module name="Vendor_PaymentFoo">
    <sequence>
        <module name="OOCS_LynxCheckout"/>
    </sequence>
</module>
```

### 2. Register the renderer - `etc/frontend/di.xml`

One item per **payment method code** in the `paymentMethodRenderers` argument of
`OOCS\LynxCheckout\Provider\Checkout\Step\Payment`:

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="OOCS\LynxCheckout\Provider\Checkout\Step\Payment">
        <arguments>
            <argument name="paymentMethodRenderers" xsi:type="array">
                <item name="vendor_foo" xsi:type="array">
                    <item name="component" xsi:type="string">Vendor_PaymentFoo/Components/Foo</item>
                    <item name="config" xsi:type="object">Vendor\PaymentFoo\Model\Config\Foo</item>
                    <!-- optional; default place_order -->
                    <item name="recaptcha" xsi:type="string">place_order</item>
                </item>
            </argument>
        </arguments>
    </type>
</config>
```

| Key | Meaning |
|---|---|
| `component` | `<Module>/<path under src>` without extension, **slash** after the module name (not `::`). Resolves to the `.vue` on a Vue theme and the `.tsx` on a React theme |
| `config` | Object implementing `OOCS\LynxCheckout\Api\PaymentMethodRendererConfigInterface`; `getConfig()` becomes the `config` prop. A wrong class throws a `LogicException` naming the entry |
| `recaptcha` | reCAPTCHA key guarding place order for this method. Absent = `place_order`; `none` skips it. Use the key the gateway's own module validates under (Lynx's Braintree card uses `braintree`, Braintree PayPal/Google Pay use `none`). See **lynx-forms-analytics** |

### 3. The config class - `Model/Config/Foo.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\PaymentFoo\Model\Config;

use Magento\Framework\App\Config\ScopeConfigInterface;
use Magento\Store\Model\ScopeInterface;
use OOCS\LynxCheckout\Api\PaymentMethodRendererConfigInterface;

class Foo implements PaymentMethodRendererConfigInterface
{
    /**
     * @param ScopeConfigInterface $scopeConfig
     */
    public function __construct(
        private readonly ScopeConfigInterface $scopeConfig,
    ) {
    }

    /**
     * Public values and labels for the renderer
     *
     * @return array
     */
    public function getConfig(): array
    {
        return [
            'publishableKey' => (string) $this->scopeConfig->getValue(
                'payment/vendor_foo/publishable_key',
                ScopeInterface::SCOPE_STORE
            ),
            'labels' => [
                'cardNumber' => (string) __('Credit Card Number'),
                'required' => (string) __('This is a required field.'),
            ],
        ];
    }
}
```

- Public values only: everything returned is printed into the page. Secrets stay server-side.
- Every visible string goes under `labels`, translated with `__()`; reuse Magento's own phrases where they exist.
- `getConfig()` runs only when the Payment step is fetched and only for methods available on the quote; put network
  calls there, never in the constructor. If the constructor must be heavy, register `...\Foo\Proxy` in the pool.

### 4. The renderer - pick the shape

Files: `view/frontend/lynx/vue/src/Components/Foo.vue` and `view/frontend/lynx/react/src/Components/Foo.tsx`
(same relative path in both). Gateway SDKs from npm go in the module's own `view/frontend/lynx/<vue|react>/package.json`
(pinned version), never in the theme's.

**Props in:**

| Prop | Type | Meaning |
|---|---|---|
| `config` | `Record<string, unknown>` | What `getConfig()` returned |
| `placeOrder` | `(additionalData?: Record<string, unknown>) => Promise<boolean>` | Places the order; `true` when placed. On failure it shows the error itself and resolves `false` |
| `validate` | `() => boolean` | Checks a method is picked and every manual Terms and Conditions box is ticked |

**Exposed out** (Vue `defineExpose`, React `useImperativeHandle(ref, ...)` on the `ref` prop - React 19 passes `ref` as
a prop). All optional:

| Member | Type | Meaning |
|---|---|---|
| `preparePayment` | `() => Promise<Record<string, unknown>>` | Called by the shared **Place Order** button; resolve the data (token, nonce), reject with a message to stop |
| `afterPlaceOrder` | `(orderId: string) => Promise<string \| null>` | After the order exists; a URL redirects there, `null` goes to the success page |
| `ownsAction` | `boolean` | `true` hides the shared **Place Order** button (wallet draws its own) |
| `loaded` | `boolean` | `false` while the SDK/iframe loads: Place Order stays disabled and the checkbox reCAPTCHA waits. Omit if ready on mount. Vue: expose the `ref` itself; React: list it in the `useImperativeHandle` deps |

What `placeOrder()` does, in this order (from `Step/Payment.vue`):

```
validate() -> reCAPTCHA token for the method's key (unless 'none') -> track('add_payment_info')
  -> preparePayment()   (only when placeOrder() got no additionalData)
  -> POST payment-information { paymentMethod: { method, additional_data, po_number?, agreement_ids }, billingAddress, email? }
  -> checkout.orderPlaced() -> afterPlaceOrder(orderId) -> location.assign(url || successUrl)
```

- What you resolve/pass becomes `paymentMethod.additional_data`. Key **`po_number`** is moved to
  `paymentMethod.po_number` (the `PaymentInterface` field core's Purchase Order reads).
- Billing address, guest email, agreement ids and the reCAPTCHA header are added by the step; do not send them.
- A renderer is mounted the first time its method is picked and stays mounted (hidden) when the shopper switches away,
  so hosted fields keep their input.
- A rejected `preparePayment()` shows `error.message` as an error message; reject with a gateway message or a label.

Minimal **form** renderer (Vue; the React twin and the full SDK version are in
[reference/renderer-examples.md](reference/renderer-examples.md)):

```vue
<script setup lang="ts">
import { onMounted, ref } from 'vue'
import { useCheckoutStore } from 'OOCS_LynxCheckout/stores/checkoutStore'

const props = defineProps<{ config: { publishableKey: string; labels?: Record<string, string> } }>()

const checkout = useCheckoutStore()
const container = ref<HTMLElement>()
const loaded = ref(false)
const error = ref('')

onMounted(async () => {
    // mount the gateway's hosted field into container.value, then:
    loaded.value = true
})

defineExpose({
    loaded,
    async preparePayment(): Promise<Record<string, unknown>> {
        const token = '' // tokenize with the gateway SDK
        if (!token) {
            error.value = props.config.labels?.required ?? ''
            throw new Error(error.value)
        }
        return { foo_token: token }
    },
})
</script>

<template>
    <div>
        <label class="block text-sm text-base-content mb-1.5">{{ config.labels?.cardNumber }}</label>
        <div ref="container" :class="checkout.inputClass(!!error)"></div>
        <p v-if="error" data-checkout-error class="mt-1 text-sm text-error">{{ error }}</p>
    </div>
</template>
```

- **Button** shape: `defineExpose({ ownsAction: true })`; in the wallet click handler `if (!props.validate()) return`,
  open the wallet, then `await props.placeOrder({ foo_token: token })`. A flow that leaves the site *before* the order
  exists and returns to a review page also uses this shape.
- **Redirect** shape: expose `afterPlaceOrder(orderId)` that asks your own frontend controller for the signed URL and
  returns it (or `null`). The URL endpoint and the return controller are ordinary frontend controllers answering JSON or
  a redirect; they need no Lynx route.

### 5. Keep the data on the payment (server)

Plain Magento: an observer on `payment_method_assign_data_<code>` extending
`Magento\Payment\Observer\AbstractDataAssignObserver` copies keys from `additional_data` into the payment's additional
information. Full code in [reference/renderer-examples.md](reference/renderer-examples.md#offline-method-with-its-own-field).

### 6. Optional: express checkout button

A separate PageBuilder component in group `express_checkout`, registered in the **global** `etc/di.xml` registry, that
the merchant drops into the **Express Checkout** slot (`express`) of **Checkout** (`default_checkout`, above the steps)
or **Checkout Cart** (`default_checkout_cart`, under Proceed to Checkout). It has no `placeOrder`/`validate` props; it
drives the quote through the checkout store. Registry entry, provider (`enabled` from the method's own
`isAvailable($quote)`), store calls, terms and the cart-page limit: [reference/express-buttons.md](reference/express-buttons.md).

### 7. Build

```bash
bin/magento module:enable Vendor_PaymentFoo && bin/magento setup:upgrade
bin/magento lynx:workspace:setup OOCS_LynxVue OOCS_LynxReact   # new module or new package.json
bin/magento lynx:static:deploy
bin/magento lynx:compile                                       # if the store uses compiled config
bin/magento cache:clean
```

## Rules

- **Pool scope: `etc/frontend/di.xml`.** Magento merges an area's array argument over the global one with
  `array_replace` per argument name; Lynx's own entries (`purchaseorder`, Braintree, Stripe, PayPal) are in
  `etc/frontend/di.xml`, so one in `etc/di.xml` would be replaced and vanish. The express button's registry entry is the
  opposite: global `etc/di.xml` (the registry is read in several areas).
- **Path syntax differs:** pool `component` uses `Vendor_Module/Components/Foo`; the component registry uses
  `Vendor_Module::Components/Express/Foo`.
- **Online methods need an entry** or they are hidden; do not "fix" a missing method by marking it offline.
- **No UI text in the component.** Every string comes from `config.labels` (translated in PHP).
- **Ship both themes** (`.vue` + `.tsx`, same path) unless the store uses only one; a missing file breaks the method on
  that theme.
- **Styling:** `checkout.inputClass(invalid)` for inputs, `l-btn l-btn-primary` for buttons, tokens only, no text below
  `text-sm`. Mark inline errors with `data-checkout-error`.
- **CSP:** add gateway domains to your module's `etc/csp_whitelist.xml`. An SDK that injects inline scripts must get the
  page nonce `checkout.cspNonce` (PayPal SDK: `data-csp-nonce` attribute; via braintree-web
  `dataAttributes: { 'csp-nonce': checkout.cspNonce }`). Never use a nonce from step props: it belongs to another
  response. Test with a full page load of the checkout (an Inertia visit keeps the previous page's CSP header).
- **SRI:** the checkout pins the hash of every built chunk. Load your code with `import`, never a `<script>` with a URL
  you build. External SDK scripts are outside SRI.
- **No heavy constructors** in the config class or express provider; never fetch client tokens in an express provider
  (it would hold back the whole checkout page) - let the component fetch after mount.
- React: read values a handle method needs through a `useRef` mirror (see PurchaseOrder.tsx) or list them in the deps,
  otherwise `preparePayment` sees stale state.

## Check

- [ ] Method appears in the Payment step on the Vue **and** React theme; its chunk loads only when picked.
- [ ] Place Order disabled until `loaded`; the checkbox reCAPTCHA (when the key is protected) draws after it.
- [ ] Successful order: `additional_data` arrives (check `sales_order_payment.additional_information`).
- [ ] Each failure: tokenization error, shopper closes the popup, expired session, rejected card - message shown,
      shopper stays on the page, can retry.
- [ ] Switching methods and back keeps the typed fields.
- [ ] Terms and Conditions enabled: unticked blocks the order (and the wallet does not open).
- [ ] Full page load of the checkout: no CSP violation in the console.
- [ ] Express button: hidden when the method is disabled, the quote is virtual or the browser is not eligible; the
      Express Checkout card hides when no button renders.
- [ ] Customer order page shows the payment section via the method's info block.

## Reference

- [reference/renderer-examples.md](reference/renderer-examples.md) - complete form renderer (Vue + React), button and
  redirect shapes in both frameworks, offline method with its own field + data-assign observer.
- [reference/express-buttons.md](reference/express-buttons.md) - express button registry entry, provider, component
  sequence, Terms and Conditions, cart page.
- https://docs.oocscommerce.com/lynx/developer/payment-methods
