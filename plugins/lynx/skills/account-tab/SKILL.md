---
name: account-tab
description: Adds a My Account tab to a Lynx store from a third-party Magento module - the sidebar entry in the `tabs` pool of OOCS\LynxCustomer\Model\Customer\AccountTabs, a section component (group `account_section`) that sits in the slot of the My Account frame (`default_account`), the customer-only controller and its `lynx_<handle>.xml` (lynx_no_cacheable, ssr off, encrypted history), the PreviewPagebuilder page with `sample_data`, a provider implementing SampleProviderInterface for the editor preview, and a data patch with the starting layout. Use when a developer says "add a tab to My Account on Lynx", "customer account page for my module on Lynx", "my account link missing in the Lynx sidebar", "port customer_account_navigation link to Lynx", "account section component". Use with the basics skill.
---
# Lynx My Account tab

Produces a new tab in the Lynx customer account: a sidebar link, a page at your route that shows the shared My Account
frame (sidebar + one section) with your section component in its slot, editable in PageBuilder and previewed there with
sample rows instead of a signed-in customer.

## Before you start

- Read **basics**; the section is a normal component (**component**) with a provider (**provider**).
  Page mechanics (routes, `pages` keys, layout patches) are in **page**.
- The Luma `customer_account_navigation` link and the extension's regular layout XML are not read on Lynx. If you are
  porting an existing extension, put the Lynx side in a compatibility module (`Vendor_ProductQuestionsLynx`) that
  sequences the original module and `OOCS_LynxCustomer`.
- Imitate a built-in tab: `vendor/oocscommerce/module-lynx-customer/` - `etc/frontend/di.xml` (tabs),
  `etc/components/MyAccount.graphqls` (`default_account`) and `etc/components/Wishlist.graphqls` (a section), `etc/di.xml`
  (`pages`), `Provider/Customer/Wishlist.php`,
  `view/frontend/lynx/vue/src/Components/Account/*.vue`, `Setup/Patch/Data/InstallCustomerLayouts.php`,
  `view/frontend/layout/lynx_wishlist_index_index.xml`.

Example below: `Vendor_ProductQuestionsLynx`, tab **My Product Questions** at `/productquestions/my/index`
(full action name `productquestions_my_index`, tab key `product_questions`).

## Steps

### 1. Dependencies

`etc/module.xml` `<sequence>`: `OOCS_LynxCustomer`, `OOCS_LynxPageBuilder`, `OOCS_LynxComponent` (and the original
extension if porting). `composer.json`: `oocscommerce/module-lynx-customer`, `oocscommerce/module-lynx-page-builder`,
`oocscommerce/module-lynx-component`.

### 2. Controller (skip if the extension already has one)

Customer-only: implement `Magento\Customer\Controller\AccountInterface` (core's plugin on it redirects guests to sign
in) and return a result page; the content comes from the layout.

```php
<?php
declare(strict_types=1);

namespace Vendor\ProductQuestionsLynx\Controller\My;

use Magento\Customer\Controller\AccountInterface;
use Magento\Framework\App\Action\HttpGetActionInterface;
use Magento\Framework\View\Result\Page;
use Magento\Framework\View\Result\PageFactory;

class Index implements AccountInterface, HttpGetActionInterface
{
    /**
     * @param PageFactory $pageFactory
     */
    public function __construct(
        private readonly PageFactory $pageFactory,
    ) {
    }

    /**
     * Result page; the My Account frame and section come from the layout
     *
     * @return Page
     */
    public function execute(): Page
    {
        return $this->pageFactory->create();
    }
}
```

With `etc/frontend/routes.xml` route `id="productquestions" frontName="productquestions"`.

### 3. Page config `view/frontend/layout/lynx_productquestions_my_index.xml`

