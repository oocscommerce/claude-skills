---
name: extend-component
description: Adds markup inside a built-in Lynx component and runs code at its actions without overriding its file - a block in a container (`@block` in an `extend type` of the owning component, a Vue/React file receiving `context` and `parent`, data from an `afterResolve` plugin on the owner's provider) and an observer on an event (`@observer`, a `.ts` file under `src/observers/` that changes the data or throws to stop). Also lets a module's own component offer containers and events (`@container`, `@event`, `<Container>`, `dispatch()`). Use when a developer says "add something under each cart item on Lynx", "badge on Lynx product cards", "show my data on the Lynx order page", "run code before add to cart on Lynx", "validate cart update", "hook into Lynx place order", "port a RequireJS mixin of add-to-cart", or "block/observer not running after theme override". Use with the basics skill; the provider skill covers the plugin that adds the data.
---

# Lynx containers, blocks, events and observers

Puts your module's markup inside a Lynx component (a block in one of its containers) and your code at one of its
actions (an observer on one of its events), without copying the component. Both are merged into the owning component's
JavaScript when the theme is built, and run on every page that uses the component; nothing shows in the editor.

## Before you start

- Read the basics skill (module skeleton, `<sequence>`, styling, CLI).
- Pick the container or event from the tables below; read the owner's `.graphqls` (path in the table) for its exact
  type name and the `context` / data types, or `Lynx.Containers` / `Lynx.Events` in the theme's
  `lynx/dist/props.d.ts`.
- Use a slot instead when the merchant should place your component (editor), and a container when the markup repeats
  per list item or must be there without the merchant.
- Example in Lynx itself: `OOCS_LynxGiftMessage` (`etc/components/GiftMessage.graphqls`, `Plugin/*`, four blocks).

| Container | Owner type (`.graphqls`, module) | Where | `context` |
|---|---|---|---|
| `checkout.cart.item.actions` | `CheckoutCart` (`CheckoutCart.graphqls`, `OOCS_LynxCheckout`) | under each cart line | `CartItem` |
| `checkout.cart.order.actions` | `CheckoutCart` | under the cart lines | none |
| `sales.order.item.after` | `AccountOrderView` (`OrderView.graphqls`, `OOCS_LynxCustomer`) | under each item name, Items Ordered tab | `OrderDetailItems` |
| `sales.order.items.after` | `AccountOrderView` | under the items table, Items Ordered tab | none |
| `category.product.labels` | `ProductListing` (`ProductListing.graphqls`, `OOCS_LynxCatalog`) | over each card image | `ProductListingProduct` |
| `catalog.product.list.labels` | `ProductCarousel` (`ProductCarousel.graphqls`, `OOCS_LynxCatalog`) | over each card image (also Related, Up-sell, Cross-sell) | `ProductCarouselProduct` |

| Event | Declared on | Fired by | Data |
|---|---|---|---|
| `checkout_cart_product_add_before` / `checkout_cart_product_add_after` | `ProductDetailSimple` (`Simple.graphqls`) | Add to Cart of every Product Detail type, `ProductListing`, `ProductCarousel`, `ProductCompare` | `CartProductAdd` (`product_id`, `qty`, `params`: the `URLSearchParams`/`FormData` to post) / `CartProductAdded` |
| `checkout_cart_update_items_before` | `CheckoutCart` | Update Cart | `CartUpdateItems` (`cart`: item id → `{ qty }`) |
| `checkout_cart_coupon_apply_before` | `CheckoutCart` | Apply Discount | `CartCouponApply` (`code`) |
| `checkout_submit_before` / `checkout_submit_all_after` | `CheckoutStepPayment` (`Payment.graphqls`, `OOCS_LynxCheckout`) | Place Order through the Payment step (every renderer) and Braintree PayPal express; not Stripe wallets nor PayPal Express shortcut/review | `CheckoutSubmit` (`method`, `payload` of `payment-information`) / `CheckoutSubmitted` (`order_id`) |
| `customer_register_before` | `CreateAccountForm` (`OOCS_LynxCustomer`) | Create an Account (native form submit) | `CustomerRegister` (`form`: the `<form>`) |
| `customer_account_edit_before` | `AccountInfo` | Save on Account Information | `CustomerAccountEdit` |
| `customer_address_save_before` | `AccountAddressForm` (`AddressForm.graphqls`) | Save Address in the account | `CustomerAddressSave` |

## Steps

### 1. Module dependency - `etc/module.xml`

Add the owner's module (`OOCS_LynxCheckout`, `OOCS_LynxCustomer`, `OOCS_LynxCatalog`) to `<sequence>`, and require
its package in `composer.json`.

### 2. Declare - `etc/components/DeliveryNote.graphqls`

On the owner type, next to the field your block reads:

```graphql
extend type CheckoutCart
  @block(container: "checkout.cart.item.actions", name: "vendor_delivery_note",
         template: "Vendor_Module::Components/ItemDeliveryNote")
  @observer(event: "checkout_cart_update_items_before", name: "vendor_qty_cap",
            instance: "Vendor_Module::observers/capQty") {
  "Note per cart item id"
  vendor_delivery_notes: StringMap @optional
}
```

- `name` is unique per container/event; another module declaring the same `name` replaces yours, `remove: true`
  drops it. `before` / `after`: another block/observer name, or `"-"` for first/last; default is module sequence order.
- `template` → `view/frontend/lynx/<vue|react>/src/Components/<Name>.vue|.tsx`; `instance` →
  `view/frontend/lynx/<vue|react>/src/observers/<name>.ts`, one per framework.
- `extend type X @observer(...)` without fields is valid.

### 3. Data - plugin on the owner's provider

Owner providers: `CheckoutCart` → `OOCS\LynxThemeComponent\Provider\Checkout\Cart`; `AccountOrderView` →
`OOCS\LynxCustomer\Provider\Customer\OrderView` (`GuestOrderView` extends it); listing/carousel → read the
`provider:` of the owner's `@component`. Add the key only when there is data (props rule: add, never build then
`unset`); also `afterResolveSample` when the owner has sample data in the editor:

