# Worked example: gift card type `vendor_giftcard`

Extension `Vendor_GiftCard` defines type `vendor_giftcard` (class `Vendor\GiftCard\Model\Product\Type\GiftCard`,
constant `TYPE_CODE`, method `getAmounts($product)` returning floats). Its buy request reads `giftcard_amount`,
`giftcard_recipient_name`, `giftcard_recipient_email`, `giftcard_message`. The Lynx front goes into
`Vendor_GiftCardLynx`. Replace `getAmounts()` and the field names with what the real type offers.

## Option b

Keeps `default_product_detail_simple` (its `availableOn` extended as in SKILL.md step 3a, like Product and Product
Media) and adds a Gift Card Options block in the Product Options slot.

### `etc/components/GiftCardOptions.graphqls`

As in SKILL.md step 3b: `VendorGiftCardOptions` (code `vendor_giftcard_options`) with `product_id`, `amounts`
(`[VendorGiftCardAmount!]!`, each `value` and `price`) and `labels`, all `@optional`.

### Provider `Provider/GiftCardOptions.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\GiftCardLynx\Provider;

use Magento\Catalog\Model\Product;
use Magento\Framework\Pricing\PriceCurrencyInterface;
use Magento\Framework\Registry;
use OOCS\LynxComponent\Model\AbstractComponentProvider;
use Vendor\GiftCard\Model\Product\Type\GiftCard;

class GiftCardOptions extends AbstractComponentProvider
{
    public function __construct(
        private readonly Registry $registry,
        private readonly PriceCurrencyInterface $priceCurrency,
    ) {
    }

    public function resolve(array $props): array
    {
        /** @var Product|null $product */
        $product = $this->registry->registry('current_product');
        if (!$product || $product->getTypeId() !== GiftCard::TYPE_CODE) {
            return $props;
        }

        /** @var GiftCard $type */
        $type = $product->getTypeInstance();

        // product_id keys this block's share of the request in the product store
        $props['product_id'] = (int) $product->getId();
        $props['amounts'] = array_map(
            fn (float $amount): array => [
                // Posted as the type reads it
                'value' => $amount,
                // Shown and added to the displayed price, in the current currency
                'price' => (float) $this->priceCurrency->convert($amount),
            ],
            $type->getAmounts($product)
        );
        $props['labels'] = [
            'amount'         => (string) __('Amount'),
            'recipientName'  => (string) __('Recipient Name'),
            'recipientEmail' => (string) __('Recipient Email'),
            'message'        => (string) __('Message'),
        ];

        return $props;
    }
}
```

### `view/frontend/lynx/vue/src/Components/GiftCardOptions.vue`

```vue
<script setup lang="ts">
import { computed, onUnmounted, ref, watch } from 'vue'
import { useProductStore } from 'OOCS_LynxCatalog/stores/productStore'
import { formatPrice } from 'OOCS_LynxTheme/utils/price'

const props = withDefaults(defineProps<Lynx.VendorGiftCardOptions>(), {
    product_id: 0,
    amounts: () => [],
    labels: () => ({}),
})

// This block's key among the contributions to the product's request
const CONTRIBUTOR = 'vendor_giftcard'

const optionsStore = useProductStore()
const amount = ref<Lynx.VendorGiftCardAmount | null>(null)
const name = ref('')
const email = ref('')
const message = ref('')
const section = ref<HTMLElement | null>(null)

const missing = computed(() =>
    [!amount.value && props.labels.amount, !name.value.trim() && props.labels.recipientName,
        !email.value.trim() && props.labels.recipientEmail].filter((title): title is string => !!title)
)

watch(
    [amount, name, email, message, missing],
    () =>
        optionsStore.contribute(props.product_id, CONTRIBUTOR, {
            fields: {
                giftcard_amount: amount.value ? String(amount.value.value) : '',
                giftcard_recipient_name: name.value,
                giftcard_recipient_email: email.value,
                giftcard_message: message.value,
            },
            files: {},
            price: amount.value?.price ?? 0,
            missing: missing.value,
        }),
    { immediate: true }
)

// Add to Cart was pressed with a field empty
watch(
    () => optionsStore.validationTicks[props.product_id],
    tick => tick && section.value?.querySelector<HTMLElement>('input')?.focus()
)

// An Inertia visit to another product can keep the block mounted
watch(
    () => props.product_id,
    (id, previous) => previous && previous !== id && optionsStore.forget(previous, CONTRIBUTOR)
)
onUnmounted(() => optionsStore.forget(props.product_id, CONTRIBUTOR))
</script>

