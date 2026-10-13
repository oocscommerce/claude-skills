---
name: shipping-method
description: Adds a shipping method renderer to the Lynx one-page checkout (OOCS_LynxCheckout) - a Vue and React component shown under a carrier's rate in the Shipping Method step when the shopper picks it (store or locker picker, delivery date, time slot), registered in the `shippingMethodRenderers` pool by carrier code, fed by a PHP config class, exposing `validate` / `data` / `summary`, with its data written to the quote by an idempotent `beforeSave` plugin on the ShippingMethod step provider. Use when a developer says "shipping renderer for Lynx checkout", "my carrier needs a pickup point / delivery date in Lynx", "make my store pickup / locker module work with Lynx", "save extra shipping data from the Lynx checkout". Not for a question every shopper answers (use checkout-step) or payment input (use payment-method). Use with the basics skill.
---

# Lynx shipping method renderer

Produces the Lynx side of a carrier that needs input from the shopper: a renderer (Vue + React) under the carrier's
rate, its pool entry, a config class, and a `beforeSave` plugin that writes the renderer's data to the quote. The
carrier itself (rates, `config.xml`, `system.xml`) stays a standard Magento carrier; every carrier already works in
Lynx without a renderer.

## Before you start

- Read **basics** (module layout, `view/frontend/lynx/<vue|react>/src`, aliases, build). The Lynx reference
  implementation is `OOCS_LynxInStorePickup` (in
  `vendor/oocscommerce/module-lynx-in-store-pickup`): `Model\PickupLocations` (config), `Plugin\SavePickupLocation` (beforeSave),
  `Components/StorePickup.{vue,tsx}`.
- Find the **carrier code** (the first segment of the rate code: rate `instore_pickup` = carrier `instore`, method
  `pickup`). The pool is keyed by carrier code, so one renderer serves every method of the carrier.
- Decide where the data lives on the quote (a quote / quote address column via `db_schema.xml`, an extension
  attribute, or the shipping address itself as In-Store Pickup does) and how it reaches the order (fieldset
  `sales_convert_quote`, observer). That part is plain Magento.
- Code to read when in doubt: `OOCS\LynxCheckout\Provider\Checkout\Step\ShippingMethod` (`save()`, `rendererOf()`)
  and `OOCS_LynxCheckout/Components/Checkout/Step/ShippingMethod.{vue,tsx}`.

## Steps

### 1. Module wiring

