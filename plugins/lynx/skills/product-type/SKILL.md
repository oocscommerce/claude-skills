---
name: product-type
description: Adds Lynx storefront support for a new or third-party Magento product type (gift card, subscription, rental, any custom AbstractType). Covers the per-type product page path catalog_product_view_type_<typeId>, the editor page in PreviewPagebuilder with a sample product of that type, the Product frame and Product Media (available_on), the Product Detail component (reuse Simple, add a Product Options block through productStore, or a custom provider extending AbstractDetail), cards in listings and carousels (needs_options from Helper\BuyFromList and isPossibleBuyFromList), the AJAX add-to-cart JSON (success, messages, backUrl), the default layout data patch, and cart / mini cart line options. Use when a developer says "product page is empty for my product type", "support gift card type on Lynx", "custom product type Lynx", "Product Detail for <type>", "add to cart for my type on Lynx". Use with the basics skill.
---

# Lynx Product Type

Makes a product type that Lynx does not know render and sell on a Lynx store: an editor page for its layout, a
Product Detail that posts the right `checkout/cart/add` parameters, correct cards in product lists, and a starting
layout. Output is a compatibility module (here `Vendor_GiftCardLynx` for a type `vendor_giftcard`).

## Before you start

- Read `basics` (module layout, registry entry, provider contract, build commands).
- Find the type code exactly as `$product->getTypeId()` returns it, the type class, and the buy request keys its
  `prepareForCartAdvanced()` / `_prepareProduct()` reads (`giftcard_amount`, …).
- Open the Lynx sources to imitate (under `vendor/oocscommerce/module-lynx-catalog/`):
  - `Plugin/ProductTypeLayoutPath.php`, `Helper/BuyFromList.php`, `Plugin/CartAddJson.php`
  - `Provider/Product/AbstractDetail.php`, `Provider/Product/Detail/Simple.php`, `Detail/Downloadable.php`,
    `Provider/Product/Page.php` (Product: breadcrumbs), `Provider/Product/Media.php` (Product Media: gallery)
  - `etc/di.xml`: entries `default_product` (Product, slots `media` / `detail`), `default_product_media` (Product
    Media, slot `gallery_after`), `default_product_detail_simple` (slots, settings), `pages` of `PreviewPagebuilder`
  - `view/frontend/lynx/{vue,react}/src/Components/Product/Simple.*`, `Page.*`, `Media.*`, `stores/productStore.ts`
  - `Setup/Patch/Data/InstallCatalogLayouts.php` (`productPage()`), `Setup/Patch/Data/SplitProductMedia.php`

### What happens to an unknown type today

`OOCS\LynxCatalog\Plugin\ProductTypeLayoutPath` (before plugin on
`OOCS\LynxPageBuilder\Api\LayoutRepositoryInterface::getLayout()`, `etc/frontend/di.xml`, sortOrder 10) rewrites
`catalog_product_view` to `catalog_product_view_type_<typeId>` whenever the registry holds `current_product`. For a
new type no design has that path, so the content area is **empty** (header/footer, title, meta, FPC tags still work).
There is no fallback to the simple layout, the editor has no page for it, and the built-in Product (`default_product`),
Product Media (`default_product_media`) and Product Detail entries are limited by `available_on` to exact type paths.

A product page is a **Product** frame (breadcrumbs; slot `media` "Images", slot `detail` "Details") holding
**Product Media** (gallery) and a **Product Detail** (title, price, add to cart). Your type needs all three.