```php
public function afterResolve(Cart $subject, array $result): array
{
    $notes = $this->notes->forQuote((int) $this->checkoutSession->getQuoteId());
    if ($notes) {
        $result['vendor_delivery_notes'] = $notes;
    }

    return $result;
}
```

Register it in `etc/frontend/di.xml` (`<type name="OOCS\LynxThemeComponent\Provider\Checkout\Cart"><plugin …/>`).

### 4. Block - `view/frontend/lynx/vue/src/Components/ItemDeliveryNote.vue`

Props are always `context` (the loop item, typed per container) and `parent` (the owner's props):

```vue
<script setup lang="ts">
import { computed } from 'vue'

const props = defineProps<{ context: Lynx.CartItem; parent: Lynx.CheckoutCart }>()

const note = computed(() => props.parent.vendor_delivery_notes?.[props.context.item_id])
</script>

<template>
    <p v-if="note" class="col-span-full text-sm text-base-content/70">{{ note }}</p>
</template>
```

React - `view/frontend/lynx/react/src/Components/ItemDeliveryNote.tsx`:

```tsx
type Props = { context: Lynx.CartItem; parent: Lynx.CheckoutCart }

export default function ItemDeliveryNote({ context, parent }: Props) {
    const note = parent.vendor_delivery_notes?.[context.item_id]

    return note ? <p className="col-span-full text-sm text-base-content/70">{note}</p> : null
}
```

Cart line containers sit in the line's grid: start the root with `col-span-full` to span it.

### 5. Observer - `view/frontend/lynx/vue/src/observers/capQty.ts` (same file under `react/`)

```ts
// Caps every line at 10 before the cart is updated
export default (data: Lynx.Events['checkout_cart_update_items_before']) => {
    for (const line of Object.values(data.cart as Record<string, { qty: number }>)) {
        line.qty = Math.min(line.qty, 10)
    }
}
```

Change `data` in place to alter what is sent; throw to stop the action (the owner shows `error.message` as an error
and sends nothing). May be `async`; observers run in turn on the same object. An observer only gets the event data:
a message it throws must come from that data or from your own server call (translated in PHP), never a hardcoded
string.

### 6. Your own component offering containers and events

```graphql
type VendorDeals @component(code: "vendor_deals", template: "Vendor_Module::Components/Deals", label: "Deals")
  @container(name: "vendor.deals.item.after", context: "VendorDealsItem")
  @event(name: "vendor_deals_claim_before", data: "VendorDealsClaim") {
  items: [VendorDealsItem!]!
}

input VendorDealsClaim {
  deal_id: Int!
}
```

```vue
<script setup lang="ts">
import Container from 'OOCS_LynxTheme/Components/Container.vue'
import { dispatch } from 'OOCS_LynxTheme/utils/dispatch'
const props = defineProps<Lynx.VendorDeals>()
</script>
<!-- in the loop -->
<Container name="vendor.deals.item.after" :context="item" :parent="props" />
```

React: `import Container from 'OOCS_LynxTheme/Components/Container'`,
`<Container name="vendor.deals.item.after" context={item} parent={props} />`; `dispatch` is the same. Call
`await dispatch('vendor_deals_claim_before', data)` in `try`/`catch` and show the error message.

## Rules

- **Build-time merge.** `<Container>` and `dispatch()` are rewritten in the owner's source before compiling: blocks are
  imported statically into the owner's chunk (one copy per owner; module-level state of a block is not shared between
  owners), `dispatch` becomes a call of the event's observers. The stub files only serve type checking.
