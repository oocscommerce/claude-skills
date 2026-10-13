---
name: forms-analytics
description: Wires forms and analytics in a Magento module's Lynx storefront components - posting to Magento controllers with Inertia router.post/useForm (forceFormData + form_key from the form key store) or a native form post or fetch, protecting a form with Google reCAPTCHA via data-recaptcha keys, showing server flash messages through the messages store, refreshing customer-data sections after an action (sections.xml + OOCS\LynxComponent\Helper\SectionInvalidation + customerDataStore.reload), and analytics through the analyticsStore bus (track, trackOnce, the GA4-style events Lynx emits, adding a tracker component for a third-party analytics or pixel service in the Backstage zone). Use when a developer says "form on Lynx", "form_key invalid on Lynx", "Inertia post to a Magento controller", "add reCAPTCHA to my Lynx form", "messages not showing", "minicart not updating after my action", "track an event in Lynx", "add a pixel to Lynx", "analytics tracker for Lynx". Use with the basics skill.
---
# Lynx forms and analytics

Produces the client side of a module's form or action on Lynx (post, form key, reCAPTCHA token, server messages,
customer-data refresh) and its analytics: events tracked on the Lynx bus, or a tracker component that forwards them
to an outside service.

## Before you start

- Read **basics**. The component itself is built with **component**; its URLs, labels and section names come
  from its provider (**provider**).
- Store imports, identical in Vue (pinia) and React (jotai) themes:
  `OOCS_LynxTheme/stores/global/formKeyStore` (`useFormKeyStore`), `.../messagesStore` (`useMessagesStore`),
  `.../customerDataStore` (`useCustomerDataStore`), `.../analyticsStore` (`useAnalyticsStore`);
  `OOCS_LynxTheme/utils/storeConfig` (`config<T>(key, fallback)`, e.g. `config('currencyCode', '')`).
- Imitate: `vendor/oocscommerce/module-lynx-customer/view/frontend/lynx/vue/src/Components/Account/AccountInfo.vue`
  (router.post + reCAPTCHA), `module-lynx-catalog/.../Components/Product/Simple.vue` (fetch + messages + sections +
  `add_to_cart` event), `module-lynx-google-tag/.../Components/GoogleTag.vue` (tracker).

## Steps

### 1. Choose how the form posts

| Way | When | Result handling |
|---|---|---|
| Inertia `router.post` / `useForm` | controller validates, adds a message, returns a redirect | Inertia follows the redirect as a visit; messages in the `mage-messages` cookie show on the next page |
| Native `<form method="post" :action>` | full page reload is fine (registration-style) | browser reload; messages show on load |
| `fetch` to a JSON/AJAX endpoint | stay on the page, partial UI update | read the JSON; call `messagesStore.sync()` or `addServerMessage()`; refresh sections |

Magento controllers read `$_POST` and check `form_key`. Inertia sends JSON by default, so always pass
`forceFormData: true` and include `form_key`:

```vue
<script setup lang="ts">
import { ref } from 'vue'
import { router } from '@inertiajs/vue3'
import { useFormKeyStore } from 'OOCS_LynxTheme/stores/global/formKeyStore'

const props = defineProps<{ urls: { post: string }; labels: Record<string, string> }>()
const formKey = useFormKeyStore()
const question = ref('')
const busy = ref(false)

function submit(event: SubmitEvent) {
    // Sample preview endpoint: nothing to send
    if (props.urls.post === '#' || busy.value) return
    const form = event.currentTarget as HTMLFormElement
    // Read now: the reCAPTCHA widget resets once the submit event is over
    const captcha = (form.elements.namedItem('g-recaptcha-response') as HTMLTextAreaElement | null)?.value
    busy.value = true
    router.post(
        props.urls.post,
        { form_key: formKey.ensure(), question: question.value, ...(captcha ? { 'g-recaptcha-response': captcha } : {}) },
        { forceFormData: true, preserveScroll: true, onFinish: () => (busy.value = false) }
    )
}
</script>

<template>
    <form data-recaptcha="vendor_faq_ask" novalidate @submit.prevent="submit">
        <textarea v-model="question" name="question" class="l-textarea w-full" />
        <div data-recaptcha-slot></div>
        <button class="l-btn l-btn-primary" :disabled="busy">{{ labels.submit }}</button>
    </form>
</template>
```