Works unchanged: cart page and mini cart options (Magento's `ConfigurationPool` / customer-data `ItemPool`),
wish list (`canConfigure()` / `hasRequiredOptions()`), FPC tags `cat_p` / `cat_p_<id>` (`Block\ProductCacheTags`).

## Steps

### 1. Module skeleton

`etc/module.xml` sequence: the type's module, `OOCS_LynxComponent`, `OOCS_LynxPageBuilder`, `OOCS_LynxCatalog`
(must load after LynxCatalog so `available_on` merges into its entries). `composer.json` requires
`oocscommerce/module-lynx-component`, `oocscommerce/module-lynx-page-builder`, `oocscommerce/module-lynx-catalog`.

### 2. Declare the editor page (global `etc/di.xml`)

```xml
<type name="OOCS\LynxPageBuilder\Model\PreviewPagebuilder">
    <arguments>
        <argument name="pages" xsi:type="array">
            <item name="product_vendor_giftcard" xsi:type="array">
                <item name="sort_order" xsi:type="number">20</item>
                <item name="label" xsi:type="string">Gift card product page</item>
                <item name="path" xsi:type="string">catalog_product_view_type_vendor_giftcard</item>
                <item name="route" xsi:type="string">catalog/product/view</item>
                <item name="sample" xsi:type="string">product</item>
                <item name="type" xsi:type="string">vendor_giftcard</item>
                <item name="group" xsi:type="string">Products</item>
            </item>
        </argument>
    </arguments>
</type>
```

`path` = `catalog_product_view_type_` + type code. `sample`=`product` with `type`: `Model\PreviewRoute` opens the
first enabled, visible product of that type in the previewed store view; none means the not-found page. For other
page keys see `page`.

### 3. Pick the Product Detail — lightest option that fits

Whatever the option, add your path to the `available_on` of Product and Product Media (Magento merges the items
into Lynx's entries):

```xml
<type name="OOCS\LynxComponent\Model\Registry">
    <arguments>
        <argument name="components" xsi:type="array">
            <item name="default_product" xsi:type="array">
                <item name="available_on" xsi:type="array">
                    <item name="vendor_giftcard" xsi:type="string">catalog_product_view_type_vendor_giftcard</item>
                </item>
            </item>
            <item name="default_product_media" xsi:type="array">
                <item name="available_on" xsi:type="array">
                    <item name="vendor_giftcard" xsi:type="string">catalog_product_view_type_vendor_giftcard</item>
                </item>
            </item>
        </argument>
    </arguments>
</type>
```

**a. Bought like a simple product** (`product` + `qty` + custom options is enough). Add the same `available_on` item
to `default_product_detail_simple` as well.

**b. A few extra fields.** Keep option a and add a component in group `product_option` (the only group the
**Product Options** slot `product.info.additional` accepts). It contributes to the request through
`useProductStore` from `OOCS_LynxCatalog/stores/productStore`:

| Contribution key | Meaning |
|---|---|
| `fields` | Extra `checkout/cart/add` params; an array value is posted once per entry |
| `files` | Uploads by field name; any file switches the request to multipart |
| `price` | Added to the shown price and analytics value |
| `missing` | Titles of empty required fields; non-empty blocks the post and bumps `validationTicks[product_id]` (the block owning the field points at it) |
| `base` | Optional `{ price, original_price, is_on_sale }` replacing the product's own prices (the picked configurable child) |
| `images` | Optional `GalleryImage[]` (exported by the store) that Product Media shows before the product's own |

Store API (Vue pinia and React hook have the same names): `contribute(productId, key, contribution)`,
`forget(productId, key)`, `requestValidation(productId)`, `validationTicks`, `collect(productId)` (merged). Contribute on every change
(`immediate`), `forget` on unmount and when `product_id` changes (Inertia can keep the block mounted), focus the first
empty field when the tick for the product changes.

Registry entry (global `etc/di.xml`, inside the `components` argument):

```xml
<item name="vendor_giftcard_options" xsi:type="array">
    <item name="component" xsi:type="string">Vendor_GiftCardLynx::Components/GiftCardOptions</item>
    <item name="provider" xsi:type="object">Vendor\GiftCardLynx\Provider\GiftCardOptions\Proxy</item>
    <item name="label" xsi:type="string">Gift Card Options</item>
    <item name="description" xsi:type="string">Amount, recipient and message of a gift card, posted with Add to Cart</item>
    <item name="icon" xsi:type="string">ph-gift</item>
    <item name="group" xsi:type="string">product_option</item>
    <item name="zones" xsi:type="array"><item name="content" xsi:type="string">content</item></item>
    <item name="available_on" xsi:type="array">
        <item name="vendor_giftcard" xsi:type="string">catalog_product_view_type_vendor_giftcard</item>
    </item>
    <item name="settings" xsi:type="array"/>
</item>
```

Provider extends `OOCS\LynxComponent\Model\AbstractComponentProvider`, reads `current_product` from
`Magento\Framework\Registry`, returns `$props` unchanged when there is none or the type differs, and sends
`product_id` (keys the contribution), the type's data and translated `labels`. Full provider, `.vue` and `.tsx`:
[reference/giftcard-example.md](reference/giftcard-example.md#option-b).

**c. Its own Product Detail** (no qty stepper, price range, preview…). Provider extends
`OOCS\LynxCatalog\Provider\Product\AbstractDetail` and overrides only `addTypeProps()`:

```php
namespace Vendor\GiftCardLynx\Provider\Product\Detail;

use Magento\Catalog\Model\Product;
use OOCS\LynxCatalog\Provider\Product\AbstractDetail;

class GiftCard extends AbstractDetail
{
    /**
     * Amounts and the labels of the recipient fields
     *
     * @param Product $product
     * @param array $props
     * @return array
     */
    protected function addTypeProps(Product $product, array $props): array
    {
        // Sold one at a time at a picked amount: no addQuantityProps(), no addRegularPrice()
        $type = $product->getTypeInstance();
        $props['amounts'] = array_map(static fn (float $a): array => ['value' => $a], $type->getAmounts($product));
        $props['labels'] += [
            'amount'         => (string) __('Amount'),
            'recipientName'  => (string) __('Recipient Name'),
            'recipientEmail' => (string) __('Recipient Email'),
            'requiredField'  => (string) __('This is a required field.'),
        ];

        return $props;
    }
}
```

`AbstractDetail::resolve()` already handles no `current_product`, tags the page (`cat_p`, `cat_p_<id>`) and always
sends `product_id`, `title`, `sku`, `price`, `is_salable`, `rating`, `urls.add_to_cart`, `sections.add_to_cart` and
the base `labels`. Only when the node's setting is on and Magento allows it: `urls.add_to_wishlist` (`show_wishlist`
`yes` and the Wish List enabled) and `urls.add_to_compare` (`show_compare` `yes` and the product comparable), each
with its section and labels. Helpers a type calls from `addTypeProps()` when it shows them: `addQuantityProps()`
(`tier_prices` and the quantity labels) and `addRegularPrice()` (`original_price`, `is_on_sale`). It sends no
`type_id`, images or breadcrumbs: those are Product Media's and Product's.
Register it as `vendor_product_detail_giftcard` in group `product_page`, `available_on` your path, copy the three
slots (`price_after`, `product.info.additional` with `max_per_component` 1, `info_after`) with their full
`allowed_groups`, and the `show_wishlist` / `show_compare` settings, from `default_product_detail_simple`. Component:
copy `Components/Product/Simple.vue` and `Simple.tsx` to
`view/frontend/lynx/<vue|react>/src/Components/Product/GiftCard.*`, replace the quantity section with the type's
fields and add the type's params to the `URLSearchParams` body. Details:
[reference/giftcard-example.md](reference/giftcard-example.md#option-c).

### 4. Follow the add-to-cart response contract

Post to `urls.add_to_cart` with `X-Requested-With: XMLHttpRequest`, `credentials: 'same-origin'`, `form_key` from
`useFormKeyStore().ensure()`, and all `collect()`ed option fields/files. On Lynx stores
`OOCS\LynxCatalog\Plugin\CartAddJson` (after plugin on `Magento\Checkout\Controller\Cart\Add::execute`) answers:

| Key | Meaning | Component does |
|---|---|---|
| `success` | `true` only when `checkout_cart_add_product_complete` fired | Track `add_to_cart`, `customerDataStore.reload(sections.add_to_cart)` |
| `messages` | `[{type, text}]` taken out of the session (shown once) | `messagesStore.addServerMessage(type, text)` for each |
| `backUrl` | Magento's: cart (redirect-to-cart setting) or product page (options missing) | `router.visit(backUrl)` |

Never show a success message of your own; validate required fields before posting
(`optionsStore.requestValidation(product_id)` when `collect(product_id).missing` is not empty).

### 5. Cards in product lists

Listing, Search Results, Advanced Search, Product Carousel, Related, Up-sell, Cross-sell send each product's
`needs_options` from `OOCS\LynxCatalog\Helper\BuyFromList::needsOptions()`: `true` for `configurable` and `grouped`,
otherwise `!$product->getTypeInstance()->isPossibleBuyFromList($product)` (core: false when the product has required
options). With the flag the card opens the product page; without it, the card posts `product` + `qty=1`.

If the type needs input before it can go in the cart, make the type class say so:

```php
public function hasRequiredOptions($product)
{
    return true;
}
```

For a type owned by another vendor, do it from the compatibility module with a plugin on the type class
(`afterHasRequiredOptions` or `afterIsPossibleBuyFromList`). No theme override needed. Salable state needs nothing.

### 6. Ship a starting layout (optional, recommended)

A data patch depending on `OOCS\LynxCatalog\Setup\Patch\Data\SplitProductMedia` (which runs after
`InstallCatalogLayouts`, so the simple layout already has the Product frame) copies the default design's
`catalog_product_view_type_simple` row of `lynx_pagebuilder_layout` (`design_id` =
`OOCS\LynxPageBuilder\Model\Design::DEFAULT_ID`) to your path. `$nodes[0]` is a Group whose
`['slots']['default'][0]` is Product; the Product Detail is `$nodes[0]['slots']['default'][0]['slots']['detail'][0]`
(Tabs and Related Products follow). Add your options node to that detail's `['slots']['product.info.additional']`
(option b) or rename the detail (option c). Write only to the
default design and only your own path: merchant designs inherit it, and their saved layouts are theirs. Full patch:
[reference/giftcard-example.md](reference/giftcard-example.md#starting-layout).
In layout rows `component` is `Vendor_GiftCardLynx/Components/GiftCardOptions` (slash); in `di.xml` it is `::`.

### 7. Cart page and mini cart

No Lynx code. Register what Luma needs: the type's helper in `Magento\Catalog\Helper\Product\ConfigurationPool`
argument `instancesByType` (cart page options, read by `OOCS\LynxThemeComponent\Provider\Checkout\Cart`) and, if
the default item is not enough, an item class in `Magento\Checkout\CustomerData\ItemPool` argument `itemMap`
(mini cart, `cart` section `options`). Both live in the type's own module (frontend `di.xml`).

### 8. Components around the Product Detail

Built-ins using `catalog_product_view*` (Description, Specifications, Reviews, Custom Options, Product Attribute,
Product Image Attribute, Related, Up-sell) appear on the new page automatically. Product, Product Media and every
Product Detail list exact paths: add yours to Product and Product Media (step 3). Components from other modules that
list exact paths (`catalog_product_view_type_simple`) need your path added to their `available_on` the same way.

## Rules

- `available_on` of Product, Product Media and a Product Detail lists exact type paths (never
  `catalog_product_view*`); other product-page components use `catalog_product_view*`. A missing path makes the
  entry invisible in the editor, so the page cannot hold a gallery or a frame.
- Send a prop, label or URL only when the component will show it (setting on, config allows), as `AbstractDetail`
  does for wish list and compare; the component renders those parts only when the URL is present.
- Extend `AbstractDetail`, not `Detail\Simple`: plugins declared on `Simple` would also run for your type.
  A subclass needing another dependency repeats the nine parent arguments and calls `parent::__construct()`
  (see `Detail/Downloadable.php`).
- To change one existing type, plug into its provider (`afterResolve` on `Provider\Product\Detail\Configurable`,
  return early when `!isset($result['product_id'])`); a plugin on `AbstractDetail` hits every type.
- Prices for display come from `priceInfo` / `PriceCurrencyInterface::convert()`; buy request values stay in the
  type's own unit.
- Providers never read the customer session: product pages are in the full page cache.
- Labels translated in PHP (`__()`), Magento's own phrases reused (`Add to Cart`, `Adding...`, `Add to Wish List` are
  already in `AbstractDetail` labels); new phrases go to `i18n/en_US.csv`.
- Styling only with tokens and `l-` daisyUI classes (`l-btn`, `l-input`, `l-textarea`); no size below `text-sm`.
- No `window`/`document` during setup/render (SSR). One root element.

## Check

- `bin/magento setup:upgrade`, `bin/magento lynx:workspace:setup <themes>` (new module), `bin/magento
  lynx:static:deploy`, `bin/magento cache:clean` (di.xml, `lynx_component` layout cache, full page cache).
- A real product of the type renders its layout on a Lynx store view, in both Vue and React themes.
- Editor lists **Products › Gift card product page**; preview opens a product of that type; your entries are offered
  there and only there.
- Add to Cart filled: product added with options; cart page and mini cart show them, mini cart count updates without
  reload. Empty required field: message, no POST, focus moves. Out of stock / qty refusal: Magento's message.
- Cards of the type in Listing, Carousel, Related open the page or add correctly.
- Admin save of the product refreshes its cached page; SSR on renders without hydration warnings.

## Reference

- [reference/giftcard-example.md](reference/giftcard-example.md) — full provider/component/di/data patch code
- https://docs.oocscommerce.com/lynx/developer/product-types
- Related skills: `component`, `provider`, `page`, `theme` (override a built-in type's component)
