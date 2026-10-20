# Example: a delivery date under a carrier's rate

`Vendor_ShippingFoo` has a carrier `vendorfoo` and asks for a delivery date under its rate. The date is stored in a
quote column `vendor_foo_delivery_date` and copied to the order.

```
app/code/Vendor/ShippingFoo/
├── etc/
│   ├── module.xml               sequence OOCS_LynxCheckout
│   ├── db_schema.xml            quote + sales_order column
│   ├── fieldset.xml             sales_convert_quote -> to_order
│   └── frontend/di.xml          renderer pool entry + beforeSave plugin (SKILL.md step 2)
├── i18n/en_US.csv
├── Model/Config/DeliveryDate.php   (SKILL.md step 3)
├── Plugin/SaveDeliveryDate.php     (SKILL.md step 5)
└── view/frontend/lynx/
    ├── vue/src/Components/DeliveryDate.vue    (SKILL.md step 4)
    └── react/src/Components/DeliveryDate.tsx  (below)
```

## Storage (plain Magento)

`etc/db_schema.xml`:

```xml
<?xml version="1.0"?>
<schema xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Setup/Declaration/Schema/etc/schema.xsd">
    <table name="quote">
        <column xsi:type="date" name="vendor_foo_delivery_date" nullable="true" comment="Foo Delivery Date"/>
    </table>
    <table name="sales_order">
        <column xsi:type="date" name="vendor_foo_delivery_date" nullable="true" comment="Foo Delivery Date"/>
    </table>
</schema>
```

Then `bin/magento setup:db-declaration:generate-whitelist --module-name=Vendor_ShippingFoo`.

`etc/fieldset.xml` (Magento copies listed quote fields to the order):

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:DataObject/etc/fieldset.xsd">
    <scope id="global">
        <fieldset id="sales_convert_quote">
            <field name="vendor_foo_delivery_date">
                <aspect name="to_order"/>
            </field>
        </fieldset>
    </scope>
</config>
```

Validate the posted value in the plugin before writing it, for example:

```php
$parsed = \DateTimeImmutable::createFromFormat('!Y-m-d', $date);
if ($parsed === false || $parsed->format('Y-m-d') !== $date || $date <= date('Y-m-d')) {
    throw new \Magento\Framework\Exception\LocalizedException(__('Please enter a valid date.'));
}
```

`i18n/en_US.csv`: add `"Delivery Date","Delivery Date"` (and any phrase that is not already Magento's).

## React renderer - `Components/DeliveryDate.tsx`

```tsx
import { useImperativeHandle, useState, type Ref } from 'react'
import { useCheckoutStore } from 'OOCS_LynxCheckout/stores/checkoutStore'

type Handle = {
    validate: () => boolean
    data: () => { vendorFooDeliveryDate: string }
    summary: string
}

type Props = {
    rate: { carrier_code: string; method_code: string }
    config: { selected?: string; min?: string; labels?: Record<string, string> }
    ref?: Ref<Handle>
}

export default function DeliveryDate({ config, ref }: Props) {
    const checkout = useCheckoutStore()
    const [date, setDate] = useState(config.selected ?? '')
    const [error, setError] = useState('')

    useImperativeHandle(
        ref,
        () => ({
            validate() {
                const message = date ? '' : (config.labels?.required ?? '')
                setError(message)
                return !message
            },
            data: () => ({ vendorFooDeliveryDate: date }),
            summary: date,
        }),
        [date, config.labels?.required]
    )

    return (
        <div>
            <label htmlFor="vendor-foo-date" className="block text-sm text-base-content mb-1.5">
                {config.labels?.deliveryDate} <span className="text-error">*</span>
            </label>
            <input
                id="vendor-foo-date"
                value={date}
                type="date"
                min={config.min}
                className={checkout.inputClass(!!error)}
                onChange={e => {
                    setDate(e.target.value)
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

## How In-Store Pickup uses the same pieces

`OOCS_LynxMSI` (depends on Magento's Inventory In-Store Pickup modules; disable it on a store without MSI):

1. **Pool entry** (`etc/frontend/di.xml`): carrier `instore` -> `OOCS_LynxMSI/Components/StorePickup`,
   config `OOCS\LynxMSI\Model\PickupLocations`. Plugin `OOCS\LynxMSI\Plugin\SavePickupLocation`
   on `OOCS\LynxCheckout\Provider\Checkout\Step\ShippingMethod`, `sortOrder="10"`.
2. **Config** (`PickupLocations::getConfig()`): not the stores themselves, but `search_url` (anonymous core REST
   `GET /V1/inventory/in-store-pickup/pickup-locations/`), `website_code`, `skus` (leaf items only), `page_size`,
   `selected` (the store already on the quote, in the REST item shape) and `labels`.
3. **Renderer**: searches the REST endpoint with the website and the cart SKUs (only stores that can fulfil the whole
   cart), shows the selected store card with "Select Other". `validate()` requires a store, `data()` returns
   `{ pickupLocationCode }`, `summary` is the store name (a `computed` in Vue).
4. **beforeSave**: acts only when `$draft['shippingMethod'] === InStorePickup::DELIVERY_METHOD` and a code is posted;
   returns early when the quote address already has that method **and** that pickup location code; otherwise builds
   the store address with core's `ToQuoteAddress::convert()`, sets `pickup_location_code` on the address extension
   attributes and calls `saveAddressInformation()` with carrier `instore` / method `pickup`. An unknown code throws
   `NoSuchEntityException` (a `LocalizedException`), which comes back as the step's error. The step's own `save()`
   then sees the method already set and does nothing. Core's In-Store Pickup code validates the quote and copies the
   location to the order.