<template>
    <section ref="section" class="border-0 border-b border-solid border-base-300 py-4 grid gap-4">
        <fieldset class="grid gap-2">
            <legend class="mb-2 font-semibold text-base-content">{{ labels.amount }}</legend>
            <div class="flex flex-wrap gap-2">
                <button
                    v-for="option in amounts"
                    :key="option.value"
                    type="button"
                    class="l-btn"
                    :class="amount?.value === option.value ? 'l-btn-primary' : 'l-btn-outline'"
                    :aria-pressed="amount?.value === option.value"
                    @click="amount = option"
                >
                    {{ formatPrice(option.price) }}
                </button>
            </div>
        </fieldset>
        <label class="grid gap-1">
            <span class="text-sm">{{ labels.recipientName }}</span>
            <input v-model="name" type="text" required class="l-input w-full" />
        </label>
        <label class="grid gap-1">
            <span class="text-sm">{{ labels.recipientEmail }}</span>
            <input v-model="email" type="email" required class="l-input w-full" />
        </label>
        <label class="grid gap-1">
            <span class="text-sm">{{ labels.message }}</span>
            <textarea v-model="message" rows="3" class="l-textarea w-full" />
        </label>
    </section>
</template>
```

### `view/frontend/lynx/react/src/Components/GiftCardOptions.tsx`

```tsx
import { useEffect, useMemo, useRef, useState } from 'react'
import { useProductStore } from 'OOCS_LynxCatalog/stores/productStore'
import { formatPrice } from 'OOCS_LynxTheme/utils/price'

type Props = Lynx.VendorGiftCardOptions

// This block's key among the contributions to the product's request
const CONTRIBUTOR = 'vendor_giftcard'
const EMPTY_LIST: never[] = []
const EMPTY_MAP: Record<string, never> = {}

export default function GiftCardOptions({ product_id = 0, amounts = EMPTY_LIST, labels = EMPTY_MAP }: Props) {
    const optionsStore = useProductStore()
    // The store object changes with every update; effects read the current one
    const storeRef = useRef(optionsStore)
    storeRef.current = optionsStore

    const [amount, setAmount] = useState<Lynx.VendorGiftCardAmount | null>(null)
    const [name, setName] = useState('')
    const [email, setEmail] = useState('')
    const [message, setMessage] = useState('')
    const section = useRef<HTMLElement>(null)

    const missing = useMemo(
        () =>
            [!amount && labels.amount, !name.trim() && labels.recipientName, !email.trim() && labels.recipientEmail]
                .filter((title): title is string => !!title),
        [amount, name, email, labels]
    )

    useEffect(() => {
        storeRef.current.contribute(product_id, CONTRIBUTOR, {
            fields: {
                giftcard_amount: amount ? String(amount.value) : '',
                giftcard_recipient_name: name,
                giftcard_recipient_email: email,
                giftcard_message: message,
            },
            files: {},
            price: amount?.price ?? 0,
            missing,
        })
    }, [product_id, amount, name, email, message, missing])

    // Withdraw from the product just left, and when the block goes away
    useEffect(() => () => storeRef.current.forget(product_id, CONTRIBUTOR), [product_id])

    // Add to Cart was pressed with a field empty
    const tick = optionsStore.validationTicks[product_id]
    useEffect(() => {
        if (tick) section.current?.querySelector('input')?.focus()
    }, [tick])

    return (
        <section ref={section} className="border-0 border-b border-solid border-base-300 py-4 grid gap-4">
            <fieldset className="grid gap-2">
                <legend className="mb-2 font-semibold text-base-content">{labels.amount}</legend>
                <div className="flex flex-wrap gap-2">
                    {amounts.map(option => (
                        <button
                            key={option.value}
                            type="button"
                            className={`l-btn ${amount?.value === option.value ? 'l-btn-primary' : 'l-btn-outline'}`}
                            aria-pressed={amount?.value === option.value}
                            onClick={() => setAmount(option)}
                        >
                            {formatPrice(option.price)}
                        </button>
                    ))}
                </div>
            </fieldset>
            <label className="grid gap-1">
                <span className="text-sm">{labels.recipientName}</span>
                <input value={name} onChange={e => setName(e.target.value)} type="text" required className="l-input w-full" />
            </label>
            <label className="grid gap-1">
                <span className="text-sm">{labels.recipientEmail}</span>
                <input value={email} onChange={e => setEmail(e.target.value)} type="email" required className="l-input w-full" />
            </label>
            <label className="grid gap-1">
                <span className="text-sm">{labels.message}</span>
                <textarea value={message} onChange={e => setMessage(e.target.value)} rows={3} className="l-textarea w-full" />
            </label>
        </section>
    )
}
```

Add `Amount`, `Recipient Name`, `Recipient Email`, `Message` to `i18n/en_US.csv`. The Product Detail does the rest:
blocking the POST while a field is empty, the POST, the response messages and the section reload.

## Option c

### `etc/components/GiftCard.graphqls`

```graphql
"Product detail page for a gift card: amount, recipient and message"
type VendorProductDetailGiftCard @component(
  code: "vendor_product_detail_giftcard"
  template: "Vendor_GiftCardLynx::Components/Product/GiftCard"
  provider: "Vendor\\GiftCardLynx\\Provider\\Product\\Detail\\GiftCard"
  label: "Product Detail (Gift Card)"
  group: "product_page"
  icon: "ph-tag"
  zones: ["content"]
  availableOn: ["catalog_product_view_type_vendor_giftcard"]
) @inherit(from: "ProductDetailSimple", except: ["original_price", "is_on_sale", "tier_prices"]) {
  amounts: [VendorGiftCardDetailAmount!]! @optional
}