- **Written exactly**, or the build fails: `<Container … />` self-closing, `name` a plain string (`name="…"`, React
  also `name={'…'}`), no attribute value containing `>`; `dispatch('…', data)` with a plain string name; imports
  exactly `import Container from 'OOCS_LynxTheme/Components/Container(.vue)'` and
  `import { dispatch } from 'OOCS_LynxTheme/utils/dispatch'`, each on its own line; Vue files with `<script setup>`.
- **Errors** (naming the file): a container or event declared twice, `@block` into an unknown container, `@observer` on
  an unknown event, a block without `template`, an observer without `instance`.
- **Theme overrides of an owner must keep its `<Container>` tags and `dispatch()` calls.** If the winning file lacks
  one, the build warns `container <name> is declared on <owner>, but <file> does not use it` (or `event …`) and those
  blocks/observers silently do nothing.
- **Names are global** and follow Magento's: containers like layout XML (`a.b.c`), events like `events.xml`
  (`x_y_before` / `x_y_after`).
- **Not in the editor**: blocks are not nodes; they have no settings, no provider and no `availableOn`. Their data
  comes only through the owner's props (declared with `extend type`, or the server drops it).
- **No hardcoded UI text, theme tokens only, both frameworks** - as for any component (basics, component skills).

## Check

1. `bin/magento setup:upgrade` (new module), `bin/magento lynx:workspace:setup <themes>` when the module first gets a
   `view/frontend/lynx` folder.
2. After a `.graphqls` change or enabling/disabling the module: `bin/magento cache:clean lynx_component`, then
   `bin/magento lynx:static:deploy`. The build must print no `is declared on … does not use it` warning; the
   theme's `lynx/dist/extensions.json` lists your block/observer.
3. Storefront on each Lynx theme (Vue and React): the block renders in every loop item, hydrates without console
   warnings, and is gone when your data key is absent; the observer changes the request or stops it with its message.
4. Tests: render the block directly with `context` and `parent`; in the owner's tests containers are empty and
   `dispatch` does nothing.
5. Disable the module, clean `lynx_component`, rebuild: the owner renders as before.

## Reference

- Skills: **basics**, **provider** (plugins on providers), **component** (`.graphqls`, `extend type`), **theme**
  (overrides), **testing**.
- https://docs.oocscommerce.com/lynx/developer/extend-components
