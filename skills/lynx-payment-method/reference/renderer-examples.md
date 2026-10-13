# Payment renderer examples

All examples use `Vendor_PaymentFoo` / method `vendor_foo` (online) and `Vendor_PaymentInvoice` / `vendor_invoice`
(offline). Paths: `view/frontend/lynx/vue/src/...vue` and `view/frontend/lynx/react/src/...tsx`, same relative path.

## Form shape: card fields from a gateway SDK

The renderer mounts the field on mount, reports `loaded`, tokenizes in `preparePayment()`. `@vendor/foo-js` stands for
the gateway's npm SDK, declared in the module's own `view/frontend/lynx/<vue|react>/package.json`:

```json
{
  "name": "vendor-payment-foo-vue",
  "version": "1.0.0",
  "private": true,
  "dependencies": { "@vendor/foo-js": "<exact latest version>" }
}
```

Add `.gitignore` with `node_modules/` next to it; commit `package.json`. Re-run `bin/magento lynx:workspace:setup`.

### Vue - `Components/Foo.vue`

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

### React - `Components/Foo.tsx`

```tsx
import { useEffect, useImperativeHandle, useRef, useState, type Ref } from 'react'
import { loadFoo, type FooCard } from '@vendor/foo-js'
import { useCheckoutStore } from 'OOCS_LynxCheckout/stores/checkoutStore'

type Handle = {
    loaded: boolean
    preparePayment: () => Promise<Record<string, unknown>>
}

type Props = {
    config: { publishableKey: string; labels?: Record<string, string> }
    ref?: Ref<Handle>
}

export default function Foo({ config, ref }: Props) {
    const checkout = useCheckoutStore()
    const container = useRef<HTMLDivElement>(null)
    const card = useRef<FooCard | null>(null)
    const [loaded, setLoaded] = useState(false)
    const [error, setError] = useState('')

    useEffect(() => {
        void loadFoo(config.publishableKey).then(foo => {
            card.current = foo.mountCard(container.current!)
            setLoaded(true)
        })
    }, [config.publishableKey])

    useImperativeHandle(
        ref,
        () => ({
            loaded,
            async preparePayment() {
                const result = await card.current!.tokenize()
                if (!result.token) {
                    const message = result.message ?? config.labels?.required ?? ''
                    setError(message)
                    throw new Error(message)
                }
                setError('')
                return { foo_token: result.token }
            },
        }),
        // Rebuilt when loaded changes, so the checkout sees the new value
        [loaded, config.labels?.required]
    )

    return (
        <div>
            <label className="block text-sm text-base-content mb-1.5">{config.labels?.cardNumber}</label>
            <div ref={container} className={checkout.inputClass(!!error)}></div>
            {error && (
                <p data-checkout-error className="mt-1 text-sm text-error">
                    {error}
                </p>
            )}
        </div>
    )
}
```

## Button shape: a wallet that places the order itself

### Vue

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

### React

```tsx
import { useImperativeHandle, type Ref } from 'react'

type Props = {
    config: { labels?: Record<string, string> }
    placeOrder: (additionalData?: Record<string, unknown>) => Promise<boolean>
    validate: () => boolean
    ref?: Ref<{ ownsAction: boolean }>
}

export default function FooWallet({ config, placeOrder, validate, ref }: Props) {
    useImperativeHandle(ref, () => ({ ownsAction: true }), [])

    async function onWalletClick() {
        if (!validate()) return
        const { token } = await window.FooWallet.open()
        await placeOrder({ foo_token: token })
    }

    return (
        <button type="button" className="l-btn l-btn-primary w-full" onClick={onWalletClick}>
            {config.labels?.payWithFoo}
        </button>
    )
}
```

When the wallet's own button is drawn by an SDK (PayPal, Google Pay), render a skeleton (`l-skeleton`) until it is
ready and expose `loaded` as in the form shape. A wallet SDK that injects inline scripts needs `checkout.cspNonce`.

## Redirect shape: order first, then the hosted page

`config.paymentUrl` is your module's frontend controller (built with `UrlInterface::getUrl()` in `getConfig()`) that
answers `{ "url": "..." }` signed for the order.

### Vue

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

### React

```tsx
import { useImperativeHandle, type Ref } from 'react'

type Props = {
    config: { paymentUrl: string; labels?: Record<string, string> }
    ref?: Ref<{ afterPlaceOrder: (orderId: string) => Promise<string | null> }>
}

export default function FooRedirect({ config, ref }: Props) {
    useImperativeHandle(
        ref,
        () => ({
            async afterPlaceOrder(orderId: string) {
                const response = await fetch(`${config.paymentUrl}?order_id=${encodeURIComponent(orderId)}`, {
                    credentials: 'same-origin',
                    headers: { 'X-Requested-With': 'XMLHttpRequest' },
                })
                const { url } = await response.json()
                return url || null
            },
        }),
        [config.paymentUrl]
    )

    return <p className="text-sm text-base-content/80">{config.labels?.redirect}</p>
}
```

The URL controller must check that the order belongs to the current session (e.g. the checkout session's last real
order id) before signing. The return controller verifies the gateway signature, restores the quote on cancel and
redirects to `checkout/onepage/success` or `checkout/onepage/failure`. Only add a Lynx route if the return page itself
must render in the Lynx theme (see **lynx-basics** / Inertia pages).

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

### Vue - `Components/Checkout/Payment/Invoice.vue`

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

### React - `Components/Checkout/Payment/Invoice.tsx`

```tsx
import { useImperativeHandle, useRef, useState, type Ref } from 'react'
import { useCheckoutStore } from 'OOCS_LynxCheckout/stores/checkoutStore'

type Props = {
    config: { labels?: Partial<Record<'reference' | 'required', string>> }
    ref?: Ref<{ preparePayment: () => Promise<Record<string, unknown>> }>
}

export default function Invoice({ config, ref }: Props) {
    const checkout = useCheckoutStore()
    const [reference, setReference] = useState('')
    const [error, setError] = useState('')
    // preparePayment runs after the render that produced it
    const value = useRef(reference)
    value.current = reference

    useImperativeHandle(
        ref,
        () => ({
            preparePayment(): Promise<Record<string, unknown>> {
                const current = value.current.trim()
                const message = current ? '' : (config.labels?.required ?? '')
                setError(message)
                return message ? Promise.reject(new Error(message)) : Promise.resolve({ invoice_reference: current })
            },
        }),
        [config.labels?.required]
    )

    return (
        <div>
            <label htmlFor="checkout-invoice-reference" className="block text-sm text-base-content mb-1.5">
                {config.labels?.reference} <span className="text-error">*</span>
            </label>
            <input
                id="checkout-invoice-reference"
                value={reference}
                type="text"
                autoComplete="off"
                className={checkout.inputClass(!!error)}
                onChange={e => {
                    setReference(e.target.value)
                    setError('')
                }}
            />
            {error && (
                <p data-checkout-error className="mt-1 text-sm text-error">
                    {error}
                </p>
            )}
        </div>
    )
}
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