`etc/module.xml` sequences `OOCS_LynxCheckout` (plus the carrier's module); `composer.json` requires
`oocscommerce/module-lynx-checkout`.

### 2. Register the renderer and the plugin - `etc/frontend/di.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="OOCS\LynxCheckout\Provider\Checkout\Step\ShippingMethod">
        <arguments>
            <argument name="shippingMethodRenderers" xsi:type="array">
                <!-- key = carrier code -->
                <item name="vendorfoo" xsi:type="array">
                    <item name="component" xsi:type="string">Vendor_ShippingFoo/Components/DeliveryDate</item>
                    <item name="config" xsi:type="object">Vendor\ShippingFoo\Model\Config\DeliveryDate</item>
                </item>
            </argument>
        </arguments>
        <plugin name="vendor_shippingfoo_save_delivery_date"
                type="Vendor\ShippingFoo\Plugin\SaveDeliveryDate" sortOrder="20"/>
    </type>
</config>
```

| Key | Meaning |
|---|---|
| `component` | `<Module>/<path under src>` without extension, **slash** after the module name. Resolves to `.vue` on a Vue theme, `.tsx` on a React theme |
| `config` | Object implementing `OOCS\LynxCheckout\Api\ShippingMethodRendererConfigInterface`; `getConfig()` becomes the `config` prop. A wrong class throws a `LogicException` |

A renderer is attached only to rates the carrier actually quoted (`available`); error rates get none.

### 3. The config class - `Model/Config/DeliveryDate.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\ShippingFoo\Model\Config;

use Magento\Checkout\Model\Session as CheckoutSession;
use OOCS\LynxCheckout\Api\ShippingMethodRendererConfigInterface;

class DeliveryDate implements ShippingMethodRendererConfigInterface
{
    /**
     * @param CheckoutSession $checkoutSession
     */
    public function __construct(
        private readonly CheckoutSession $checkoutSession,
    ) {
    }

    /**
     * The saved date, the earliest date and the labels
     *
     * @return array
     */
    public function getConfig(): array
    {
        return [
            // What the quote already holds, so going back to the step shows it
            'selected' => (string) $this->checkoutSession->getQuote()->getData('vendor_foo_delivery_date'),
            'min' => date('Y-m-d', strtotime('+1 day')),
            'labels' => [
                'deliveryDate' => (string) __('Delivery Date'),
                'required' => (string) __('This is a required field.'),
            ],
        ];
    }
}
```

- `getConfig()` runs only when the step is fetched and only for a carrier that returned a rate for the current
  address, so reading the quote here is safe and cheap.
- Return the value already on the quote (`selected`) so **Edit**, back navigation and reloads show it.
- Do not return long lists (all stores, all slots): return what the renderer needs to search (a REST URL, the website
  code, the cart SKUs) as `PickupLocations::getConfig()` does.
- Every visible string under `labels`, translated with `__()`.

### 4. The renderer

`view/frontend/lynx/vue/src/Components/DeliveryDate.vue` and `view/frontend/lynx/react/src/Components/DeliveryDate.tsx`.

**Props in:** `rate` (`carrier_code`, `method_code`, `carrier_title`, `method_title`, `amount`, `available`,
`error_message`) and `config` (what `getConfig()` returned).

**Exposed out** (Vue `defineExpose`, React `useImperativeHandle(ref, ...)` on the `ref` prop). All optional:

| Member | Type | Meaning |
|---|---|---|
| `validate` | `() => boolean` | Called on **Continue**; `false` keeps the shopper on the step. The renderer shows its own error |
| `data` | `() => Record<string, FormDataConvertible>` | Merged into the step's `data()`, posted with every step request |
| `summary` | `string` | One line under the chosen method in the collapsed step (the store name, the date) |

Mount lifecycle (from `Step/ShippingMethod.{vue,tsx}`):

- Mounted only while its rate is the picked one; switching rates unmounts it, and it unmounts when the step collapses.
- On **Continue** the step calls `validate()`, then snapshots `data()` and `summary`, tracks `add_shipping_info`, and
  calls `checkout.nextStep()`. While mounted the live `data()` is used; after unmount the snapshot is, so your keys keep
  reaching the server on later step requests.
- Because it remounts, initialise state from `config` (the value on the quote), never assume it persists.

Vue (the React twin is in [reference/delivery-date-example.md](reference/delivery-date-example.md)):

```vue
<script setup lang="ts">
import { ref } from 'vue'
import { useCheckoutStore } from 'OOCS_LynxCheckout/stores/checkoutStore'

const props = defineProps<{
    rate: { carrier_code: string; method_code: string }
    config: { selected?: string; min?: string; labels?: Record<string, string> }
}>()

const checkout = useCheckoutStore()
const date = ref(props.config.selected ?? '')
const error = ref('')

defineExpose({
    validate() {
        error.value = date.value ? '' : (props.config.labels?.required ?? '')
        return !error.value
    },
    data: () => ({ vendorFooDeliveryDate: date.value }),
    // A ref is unwrapped on the exposed instance, so the step reads the current value
    summary: date,
})
</script>

<template>
    <div>
        <label for="vendor-foo-date" class="block text-sm text-base-content mb-1.5">
            {{ config.labels?.deliveryDate }} <span class="text-error">*</span>
        </label>
        <input
            id="vendor-foo-date"
            v-model="date"
            type="date"
            :min="config.min"
            :class="checkout.inputClass(!!error)"
            @input="error = ''"
        />
        <p v-if="error" data-checkout-error class="mt-1 text-sm text-error">{{ error }}</p>
    </div>
</template>
```

In Vue a derived summary is exposed as a `computed` (`summary: computed(() => selected.value?.name ?? '')` in
StorePickup.vue). In React put the plain value in the handle and its source in the deps.

### 5. Save the data - `Plugin/SaveDeliveryDate.php`

The step's own `save()` (`CheckoutStepInterface`) writes only the carrier and method, and returns early when the quote
already has that method. Your `beforeSave` plugin runs before it, each time the step **after** Shipping Method is
fetched (Continue, and again after Edit / back navigation / an Inertia reload of that step).

```php
<?php
declare(strict_types=1);

namespace Vendor\ShippingFoo\Plugin;

use Magento\Checkout\Model\Session as CheckoutSession;
use Magento\Quote\Api\CartRepositoryInterface;
use OOCS\LynxCheckout\Provider\Checkout\Step\ShippingMethod;

class SaveDeliveryDate
{
    /**
     * @param CheckoutSession $checkoutSession
     * @param CartRepositoryInterface $quoteRepository
     */
    public function __construct(
        private readonly CheckoutSession $checkoutSession,
        private readonly CartRepositoryInterface $quoteRepository,
    ) {
    }

    /**
     * Write the delivery date to the quote before the step saves the carrier
     *
     * @param ShippingMethod $subject
     * @param array $draft
     * @return array
     */
    public function beforeSave(ShippingMethod $subject, array $draft): array
    {
        $date = (string) ($draft['vendorFooDeliveryDate'] ?? '');
        if (!str_starts_with((string) ($draft['shippingMethod'] ?? ''), 'vendorfoo_') || $date === '') {
            return [$draft];
        }

        $quote = $this->checkoutSession->getQuote();
        // Idempotent: nothing to write when the quote already holds it
        if (!$quote->getId() || $quote->isVirtual() || $quote->getData('vendor_foo_delivery_date') === $date) {
            return [$draft];
        }

        $quote->setData('vendor_foo_delivery_date', $date);
        $this->quoteRepository->save($quote);

        return [$draft];
    }
}
```

- `$draft` is the merge of every step's `data()`: `shippingAddress`, `shippingMethod` (`<carrier>_<method>`), your
  keys. It comes from the browser - validate types and values (e.g. the date format, a store code that exists).
- Throw `Magento\Framework\Exception\LocalizedException` with a translated message to reject: the shopper is sent
  back to the Shipping Method step and sees it as an error message.
- Check the shipping method first: when the shopper switches to another carrier your keys may still be in the draft
  (the snapshot), so act only for your carrier. Clear stale values if switching away must remove them.
- If your carrier needs the shipping address replaced (pickup point), do what `SavePickupLocation` does: build the
  address, call `ShippingInformationManagementInterface::saveAddressInformation()` with your carrier/method, so the
  step's own `save()` then finds the method already set and does nothing. The Address step rewrites the shopper's
  address whenever it changes, so compare both the method and your code before skipping.

### 6. Build

```bash
bin/magento module:enable Vendor_ShippingFoo && bin/magento setup:upgrade
bin/magento lynx:workspace:setup OOCS_LynxVue OOCS_LynxReact   # new module
bin/magento lynx:static:deploy
bin/magento lynx:compile                                       # if the store uses compiled config
bin/magento cache:clean
```

## Rules

- **Pool scope: `etc/frontend/di.xml`.** Same reason as payment renderers: an area's array argument replaces the global
  one per argument name (`array_replace`), and Lynx's own entry (`instore`) is in `etc/frontend/di.xml`. The plugin can
  sit in the same file.
- **Keys of `data()` must not clash**: prefix with your vendor (`vendorFooDeliveryDate`; In-Store Pickup uses
  `pickupLocationCode`). Never reuse `shippingAddress`, `shippingMethod`, `email`, `billingAddress`.
- **The server enforces, the renderer helps.** `validate()` is convenience; the plugin must reject bad input itself.
- **Idempotent and cheap plugin**: it runs on every later step fetch. Compare with the quote and return early.
- **One input per carrier belongs here; input every shopper gives belongs in a checkout step** (**checkout-step**).
- **No UI text in the component**: labels from `config.labels`; reuse Magento phrases.
- **Ship `.vue` and `.tsx`** at the same relative path. Styling: `checkout.inputClass(invalid)`, tokens and `l-`
  classes only, nothing below `text-sm`, inline errors with `data-checkout-error`.
- **External scripts / map SDKs**: `etc/csp_whitelist.xml` + `checkout.cspNonce`; load your own code with `import`
  (the checkout pins chunk hashes).
- **Custom address attributes** need no renderer: the merchant adds them to the Address step's **Address Fields**
  setting (attribute in form `customer_address_edit`); they are sent as `custom_attributes`.

## Check

- [ ] Picking the rate shows the renderer (Vue **and** React theme); another rate hides it; its chunk loads lazily.
- [ ] Continue without input: renderer error, shopper stays.
- [ ] Continue with input: collapsed step shows `summary`; the value is on the quote (DB) after the Payment step loads.
- [ ] Edit Shipping Method, Continue again without changes: no extra write (plugin returns early).
- [ ] Switch to another carrier: your plugin does nothing (or clears the value, if designed so).
- [ ] Invalid value posted (tamper the request): `LocalizedException` message, back on the step.
- [ ] Reload mid-checkout: the renderer shows the saved value from `config`.
- [ ] Order placed carries the value; guest and signed-in customer; virtual cart (step skipped, plugin inert).

## Reference

- [reference/delivery-date-example.md](reference/delivery-date-example.md) - full example (React renderer, storage,
  order copy) and how In-Store Pickup uses each piece.
- **checkout-step** - the step flow, `Process` controller and `CheckoutStepInterface`.
- https://docs.oocscommerce.com/lynx/developer/shipping-methods