React version and the POST controller: [reference/forms.md](reference/forms.md). `useForm` works the same with
`forceFormData: true` and `form_key` in its data.

The POST controller is ordinary Magento: implement `HttpPostActionInterface`, `messageManager->add*Message()`, return
a `Redirect`. Validate the form key **in the controller** with `Magento\Framework\Data\Form\FormKey\Validator`: core's
`CsrfValidator` lets XHR requests through unchecked, and Inertia visits and `fetch` with `X-Requested-With` are XHR.

### 2. Form key

`formKey.ensure()` returns the `form_key` cookie value, minting and writing the cookie when it is missing. Call it at
submit time, never cache it in a prop or at setup: sign-in/sign-out replace the cookie. The theme already calls
`ensure()` on boot, after every visit and when the tab becomes visible.

### 3. reCAPTCHA

Add `data-recaptcha="<key>"` to the `<form>`; optionally `<div data-recaptcha-slot></div>` where the widget goes.
Keys are the children of config `recaptcha_frontend/type_for` (Magento's: `customer_login`, `customer_create`,
`contact`, `newsletter`, `product_review`, `sendfriend`, …). Nothing loads until a field gets focus; a submit without
a token is stopped in the capture phase, so your handler only runs with a token.

- Browser-submitted form: the attribute is all you need (`g-recaptcha-response` is posted with the fields).
- Self-submitted (`router.post`, `fetch`): read `g-recaptcha-response` synchronously in the handler (above). Send it as
  the `g-recaptcha-response` param to a controller, or in the `X-ReCaptcha` header to a REST endpoint.