type VendorGiftCardDetailAmount {
  value: Float!
}
```

`@inherit` copies from `ProductDetailSimple` (`module-lynx-catalog/etc/components/Simple.graphqls`) the `show_wishlist` /
`show_compare` settings that `AbstractDetail` reads (the wish list / compare URL is sent only when on), the slots
`price_after` "Below Price", `product_info_additional` (stored as `product.info.additional`, "Product Options",
`product_option` only, `max: 1`) and `info_after` "Below Add to Cart" with their full `allow` lists, and the data
fields `AbstractDetail` sends (`product_id`, `title`, `sku`, `price`, `is_salable`, `rating`, `urls`, `sections`,
`labels`). `except` drops what this provider never sends (no `addQuantityProps()` / `addRegularPrice()`). Declare
anything else your `addTypeProps()` returns, or the server drops it. The gallery and its Below Gallery slot belong to
Product Media, the breadcrumbs to Product: both already list your path (SKILL.md step 3).

### Component changes

Copy `Components/Product/Simple.vue` and `Simple.tsx` from
`module-lynx-catalog/view/frontend/lynx/<vue|react>/src/` to
`Vendor/GiftCardLynx/view/frontend/lynx/<vue|react>/src/Components/Product/GiftCard.*`. Keep price, rating, the
wish list and compare buttons (rendered only when `urls.add_to_wishlist` / `urls.add_to_compare` is present),
analytics and the three slots. Drop the tier prices and the struck-through price (the provider does not call
`addQuantityProps()` / `addRegularPrice()`). Change the quantity section and `addToCart()`:


```vue
<script setup lang="ts">
// Imports and the wish list, compare and analytics code of Simple.vue stay as they are

const props = withDefaults(defineProps<Lynx.VendorProductDetailGiftCard>(), {
    /* …defaults of Simple.vue, plus: */ amounts: () => [],
})

const amount = ref<number | null>(null)
const recipient = ref({ name: '', email: '' })
// Shows labels.requiredField under the empty fields once Add to Cart was pressed
const submitted = ref(false)

async function addToCart() {
    if (adding.value) return

    const { name, email } = recipient.value
    if (amount.value === null || !name.trim() || !email.trim() || chosenOptions.value.missing.length) {
        submitted.value = true
        optionsStore.requestValidation(props.product_id)
        return
    }

    const body = new URLSearchParams({
        product: String(props.product_id),
        qty: '1',
        form_key: formKeyStore.ensure(),
        giftcard_amount: String(amount.value),
        giftcard_recipient_name: name,
        giftcard_recipient_email: email,
    })

    // From here on as in Simple.vue: custom option fields and uploads, fetch(props.urls.add_to_cart, …),
    // the JSON result: messages, then analytics + customerDataStore.reload(props.sections.add_to_cart) on success, backUrl
}
</script>
```

```tsx
// Imports and the wish list, compare and analytics code of Simple.tsx stay as they are
type Props = Lynx.VendorProductDetailGiftCard & { slots?: NodeSlots }

