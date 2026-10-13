# Worked example: gift card type `vendor_giftcard`

Extension `Vendor_GiftCard` defines type `vendor_giftcard` (class `Vendor\GiftCard\Model\Product\Type\GiftCard`,
constant `TYPE_CODE`, method `getAmounts($product)` returning floats). Its buy request reads `giftcard_amount`,
`giftcard_recipient_name`, `giftcard_recipient_email`, `giftcard_message`. The Lynx front goes into
`Vendor_GiftCardLynx`. Replace `getAmounts()` and the field names with what the real type offers.

## Option b

Keeps `default_product_detail_simple` (its `available_on` extended as in SKILL.md step 3a) and adds a Gift Card
Options block in the Product Options slot.

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

        // product_id keys this block's share of the request in productOptionsStore
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
import { useProductOptionsStore } from 'OOCS_LynxCatalog/stores/productOptionsStore'
import { formatPrice } from 'OOCS_LynxTheme/utils/price'

type Amount = { value: number; price: number }

const props = withDefaults(
    defineProps<{ product_id?: number; amounts?: Amount[]; labels?: Record<string, string> }>(),
    { product_id: 0, amounts: () => [], labels: () => ({}) }
)

// This block's key among the contributions to the product's request
const CONTRIBUTOR = 'vendor_giftcard'

const optionsStore = useProductOptionsStore()
const amount = ref<Amount | null>(null)
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
import { useProductOptionsStore } from 'OOCS_LynxCatalog/stores/productOptionsStore'
import { formatPrice } from 'OOCS_LynxTheme/utils/price'

type Amount = { value: number; price: number }
type Props = { product_id?: number; amounts?: Amount[]; labels?: Record<string, string> }

// This block's key among the contributions to the product's request
const CONTRIBUTOR = 'vendor_giftcard'
const EMPTY_LIST: never[] = []
const EMPTY_MAP: Record<string, never> = {}

