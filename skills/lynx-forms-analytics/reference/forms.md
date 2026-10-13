# Forms: full examples

## React form (Inertia post)

```tsx
import { useState, type FormEvent } from 'react'
import { router } from '@inertiajs/react'
import { useFormKeyStore } from 'OOCS_LynxTheme/stores/global/formKeyStore'

type Props = { urls: { post: string }; labels: Record<string, string> }

export default function AskForm({ urls, labels }: Props) {
    const formKey = useFormKeyStore()
    const [question, setQuestion] = useState('')
    const [busy, setBusy] = useState(false)

    function submit(event: FormEvent<HTMLFormElement>) {
        event.preventDefault()
        // Sample preview endpoint: nothing to send
        if (urls.post === '#' || busy) return
        // Read now: the reCAPTCHA widget resets once the submit event is over
        const captcha = (event.currentTarget.elements.namedItem('g-recaptcha-response') as HTMLTextAreaElement | null)?.value
        setBusy(true)
        router.post(
            urls.post,
            { form_key: formKey.ensure(), question, ...(captcha ? { 'g-recaptcha-response': captcha } : {}) },
            { forceFormData: true, preserveScroll: true, onFinish: () => setBusy(false) }
        )
    }

    return (
        <form data-recaptcha="vendor_faq_ask" noValidate onSubmit={submit}>
            <textarea name="question" value={question} onChange={e => setQuestion(e.target.value)} className="l-textarea w-full" />
            <div data-recaptcha-slot />
            <button className="l-btn l-btn-primary" disabled={busy}>{labels.submit}</button>
        </form>
    )
}
```

## POST controller

```php
<?php
declare(strict_types=1);

namespace Vendor\Faq\Controller\Ask;

use Magento\Framework\App\Action\HttpPostActionInterface;
use Magento\Framework\App\RequestInterface;
use Magento\Framework\Controller\Result\Redirect;
use Magento\Framework\Controller\Result\RedirectFactory;
use Magento\Framework\Data\Form\FormKey\Validator as FormKeyValidator;
use Magento\Framework\Message\ManagerInterface;

class Post implements HttpPostActionInterface
{
    /**
     * @param RequestInterface $request
     * @param FormKeyValidator $formKeyValidator
     * @param ManagerInterface $messageManager
     * @param RedirectFactory $redirectFactory
     */
    public function __construct(
        private readonly RequestInterface $request,
        private readonly FormKeyValidator $formKeyValidator,
        private readonly ManagerInterface $messageManager,
        private readonly RedirectFactory $redirectFactory,
    ) {
    }

    /**
     * Save the question and go back to the FAQ page
     *
     * @return Redirect
     */
    public function execute(): Redirect
    {
        $redirect = $this->redirectFactory->create()->setPath('faq');
        // XHR posts skip core's CSRF check, so the form key is checked here
        if (!$this->formKeyValidator->validate($this->request)) {
            $this->messageManager->addErrorMessage(__('Invalid Form Key. Please refresh the page.'));
            return $redirect;
        }

        $question = trim((string) $this->request->getParam('question'));
        if ($question === '') {
            $this->messageManager->addErrorMessage(__('This is a required field.'));
        } else {
            // save the question
            $this->messageManager->addSuccessMessage(__('Thank you for your question.'));
        }

        return $redirect;
    }
}
```