export default function GiftCard({ amounts = EMPTY_LIST, /* …as Simple.tsx */ }: Props) {
    const [amount, setAmount] = useState<number | null>(null)
    const [recipient, setRecipient] = useState({ name: '', email: '' })
    // Shows labels.requiredField under the empty fields once Add to Cart was pressed
    const [submitted, setSubmitted] = useState(false)

    async function addToCart() {
        if (adding) return

        if (amount === null || !recipient.name.trim() || !recipient.email.trim() || chosenOptions.missing.length) {
            setSubmitted(true)
            optionsStore.requestValidation(product_id)
            return
        }

        const body = new URLSearchParams({
            product: String(product_id),
            qty: '1',
            form_key: formKeyStore.ensure(),
            giftcard_amount: String(amount),
            giftcard_recipient_name: recipient.name,
            giftcard_recipient_email: recipient.email,
        })

        // From here on as in Simple.tsx: custom option fields and uploads, fetch(urls.add_to_cart, …),
        // the JSON result: messages, then analytics + customerDataStore.reload(sections.add_to_cart) on success, backUrl
    }

    // …markup of Simple.tsx with the amount and recipient fields in place of the quantity section
}
```


Built-in components post the same endpoint for every type; only params differ (`super_attribute[<id>]`
configurable, `bundle_option[<id>][]` bundle, `super_group[<id>]` grouped, `links[]` downloadable).

## Starting layout

`Setup/Patch/Data/InstallGiftCardLayout.php` — copies the simple product's default layout (a Group holding Product,
whose Images slot has Product Media and whose Details slot has Product Detail with Product Custom Options; then the
Tabs group and Related Products) and adds the options node:

```php
<?php
declare(strict_types=1);

namespace Vendor\GiftCardLynx\Setup\Patch\Data;

use Magento\Framework\Serialize\Serializer\Json;
use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\Patch\DataPatchInterface;
use OOCS\LynxCatalog\Setup\Patch\Data\InstallCatalogLayouts;
use OOCS\LynxPageBuilder\Model\Design;

class InstallGiftCardLayout implements DataPatchInterface
{
    private const TABLE = 'lynx_pagebuilder_layout';

    public function __construct(
        private readonly ModuleDataSetupInterface $moduleDataSetup,
        private readonly Json $json,
    ) {
    }

    public static function getDependencies(): array
    {
        // The simple layout to copy is written there
        return [InstallCatalogLayouts::class];
    }

    public function getAliases(): array
    {
        return [];
    }

    public function apply(): self
    {
        $connection = $this->moduleDataSetup->getConnection();
        $table = $this->moduleDataSetup->getTable(self::TABLE);

        $value = (string) $connection->fetchOne(
            $connection->select()
                ->from($table, 'value')
                ->where('design_id = ?', Design::DEFAULT_ID)
                ->where('path = ?', 'catalog_product_view_type_simple')
        );
        if ($value === '') {
            return $this;
        }

        // Group > Product > Details slot > Product Detail; its Product Options slot holds Product Custom Options
        $nodes = $this->json->unserialize($value);
        $detail = &$nodes[0]['slots']['default'][0]['slots']['detail'][0];
        $detail['slots']['product.info.additional'][] = [
            'name'      => 'vendor_giftcard_options',
            'component' => 'Vendor_GiftCardLynx/Components/GiftCardOptions',
            'label'     => 'Gift Card Options',
            'props'     => [],
            'visible'   => true,
        ];
        unset($detail);

        $connection->insertOnDuplicate($table, [[
            'design_id' => Design::DEFAULT_ID,
            'path'      => 'catalog_product_view_type_vendor_giftcard',
            'value'     => $this->json->serialize($nodes),
        ]], ['value']);

        return $this;
    }
}
```

For option c, instead of appending the options node set `$detail['name'] = 'vendor_product_detail_giftcard'`,
`$detail['component'] = 'Vendor_GiftCardLynx/Components/Product/GiftCard'` and
`$detail['label'] = 'Product Detail (Gift Card)'`.

Layouts are cached in the `lynx_component` cache type and pages in the full page cache: clean both after the patch.
Without a patch, merchants open **Products › Gift card product page** in the editor and build the page in their own
design (the default design is read-only).

## Changing one built-in type instead

```xml
<type name="OOCS\LynxCatalog\Provider\Product\Detail\Configurable">
    <plugin name="vendor_module_size_guide" type="Vendor\Module\Plugin\AddSizeGuide"/>
</type>
```

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin;

use OOCS\LynxCatalog\Provider\Product\Detail\Configurable;

class AddSizeGuide
{
    public function afterResolve(Configurable $subject, array $result, array $props): array
    {
        // Not on a product page: the provider returned the node's settings
        if (!isset($result['product_id'])) {
            return $result;
        }
        $result['size_guide_url'] = '/size-guide';

        return $result;
    }
}
```

Read the new prop in a theme override of `OOCS_LynxCatalog/lynx/<vue|react>/src/Components/Product/Configurable.*`
(see `theme`).