- A new key for your own route, non-form actions (`useRecaptchaStore` `prepare`/`resolve`): [reference/forms.md](reference/forms.md#recaptcha).

### 4. Server messages

`useMessagesStore()`: `add(type, text)` (`success|error|warning|notice`), `addServerMessage(type?, text?)`,
`sync()` (drains the `mage-messages` cookie, returns true if any error), `remove(id)`, `clear()`.

- After Inertia visits the theme calls `sync()` itself.
- After a `fetch` that may have set the cookie (redirect results), call `messagesStore.sync()`.
- JSON results do not write the cookie (core skips `Json`); messages left in the session arrive through the
  customer-data `messages` section, which the store shows when sections reload (step 5).
- Text comes from the server (Magento's messages, provider `labels`); never hardcode UI strings in the component.

### 5. Customer-data sections

Private, per-customer data (cart, customer, wishlist, your own section) lives in customer-data sections, never in page
props (pages are FPC-cached). After an action that changes such data, refresh the sections it invalidates:

1. Declare invalidation the Magento way, `etc/frontend/sections.xml`:

   ```xml
   <?xml version="1.0"?>
   <config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
           xsi:noNamespaceSchemaLocation="urn:magento:module:Magento_Customer:etc/sections.xsd">
       <action name="vendor_faq/ask/post">
           <section name="vendor_faq"/>
       </action>
   </config>
   ```

2. In the provider, resolve the names for the endpoint with `OOCS\LynxComponent\Helper\SectionInvalidation`:
   `$props['sections'] = ['post' => $this->sectionInvalidation->getForPath('vendor_faq/ask/post')];`
   (exact route key, lowercased; the `*` entry such as `messages` is merged in; client-side sections dropped).
3. In the component, after success: `await customerData.reload(props.sections.post ?? [])`
   (`router.post` → in `onSuccess`; `fetch` → after `response.ok`).

Read a section with `customerData.section('vendor_faq')` (also `customer`, `cart`, `cartCount`, `isLoggedIn`).
A custom section source is registered as in core: `Magento\Customer\CustomerData\SectionPoolInterface`
`sectionSourceMap` in `etc/frontend/di.xml`. Full provider/component code: [reference/forms.md](reference/forms.md#sections).

### 6. Track analytics events

Producers track on the bus; trackers subscribe. The component that owns the data tracks:

```ts
import { useAnalyticsStore } from 'OOCS_LynxTheme/stores/global/analyticsStore'
import { config } from 'OOCS_LynxTheme/utils/storeConfig'

const analytics = useAnalyticsStore()
analytics.track('add_to_wishlist', {
    currency: config<string>('currencyCode', ''),
    value: price,
    items: [{ item_id: sku, item_name: name, price, quantity: 1 }],
})
```

- Use GA4 ecommerce names and params (`item_id` = SKU) so existing trackers understand them.
- `trackOnce(key, event, params)` for events that must not repeat on reload (once per browser session per `key`,
  `sessionStorage`), as Checkout Success does with `purchase_<order number>`.
- Track after the server accepted the action, from data the provider passed in props.
- Events Lynx already emits and their params: [reference/analytics.md](reference/analytics.md#events-lynx-tracks).

### 7. Add a tracker for an outside service

A headless PageBuilder component placed in the **Backstage** zone (on every page, checkout included): registry entry
with `zones` = `backstage`, group `utility`, `event_<name>` toggle settings and `consent_cookies`; a provider passing the
account id (empty when disabled); a component that on mount calls `analytics.addTracker(send)` (the current page's
events - its `page_view`, `view_item` - replay to it), maps Lynx events to the vendor's, loads the vendor script lazily
after consent, and `removeTracker` on unmount. Full Vue + React + di.xml + CSP example:
[reference/analytics.md](reference/analytics.md#tracker-component).

## Rules

- `forceFormData: true` + `form_key` on every Inertia POST to a Magento controller; otherwise `$_POST` is empty and the
  form key check fails. The controller itself must call the form key validator (XHR skips core's CSRF check).
- Read `g-recaptcha-response` synchronously inside the submit handler; it is reset right after the event.
- Endpoints in sample-data previews are `#`: return before posting when the URL is `#`.
- Never put per-customer data in page props or shared props; use sections.
- Section names come from `SectionInvalidation` in the provider, not hardcoded lists in the component.
- Tracker components: skip everything in the PageBuilder preview (`window.parent !== window` or path contains
  `/lynx_pagebuilder/`), count instances (not a boolean) so a remount does not drop the tracker, render only a small
  placeholder with `data-lynx-id` / `data-lynx-label` from attrs in preview.
- Vendor domains go in `etc/csp_whitelist.xml` (`script-src`, `connect-src`, `img-src`); the Lynx checkout enforces
  CSP. Stores running `bin/magento lynx:compile` must re-run it after installing the module.
- Producers never read other components' props; trackers never read page data. Everything flows through the bus.

## Check

1. `bin/magento setup:upgrade`, `bin/magento cache:clean config full_page`; new module:
   `bin/magento lynx:workspace:setup <themes>`, then `bin/magento lynx:static:deploy` (or `npm run build`).
2. Submit the form: request is `multipart/form-data` with `form_key`; success/error message appears; with a
   reCAPTCHA type selected for the key in **Stores › Configuration › Security › Google reCAPTCHA Storefront** (then
   clean `config` cache), the widget appears on focus and the token is posted.
3. After the action, `/customer/section/load?sections=…` is requested and the header (minicart, wishlist count) updates.
4. Tracker: add the component to Backstage in PageBuilder, save; on the storefront no vendor request until interaction/
   consent; `page_view` on every navigation; in the editor preview nothing is sent.
5. Test in both Vue and React themes if the store builds both.

## Reference

- [reference/forms.md](reference/forms.md) - React form, POST controller, reCAPTCHA keys and store, sections end to end
- [reference/analytics.md](reference/analytics.md) - store API, events Lynx tracks, tracker component (Vue, React, di.xml, CSP)
- https://docs.oocscommerce.com/lynx/developer/forms, https://docs.oocscommerce.com/lynx/developer/inertia#forms