Inertia follows the redirect as a visit; the theme drains the `mage-messages` cookie on `navigate` and shows the
messages. A classic `<form method="post" :action="urls.post">` with a hidden `form_key` input works the same with a
full reload (Lynx's registration form posts this way).

## reCAPTCHA

Keys Lynx's own forms use:

| Key | Lynx form |
|---|---|
| `customer_login` | Sign in |
| `customer_create` | Create an account |
| `customer_forgot_password` | Forgot password |
| `customer_edit` | Account information |
| `contact` | Contact us |
| `newsletter` | Newsletter subscription |
| `product_review` | Product review |
| `coupon_code` | Discount code (cart, checkout) |
| `place_order` | Place order (checkout) |

Any other key under `recaptcha_frontend/type_for` (Magento's `sendfriend`, `wishlist`, or another module's) works the
same. A key with no type selected, or without site key and secret, is ignored. reCAPTCHA settings are cached with the
storefront config: clean the `config` cache after changing them.

Widget behavior: drawn on first focus inside the form, in `[data-recaptcha-slot]` or just above the block holding the
submit button. With a token the submit proceeds and the widget resets; without one the submit is stopped (checkbox:
shown with Magento's failure message; invisible / v3: token fetched, form re-submitted).

### A new key for your own route

Magento's way, no Lynx code:

1. `etc/adminhtml/system.xml`: a field under `recaptcha_frontend/type_for/<key>` (copy a field from
   `Magento_ReCaptchaContact`'s `system.xml`).
2. An observer on `controller_action_predispatch_<your full action name>` calling
   `Magento\ReCaptchaUi\Model\RequestHandlerInterface::execute($key, $request, $response, $redirectOnFailureUrl)`
   when `Magento\ReCaptchaUi\Model\IsCaptchaEnabledInterface::isCaptchaEnabledFor($key)` (as Magento's ReCaptcha
   modules do).
3. `data-recaptcha="<key>"` on the form.

### Actions that are not form submits

```ts
import { useRecaptchaStore } from 'OOCS_LynxReCaptcha/stores/recaptchaStore'

const recaptcha = useRecaptchaStore()

// When the action becomes visible: draw a checkbox widget at the anchor ahead of time
recaptcha.prepare('vendor_foo_action', anchorEl)

// When the shopper acts
const token = await recaptcha.resolve('vendor_foo_action', anchorEl)
if (token === null) return // checkbox not ticked yet; the widget is shown at the anchor
await fetch(url, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json', ...(token ? { 'X-ReCaptcha': token } : {}) },
    body: JSON.stringify(payload),
})
```

`resolve()` returns `''` when the key is not protected (no reCAPTCHA component mounted), `null` while a checkbox still
needs ticking, the token otherwise. Same import in Vue and React. This import needs `OOCS_LynxReCaptcha` enabled in the
build: add it to your module's `<sequence>` and `composer.json` (`oocscommerce/module-lynx-re-captcha`) when you use it.

## Sections

Provider (fragment):

```php
use OOCS\LynxComponent\Helper\SectionInvalidation;

public function __construct(
    private readonly SectionInvalidation $sectionInvalidation,
    private readonly \Magento\Framework\UrlInterface $urlBuilder,
) {
}

public function resolve(array $props): array
{
    $props['urls'] = ['post' => $this->urlBuilder->getUrl('vendor_faq/ask/post')];
    // Names from the merged sections.xml for this exact route, plus the "*" entry
    $props['sections'] = ['post' => $this->sectionInvalidation->getForPath('vendor_faq/ask/post')];

    return $props;
}
```

Component, `fetch` flavor (Vue; React identical with the same store API):

```ts
import { useCustomerDataStore } from 'OOCS_LynxTheme/stores/global/customerDataStore'
import { useFormKeyStore } from 'OOCS_LynxTheme/stores/global/formKeyStore'
import { useMessagesStore } from 'OOCS_LynxTheme/stores/global/messagesStore'

const customerData = useCustomerDataStore()
const formKey = useFormKeyStore()
const messages = useMessagesStore()

async function send() {
    if (props.urls.post === '#') return
    try {
        const response = await fetch(props.urls.post, {
            method: 'POST',
            headers: { 'X-Requested-With': 'XMLHttpRequest', 'Content-Type': 'application/x-www-form-urlencoded' },
            credentials: 'same-origin',
            body: new URLSearchParams({ form_key: formKey.ensure(), question: question.value }),
        })
        // A redirect result left its messages in the cookie
        messages.sync()
        if (response.ok) await customerData.reload(props.sections.post ?? [])
    } catch {
        messages.add('error', props.labels.error ?? '')
    }
}
```

`router.post` flavor: call `void customerData.reload(sections)` in `onSuccess`.

`reload(names)` marks the names stale, requests `/customer/section/load?sections=…&force_new_section_timestamp=true`,
shows any `messages` section content, stores the rest (IndexedDB) and updates `section_data_ids`. After every Inertia
visit the theme also calls `load()`, which refetches only sections that are stale or past their lifetime.

Customer-data store API: `section(name)`, `customer`, `cart`, `cartCount`, `wishlistCount`, `displayName`,
`isLoggedIn`, `reload(names)`, `invalidate(names)`, `load(force?)`.

A section source of your own, `etc/frontend/di.xml`:

```xml
<type name="Magento\Customer\CustomerData\SectionPoolInterface">
    <arguments>
        <argument name="sectionSourceMap" xsi:type="array">
            <item name="vendor_faq" xsi:type="string">Vendor\Faq\CustomerData\Questions</item>
        </argument>
    </arguments>
</type>
```

`Vendor\Faq\CustomerData\Questions` implements `Magento\Customer\CustomerData\SectionSourceInterface::getSectionData()`.
