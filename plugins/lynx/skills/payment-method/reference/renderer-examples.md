# Payment renderer examples

All examples use `Vendor_PaymentFoo` / method `vendor_foo` (online) and `Vendor_PaymentInvoice` / `vendor_invoice`
(offline). Paths: `view/frontend/lynx/vue/src/...vue`.

## Form shape: card fields from a gateway SDK

The renderer mounts the field on mount, reports `loaded`, tokenizes in `preparePayment()`. `@vendor/foo-js` stands for
the gateway's npm SDK, declared in the module's own `view/frontend/lynx/vue/package.json`:

```json
{
  "name": "vendor-payment-foo-vue",
  "version": "1.0.0",
  "private": true,
  "dependencies": { "@vendor/foo-js": "<exact latest version>" }
}
```

Commit `package.json`; never run `npm install` there. Re-run `bin/magento lynx:theme:setup <themes>`: it installs the
SDK into each theme's `node_modules`.

### `Components/Foo.vue`

```vue
<script setup lang="ts">
import { onMounted, ref } from 'vue'
import { loadFoo, type FooCard } from '@vendor/foo-js'
import { useCheckoutStore } from 'OOCS_LynxCheckout/stores/checkoutStore'

const props = defineProps<{
    config: { publishableKey: string; labels?: Record<string, string> }
}>()

const checkout = useCheckoutStore()
const container = ref<HTMLElement>()
const loaded = ref(false)
const error = ref('')
let card: FooCard | null = null

onMounted(async () => {
    const foo = await loadFoo(props.config.publishableKey)
    card = foo.mountCard(container.value!)
    loaded.value = true
})

defineExpose({
    // A ref: the checkout sees it change from false to true
    loaded,
    async preparePayment(): Promise<Record<string, unknown>> {
        const result = await card!.tokenize()
        if (!result.token) {
            // The gateway's message, or Magento's generic required-field text
            error.value = result.message ?? props.config.labels?.required ?? ''
            throw new Error(error.value)
        }
        error.value = ''
        return { foo_token: result.token }
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

## Button shape: a wallet that places the order itself

```vue
<script setup lang="ts">
const props = defineProps<{
    config: { labels?: Record<string, string> }
    placeOrder: (additionalData?: Record<string, unknown>) => Promise<boolean>
    validate: () => boolean
}>()

defineExpose({ ownsAction: true })

async function onWalletClick() {
    // Do not open the wallet while the checkout still has errors
    if (!props.validate()) return
    const { token } = await window.FooWallet.open()
    await props.placeOrder({ foo_token: token })
}
</script>

<template>
    <button type="button" class="l-btn l-btn-primary w-full" @click="onWalletClick">
        {{ config.labels?.payWithFoo }}
    </button>
</template>
```

## Redirect shape: order first, then the hosted page

`config.paymentUrl` is your module's frontend controller (built with `UrlInterface::getUrl()` in `getConfig()`) that
answers `{ "url": "..." }` signed for the order.

```vue
<script setup lang="ts">
const props = defineProps<{
    config: { paymentUrl: string; labels?: Record<string, string> }
}>()

defineExpose({
    async afterPlaceOrder(orderId: string): Promise<string | null> {
        const response = await fetch(`${props.config.paymentUrl}?order_id=${encodeURIComponent(orderId)}`, {
            credentials: 'same-origin',
            headers: { 'X-Requested-With': 'XMLHttpRequest' },
        })
        const { url } = await response.json()
        return url || null
    },
})
</script>

<template>
    <p class="text-sm text-base-content/80">{{ config.labels?.redirect }}</p>
</template>
```

## Offline method with its own field

Lynx's own `OOCS_LynxCheckout/Components/Checkout/Payment/PurchaseOrder` (config
`OOCS\LynxCheckout\Model\Config\PurchaseOrder`) is built exactly like this. Here `vendor_invoice` asks for an
**Invoice reference**.

### `Model/Lynx/InvoiceConfig.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\PaymentInvoice\Model\Lynx;

use OOCS\LynxCheckout\Api\PaymentMethodRendererConfigInterface;

class InvoiceConfig implements PaymentMethodRendererConfigInterface
{
    /**
     * Labels of the Invoice reference field
     *
     * @return array
     */
    public function getConfig(): array
    {
        return [
            'labels' => [
                'reference' => (string) __('Invoice Reference'),
                'required' => (string) __('This is a required field.'),
            ],
        ];
    }
}
```

### `etc/frontend/di.xml`

```xml
<type name="OOCS\LynxCheckout\Provider\Checkout\Step\Payment">
    <arguments>
        <argument name="paymentMethodRenderers" xsi:type="array">
            <item name="vendor_invoice" xsi:type="array">
                <item name="component" xsi:type="string">Vendor_PaymentInvoice/Components/Checkout/Payment/Invoice</item>
                <item name="config" xsi:type="object">Vendor\PaymentInvoice\Model\Lynx\InvoiceConfig</item>
            </item>
        </argument>
    </arguments>
</type>
```

### `Components/Checkout/Payment/Invoice.vue`

```vue
<script setup lang="ts">
import { ref } from 'vue'
import { useCheckoutStore } from 'OOCS_LynxCheckout/stores/checkoutStore'

const props = defineProps<{
    config: { labels?: Partial<Record<'reference' | 'required', string>> }
}>()

const checkout = useCheckoutStore()
const reference = ref('')
const error = ref('')

defineExpose({
    preparePayment(): Promise<Record<string, unknown>> {
        const value = reference.value.trim()
        error.value = value ? '' : (props.config.labels?.required ?? '')
        return error.value ? Promise.reject(new Error(error.value)) : Promise.resolve({ invoice_reference: value })
    },
})
</script>

<template>
    <div>
        <label for="checkout-invoice-reference" class="block text-sm text-base-content mb-1.5">
            {{ config.labels?.reference }} <span class="text-error">*</span>
        </label>
        <input
            id="checkout-invoice-reference"
            v-model="reference"
            type="text"
            autocomplete="off"
            :class="checkout.inputClass(!!error)"
            @input="error = ''"
        />
        <p v-if="error" data-checkout-error class="mt-1 text-sm text-error">{{ error }}</p>
    </div>
</template>
```

### Keep the value: `etc/events.xml` + observer (plain Magento, works for Luma too)

```xml
<event name="payment_method_assign_data_vendor_invoice">
    <observer name="vendor_invoice_assign_reference" instance="Vendor\PaymentInvoice\Observer\AssignReference"/>
</event>
```

```php
<?php
declare(strict_types=1);

namespace Vendor\PaymentInvoice\Observer;

use Magento\Framework\Event\Observer;
use Magento\Payment\Observer\AbstractDataAssignObserver;
use Magento\Quote\Api\Data\PaymentInterface;

class AssignReference extends AbstractDataAssignObserver
{
    /**
     * Copy the invoice reference from additional_data onto the payment
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        $data = $this->readDataArgument($observer)->getData(PaymentInterface::KEY_ADDITIONAL_DATA);
        if (!empty($data['invoice_reference'])) {
            $this->readPaymentModelArgument($observer)
                ->setAdditionalInformation('invoice_reference', (string) $data['invoice_reference']);
        }
    }
}
```

Show it in the method's info block; Lynx's customer order page prints the info block HTML. For core's Purchase Order no
observer is needed: resolve `{ po_number }` and the step moves it to `paymentMethod.po_number`.