```xml
<?xml version="1.0"?>
<page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
    <update handle="lynx_no_cacheable"/>
    <head>
        <title>My Product Questions</title>
        <meta name="robots" content="NOINDEX,NOFOLLOW"/>
    </head>
    <body>
        <referenceBlock name="lynx.inertia">
            <arguments>
                <!-- Personal data: never cached, never server rendered -->
                <argument name="ssr" xsi:type="boolean">false</argument>
                <argument name="encrypt_history" xsi:type="boolean">true</argument>
            </arguments>
        </referenceBlock>
    </body>
</page>
```

### 4. Sidebar entry - `etc/frontend/di.xml`

The `tabs` pool is declared by Lynx in the **frontend** area, so contribute in the same scope:

```xml
<type name="OOCS\LynxCustomer\Model\Customer\AccountTabs">
    <arguments>
        <argument name="tabs" xsi:type="array">
            <item name="product_questions" xsi:type="array">
                <item name="label" xsi:type="string">My Product Questions</item>
                <item name="route" xsi:type="string">productquestions/my</item>
                <item name="sort_order" xsi:type="number">85</item>
            </item>
        </argument>
    </arguments>
</type>
```

- Item name = tab key the frame highlights (`active_tab`). Built-in keys and orders: `dashboard` 10, `orders` 20,
  `addresses` 30, `account_info` 40, `downloads` 50, `newsletter` 60, `wishlist` 70, `reviews` 75,
  `stored_payments` 80. Pick a free value.
- An entry needs both `label` and `route` to show; `route` goes through `UrlInterface::getUrl()`.
- The label is an English phrase that Lynx translates with `__()` when it builds the sidebar. Use the same wording
  as the extension's Luma link and keep the phrase in the module's `i18n/*.csv`.

### 5. Section component and frame availability - `etc/components/AccountQuestions.graphqls`

```graphql
"The questions the signed-in customer asked, with their answers"
type VendorAccountProductQuestions @component(
  code: "vendor_account_product_questions"
  template: "Vendor_ProductQuestionsLynx::Components/AccountQuestions"
  provider: "Vendor\\ProductQuestionsLynx\\Provider\\AccountQuestions"
  label: "My Product Questions"
  group: "account_section"
  icon: "ph-question"
  zones: ["content"]
  availableOn: ["productquestions_my_index"]
) {
  items: [VendorAccountQuestion!]! @optional
  "Every string the section shows, translated by the provider"
  labels: StringMap! @optional
}

type VendorAccountQuestion {
  id: Int!
  product: String!
  url: String!
  question: String!
  answer: String!
  date: String!
}

# The frame may be placed on the new page. @component args replace the old ones: keep Lynx's list and add yours
extend type Account @component(
  availableOn: ["customer_account_index", "customer_account_edit", "customer_address_index", "customer_address_form", "sales_order_history", "sales_order_view", "sales_order_invoice", "sales_order_shipment", "sales_order_creditmemo", "downloadable_customer_products", "newsletter_manage_index", "wishlist_index_index", "vault_cards_listaction", "review_customer_index", "review_customer_view", "productquestions_my_index"]
)
```

`group` `account_section` is the only group the My Account slot accepts. Copy the current `availableOn` from
`OOCS_LynxCustomer` `etc/components/MyAccount.graphqls` (type `Account`) when you write the `extend`. Every key the
provider returns must be declared here; the server drops the rest.

### 6. Editor page - global `etc/di.xml`

```xml
<type name="OOCS\LynxPageBuilder\Model\PreviewPagebuilder">
    <arguments>
        <argument name="pages" xsi:type="array">
            <item name="account_product_questions" xsi:type="array">
                <item name="sort_order" xsi:type="number">70</item>
                <item name="label" xsi:type="string">My product questions page</item>
                <item name="path" xsi:type="string">productquestions_my_index</item>
                <item name="route" xsi:type="string">productquestions/my/index</item>
                <item name="group" xsi:type="string">Account</item>
                <!-- The controller would send the editor to the sign-in page -->
                <item name="sample_data" xsi:type="boolean">true</item>
            </item>
        </argument>
    </arguments>
</type>
```