export default function GiftCardOptions({ product_id = 0, amounts = EMPTY_LIST, labels = EMPTY_MAP }: Props) {
    const optionsStore = useProductOptionsStore()
    // The store object changes with every update; effects read the current one
    const storeRef = useRef(optionsStore)
    storeRef.current = optionsStore

    const [amount, setAmount] = useState<Amount | null>(null)
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
required-field message, the POST, the response messages and the section reload.

## Option c

### Registry entry (global `etc/di.xml`, inside `OOCS\LynxComponent\Model\Registry` → `components`)

```xml
<item name="vendor_product_detail_giftcard" xsi:type="array">
    <item name="component" xsi:type="string">Vendor_GiftCardLynx::Components/Product/GiftCard</item>
    <item name="provider" xsi:type="object">Vendor\GiftCardLynx\Provider\Product\Detail\GiftCard\Proxy</item>
    <item name="label" xsi:type="string">Product Detail (Gift Card)</item>
    <item name="description" xsi:type="string">Product detail page for a gift card: amount, recipient and message</item>
    <item name="icon" xsi:type="string">ph-tag</item>
    <item name="group" xsi:type="string">product_page</item>
    <item name="zones" xsi:type="array">
        <item name="content" xsi:type="string">content</item>
    </item>
    <item name="available_on" xsi:type="array">
        <item name="vendor_giftcard" xsi:type="string">catalog_product_view_type_vendor_giftcard</item>
    </item>
    <item name="slots" xsi:type="array">
        <item name="gallery_after" xsi:type="array">
            <item name="label" xsi:type="string">Below Gallery</item>
            <item name="allowed_groups" xsi:type="array">
                <item name="layout" xsi:type="string">layout</item>
                <item name="text" xsi:type="string">text</item>
                <item name="media" xsi:type="string">media</item>
                <item name="form" xsi:type="string">form</item>
                <item name="product_list" xsi:type="string">product_list</item>
                <item name="product_attribute" xsi:type="string">product_attribute</item>
                <item name="general" xsi:type="string">general</item>
            </item>
        </item>
        <item name="price_after" xsi:type="array">
            <item name="label" xsi:type="string">Below Price</item>
            <!-- allowed_groups as gallery_after -->
        </item>
        <item name="product.info.additional" xsi:type="array">
            <item name="label" xsi:type="string">Product Options</item>
            <item name="allowed_groups" xsi:type="array">
                <item name="product_option" xsi:type="string">product_option</item>
            </item>
            <item name="max_per_component" xsi:type="number">1</item>
        </item>
        <item name="info_after" xsi:type="array">
            <item name="label" xsi:type="string">Below Add to Cart</item>
            <!-- allowed_groups as gallery_after -->
        </item>
    </item>
    <item name="settings" xsi:type="array"/>
</item>
```

Copy the full `allowed_groups` lists from `default_product_detail_simple` in
`module-lynx-catalog/etc/di.xml` rather than writing them from memory.

### Component changes

Copy `Components/Product/Simple.vue` and `Simple.tsx` from
`module-lynx-catalog/view/frontend/lynx/<vue|react>/src/` to
`Vendor/GiftCardLynx/view/frontend/lynx/<vue|react>/src/Components/Product/GiftCard.*`. Keep gallery, price, tier
prices, rating, wish list, analytics and the four slots. Change only the quantity section and `addToCart()`:


```vue
<script setup lang="ts">
// Imports, the props of Simple.vue and its gallery, wish list and analytics code stay as they are

const props = withDefaults(
    defineProps<{
        // …every prop of Simple.vue, plus:
        amounts?: { value: number }[]
    }>(),
    { /* …defaults of Simple.vue, plus: */ amounts: () => [] }
)

const amount = ref<number | null>(null)
const recipient = ref({ name: '', email: '' })

async function addToCart() {
    if (adding.value) return

    const { name, email } = recipient.value
    if (amount.value === null || !name.trim() || !email.trim() || chosenOptions.value.missing.length) {
        messagesStore.add('error', props.labels.requiredOptions ?? '')
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
// Imports, the props of Simple.tsx and its gallery, wish list and analytics code stay as they are
type Props = {
    // …every prop of Simple.tsx, plus:
    amounts?: { value: number }[]
}

export default function GiftCard({ amounts = EMPTY_LIST, /* …as Simple.tsx */ }: Props) {
    const [amount, setAmount] = useState<number | null>(null)
    const [recipient, setRecipient] = useState({ name: '', email: '' })

    async function addToCart() {
        if (adding) return

        if (amount === null || !recipient.name.trim() || !recipient.email.trim() || chosenOptions.missing.length) {
            messagesStore.add('error', labels.requiredOptions ?? '')
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

`Setup/Patch/Data/InstallGiftCardLayout.php` — copies the simple product's default layout (Product Detail with
Product Custom Options, Tabs group, Related Products) and adds the options node:

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

        // First node: Product Detail; its Product Options slot already holds Product Custom Options
        $nodes = $this->json->unserialize($value);
        $nodes[0]['slots']['product.info.additional'][] = [
            'name'      => 'vendor_giftcard_options',
            'component' => 'Vendor_GiftCardLynx/Components/GiftCardOptions',
            'label'     => 'Gift Card Options',
            'props'     => [],
            'visible'   => true,
        ];

        $connection->insertOnDuplicate($table, [[
            'design_id' => Design::DEFAULT_ID,
            'path'      => 'catalog_product_view_type_vendor_giftcard',
            'value'     => $this->json->serialize($nodes),
        ]], ['value']);

        return $this;
    }
}
```

For option c, instead of appending the options node set `$nodes[0]['name'] = 'vendor_product_detail_giftcard'` and
`$nodes[0]['component'] = 'Vendor_GiftCardLynx/Components/Product/GiftCard'`.

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