`route` must be the full three-part `front/controller/action`: the preview router matches `sample_data` pages on it.

### 7. Provider with sample data

`Provider/AccountQuestions.php` extends `OOCS\LynxComponent\Model\AbstractComponentProvider` and implements
`OOCS\LynxComponent\Api\SampleProviderInterface`:

- `resolve(array $props): array` - add translated `labels` first; if `CustomerSession::isLoggedIn()` is false return
  (the controller redirects guests); otherwise add the customer's rows.
- `resolveSample(array $resolved, array $props): array` - called in the editor preview after `resolve()`; overlay
  made-up rows (`url` = `'#'` for every link or endpoint, so nothing real is opened or written).

The page is not cached (`lynx_no_cacheable`), so reading the session in the provider is safe. Full class:
[reference/example.md](reference/example.md#provider).

### 8. Section component (Vue + React)

`view/frontend/lynx/vue/src/Components/AccountQuestions.vue` and `.../react/src/Components/AccountQuestions.tsx`.
Props come typed as `Lynx.VendorAccountProductQuestions` (generated from the `.graphqls`) and items as
`Lynx.VendorAccountQuestion`. It renders inside the frame: root `w-full`, no outer padding/margin, page heading `h1` from `labels.title`, empty
state from `labels.empty`, links with Inertia `Link`. Any action must return early when its URL is `'#'` (sample
preview). Full files: [reference/example.md](reference/example.md#component).

### 9. Starting layout - data patch

Default design (`Design::DEFAULT_ID`), insert only: a Group holding the My Account frame
(`OOCS_LynxCustomer/Components/MyAccount`, node name `default_account`) with `props.active_tab` =
`product_questions`, and your section in the frame's `default` slot. Full patch:
[reference/example.md](reference/example.md#starting-layout). Patch mechanics (insert-only, later changes, cache):
**page** `reference/layout-patch.md`.

## Rules

- `tabs` in `etc/frontend/di.xml` (Lynx declares the pool there); `pages` in global `etc/di.xml`. Declaring a pool in
  another scope than its other contributors replaces their items in that area. Components in `etc/components/*.graphqls`.
- The section's `group` must be `account_section`; the frame's slot accepts only that group, one section per frame.
- Add the page path to `default_account`'s `availableOn` (`extend type Account`, full list), or merchants cannot place
  the frame on your page.
- `active_tab` is a node prop in the layout, not a setting; pages without their own tab (a detail page under a tab) set
  the parent tab's key.
- Personal data page: `lynx_no_cacheable`, `ssr=false`, `encrypt_history=true`, NOINDEX. Never put customer data in
  shared props.
- Sample rows never touch real data: endpoints `#`, ids that cannot match real rows (negative ids), and the component
  skips posting to `#`.
- Labels translated in the provider (`__()`), reusing Magento's or the extension's phrases; no hardcoded UI text in
  `.vue` / `.tsx`. Design tokens only, sizes from the theme scale, never below `text-sm`.

## Check

1. `bin/magento setup:upgrade`; `bin/magento cache:clean config layout full_page lynx_component` (plus
   `bin/magento lynx:compile components` when the site runs compiled).
2. `bin/magento lynx:workspace:setup <themes>` (new module), then `bin/magento lynx:static:deploy` (writes the
   `Lynx.*` types, then builds).
3. Signed out, open `/productquestions/my/index`: redirected to sign in.
4. Signed in: sidebar shows **My Product Questions** in position, highlighted on the page; list or empty state shows;
   other tabs still work; mobile sidebar shows it too.
5. Admin > PageBuilder: page switcher **Account › My product questions page (Sample)** shows the frame with sample
   rows; the section appears in the component picker only for this page.
6. Both Vue and React themes if the store builds both.

## Reference

- [reference/example.md](reference/example.md) - full provider, Vue and React section components, data patch
- https://docs.oocscommerce.com/lynx/developer/account-tabs
