# Worked port: Product Questions

The Luma extension `Vendor_ProductQuestions`:

- `catalog_product_view.xml` adds `Vendor\ProductQuestions\Block\Product\Questions` to `product.info.details`, listing
  approved questions from `QuestionRepositoryInterface`.
- A KnockoutJS form posts `product_id`, `nickname`, `question` by AJAX to `productquestions/question/post`, which
  answers `{"success": true, "message": "…"}`.
- Guests must sign in unless **Allow Guests to Ask** is on. reCAPTCHA key `vendor_product_question`.
- Customer data section `product-questions` (pending questions), invalidated on `productquestions/question/post`.
- `i18n/*.csv`, some phrases used only through `$t()`.

Server side unchanged. The port is one component with one provider in `Vendor_ProductQuestionsLynx`.

## `etc/module.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Module/etc/module.xsd">
    <module name="Vendor_ProductQuestionsLynx">
        <sequence>
            <!-- The extension this module puts a Lynx storefront on -->
            <module name="Vendor_ProductQuestions"/>
            <!-- Component registry and provider contract -->
            <module name="OOCS_LynxComponent"/>
            <!-- Only for the editor page list and the starting layout -->
            <module name="OOCS_LynxPageBuilder"/>
            <!-- Only for the account frame and sidebar -->
            <module name="OOCS_LynxCustomer"/>
        </sequence>
    </module>
</config>
```

## `composer.json`

```json
{
    "name": "vendor/module-product-questions-lynx",
    "description": "Lynx storefront for Vendor_ProductQuestions",
    "type": "magento2-module",
    "require": {
        "php": ">=8.2",
        "vendor/module-product-questions": "^2.0",
        "oocscommerce/module-lynx-component": "^1.0",
        "oocscommerce/module-lynx-page-builder": "^1.0",
        "oocscommerce/module-lynx-customer": "^1.0"
    },
    "autoload": {
        "files": ["registration.php"],
        "psr-4": { "Vendor\\ProductQuestionsLynx\\": "" }
    }
}
```

Drop `module-lynx-page-builder` / `module-lynx-customer` (and their `<sequence>` lines) if you register no pages
and no account section. In the extension's own `composer.json`:
`"suggest": { "vendor/module-product-questions-lynx": "Storefront for stores running a Lynx theme" }`.

## `etc/di.xml` (global)

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="OOCS\LynxComponent\Model\Registry">
        <arguments>
            <argument name="components" xsi:type="array">
                <item name="vendor_product_questions" xsi:type="array">
                    <item name="component" xsi:type="string">Vendor_ProductQuestionsLynx::Components/Questions</item>
                    <item name="provider" xsi:type="object">Vendor\ProductQuestionsLynx\Provider\Questions\Proxy</item>
                    <item name="label" xsi:type="string">Product Questions</item>
                    <item name="description" xsi:type="string">Approved questions and answers about the product, and the form to ask one</item>
                    <item name="icon" xsi:type="string">ph-question</item>
                    <item name="group" xsi:type="string">general</item>
                    <item name="zones" xsi:type="array">
                        <item name="content" xsi:type="string">content</item>
                    </item>
                    <item name="available_on" xsi:type="array">
                        <item name="product" xsi:type="string">catalog_product_view*</item>
                    </item>
                    <item name="settings" xsi:type="array">
                        <item name="lazy" xsi:type="array">
                            <item name="label" xsi:type="string">Load when shown</item>
                            <item name="description" xsi:type="string">Off: sent with the page and visible to search engines. On: fetched only when the block appears, for example when its tab is opened</item>
                            <item name="type" xsi:type="string">toggle</item>
                            <item name="default" xsi:type="boolean">false</item>
                        </item>
                        <item name="page_size" xsi:type="array">
                            <item name="label" xsi:type="string">Questions shown</item>
                            <item name="type" xsi:type="string">number</item>
                            <item name="default" xsi:type="number">10</item>
                            <item name="min" xsi:type="number">1</item>
                            <item name="max" xsi:type="number">50</item>
                            <item name="step" xsi:type="number">1</item>
                        </item>
                    </item>
                </item>
            </argument>
        </arguments>
    </type>
</config>
```

## `Provider/Questions.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\ProductQuestionsLynx\Provider;

use Magento\Catalog\Model\Product;
use Magento\Framework\Registry;
use Magento\Framework\Stdlib\DateTime\TimezoneInterface;
use Magento\Framework\UrlInterface;
use Magento\Store\Model\StoreManagerInterface;
use OOCS\LynxComponent\Helper\SectionInvalidation;
use OOCS\LynxComponent\Model\AbstractComponentProvider;
use OOCS\LynxComponent\Model\PageCacheIdentities;
use Vendor\ProductQuestions\Api\QuestionRepositoryInterface;
use Vendor\ProductQuestions\Model\Config;
use Vendor\ProductQuestions\Model\Question;

class Questions extends AbstractComponentProvider
{
    public function __construct(
        private readonly Registry $registry,
        private readonly Config $config,
        private readonly QuestionRepositoryInterface $questionRepository,
        private readonly StoreManagerInterface $storeManager,
        private readonly TimezoneInterface $timezone,
        private readonly UrlInterface $urlBuilder,
        private readonly SectionInvalidation $sectionInvalidation,
        private readonly PageCacheIdentities $pageCacheIdentities,
    ) {
    }

    /**
     * Left out of the page until the component asks, when lazy
     *
     * @param array $props
     * @return bool
     */
    public function optional(array $props): bool
    {
        // Never lazy when disabled: Tabs hides a tab whose block answers enabled: false
        return $this->config->isEnabled() && !empty($props['lazy']);
    }

    /**
     * Approved questions of the viewed product and what the form needs
     *
     * @param array $props
     * @return array
     */
    public function resolve(array $props): array
    {
        /** @var Product|null $product */
        $product = $this->registry->registry('current_product');
        if (!$product) {
            return $props;
        }

        $props['enabled'] = $this->config->isEnabled();
        if (!$props['enabled']) {
            return $props;
        }

        // The tags the Luma block returned from getIdentities(); a lazy request is cached on its own
        $tags = [Question::CACHE_TAG];
        if (!empty($props['lazy'])) {
            $tags[] = Product::CACHE_TAG . '_' . $product->getId();
        }
        $this->pageCacheIdentities->add($tags);

        $questions = $this->questionRepository->getApprovedForProduct(
            (int) $product->getId(),
            (int) $this->storeManager->getStore()->getId(),
            max(1, (int) $props['page_size'])
        );

        $props['product_id'] = (int) $product->getId();
        $props['allow_guest'] = $this->config->isGuestAllowed();
        $props['items'] = array_map(fn ($question) => [
            'id'       => (int) $question->getId(),
            'question' => (string) $question->getQuestion(),
            'answer'   => (string) $question->getAnswer(),
            'nickname' => (string) $question->getNickname(),
            'date'     => $this->timezone->formatDate($question->getCreatedAt(), \IntlDateFormatter::MEDIUM),
        ], $questions);
        $props['urls'] = [
            'post'  => $this->urlBuilder->getUrl('productquestions/question/post'),
            'login' => $this->urlBuilder->getUrl('customer/account/login'),
        ];
        // From the extension's sections.xml: what to reload after a successful post
        $props['sections'] = $this->sectionInvalidation->getForPath('productquestions/question/post');
        // The extension's own phrases, so its CSV files translate them
        $props['labels'] = [
            'title'         => (string) __('Questions & Answers'),
            'empty'         => (string) __('No questions yet. Be the first to ask.'),
            'askedBy'       => (string) __('Asked by %1 on %2'),
            'answer'        => (string) __('Answer'),
            'ask'           => (string) __('Ask a Question'),
            'nickname'      => (string) __('Nickname'),
            'question'      => (string) __('Your Question'),
            'submit'        => (string) __('Submit Question'),
            'required'      => (string) __('This is a required field.'),
            'pending'       => (string) __('Awaiting moderation'),
            'loginRequired' => (string) __('Only registered customers can ask questions. Please %1.'),
            'signIn'        => (string) __('Sign In'),
            'error'         => (string) __('We can\'t submit your question right now.'),
        ];

        return $props;
    }
}
```

Carried over from the Luma block:

- No customer session in `resolve()`: the product page is in the full page cache.
- Cache tags: the block's `getIdentities()` no longer runs; add the same tags through `PageCacheIdentities`. The
  product page already has the product's tags, so only a lazy request adds them too.
- Labels through `__()`; the extension's CSVs (including `$t()` phrases) translate them.
- `optional()` implements the `lazy` convention (see `provider`).

## `view/frontend/lynx/vue/src/Components/Questions.vue`

```vue
<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'
import { Link, router } from '@inertiajs/vue3'
import { useCustomerDataStore } from 'OOCS_LynxTheme/stores/global/customerDataStore'
import { useFormKeyStore } from 'OOCS_LynxTheme/stores/global/formKeyStore'
import { useMessagesStore } from 'OOCS_LynxTheme/stores/global/messagesStore'

type Item = { id: number; question: string; answer: string; nickname: string; date: string }
type Pending = { product_id: number; question: string }

const props = withDefaults(
    defineProps<{
        lazy: boolean
        _key?: string
        // Written first by the provider: undefined means not fetched yet
        enabled?: boolean
        product_id?: number
        allow_guest?: boolean
        items?: Item[]
        urls?: { post: string; login: string }
        sections?: string[]
        labels?: Record<string, string>
    }>(),
    {
        _key: '',
        enabled: undefined,
        product_id: 0,
        allow_guest: false,
        items: () => [],
        urls: () => ({ post: '', login: '' }),
        sections: () => [],
        labels: () => ({}),
    }
)

const customerData = useCustomerDataStore()
const formKey = useFormKeyStore()
const messages = useMessagesStore()

onMounted(() => {
    if (props.lazy && props.enabled === undefined && props._key) {
        router.reload({ only: [props._key], preserveUrl: true })
    }
})

const canAsk = computed(() => props.allow_guest || customerData.isLoggedIn)
// The extension's own section, per visitor, so never part of the cached page
const pending = computed(() =>
    ((customerData.section('product-questions')?.pending ?? []) as Pending[]).filter(
        p => p.product_id === props.product_id
    )
)
const loginText = computed(() => (props.labels.loginRequired ?? '').split('%1'))

const nickname = ref('')
const question = ref('')
const error = ref('')
const busy = ref(false)

onMounted(() => {
    nickname.value ||= customerData.displayName
})

async function submit(event: SubmitEvent) {
    const form = event.currentTarget as HTMLFormElement
    // Read now: the reCAPTCHA widget resets once the submit event is over
    const captcha = (form.elements.namedItem('g-recaptcha-response') as HTMLTextAreaElement | null)?.value
    error.value = question.value.trim() && nickname.value.trim() ? '' : (props.labels.required ?? '')
    if (error.value || busy.value) return

    busy.value = true
    try {
        const response = await fetch(props.urls.post, {
            method: 'POST',
            credentials: 'same-origin',
            headers: {
                'X-Requested-With': 'XMLHttpRequest',
                'Content-Type': 'application/x-www-form-urlencoded',
            },
            body: new URLSearchParams({
                form_key: formKey.ensure(),
                product_id: String(props.product_id),
                nickname: nickname.value,
                question: question.value,
                ...(captcha ? { 'g-recaptcha-response': captcha } : {}),
            }),
        })
        const result: { success?: boolean; message?: string } = await response.json()
        messages.addServerMessage(result.success ? 'success' : 'error', result.message)
        if (result.success) {
            question.value = ''
            await customerData.reload(props.sections)
        }
    } catch {
        messages.add('error', props.labels.error ?? '')
    } finally {
        busy.value = false
    }
}
</script>

<template>
    <div v-if="enabled === undefined" class="grid w-full gap-3" aria-busy="true">
        <span v-for="n in 3" :key="n" class="l-skeleton block h-4 w-full"></span>
    </div>
    <section v-else-if="enabled" class="grid w-full gap-8">
        <h2 class="text-lg font-semibold text-base-content">{{ labels.title }}</h2>

        <p v-if="!items.length" class="text-base-content/70">{{ labels.empty }}</p>
        <ul v-else class="divide-y divide-base-300">
            <li v-for="item in items" :key="item.id" class="grid gap-2 py-4">
                <p class="font-semibold text-base-content">{{ item.question }}</p>
                <p class="text-sm text-base-content/60">
                    {{ (labels.askedBy ?? '').replace('%1', item.nickname).replace('%2', item.date) }}
                </p>
                <p v-if="item.answer" class="border-s-(length:--border) border-base-300 ps-4 text-base-content/70">
                    <span class="font-semibold text-base-content">{{ labels.answer }}:</span> {{ item.answer }}
                </p>
            </li>
        </ul>

        <div v-for="p in pending" :key="p.question" class="l-alert">
            <span class="font-semibold">{{ labels.pending }}:</span> {{ p.question }}
        </div>

        <form
            v-if="canAsk"
            :action="urls.post"
            data-recaptcha="vendor_product_question"
            novalidate
            class="grid gap-4"
            @submit.prevent="submit"
        >
            <h3 class="text-lg font-semibold text-base-content">{{ labels.ask }}</h3>
            <label class="grid gap-1">
                <span class="text-sm font-semibold text-base-content">{{ labels.nickname }}</span>
                <input v-model="nickname" name="nickname" class="l-input w-full" required />
            </label>
            <label class="grid gap-1">
                <span class="text-sm font-semibold text-base-content">{{ labels.question }}</span>
                <textarea v-model="question" name="question" rows="4" class="l-textarea w-full" required></textarea>
            </label>
            <p v-if="error" class="text-sm text-error" role="alert">{{ error }}</p>
            <div data-recaptcha-slot></div>
            <div>
                <button type="submit" class="l-btn l-btn-primary" :disabled="busy">{{ labels.submit }}</button>
            </div>
        </form>
        <p v-else class="text-base-content/70">
            {{ loginText[0] }}<Link :href="urls.login" class="text-link">{{ labels.signIn }}</Link>{{ loginText[1] }}
        </p>
    </section>
    <div v-else></div>
</template>
```

## `view/frontend/lynx/react/src/Components/Questions.tsx`

```tsx
import { useEffect, useState, type FormEvent } from 'react'
import { Link, router } from '@inertiajs/react'
import { useCustomerDataStore } from 'OOCS_LynxTheme/stores/global/customerDataStore'
import { useFormKeyStore } from 'OOCS_LynxTheme/stores/global/formKeyStore'
import { useMessagesStore } from 'OOCS_LynxTheme/stores/global/messagesStore'

type Item = { id: number; question: string; answer: string; nickname: string; date: string }
type Pending = { product_id: number; question: string }

type Props = {
    lazy: boolean
    _key?: string
    // Written first by the provider: undefined means not fetched yet
    enabled?: boolean
    product_id?: number
    allow_guest?: boolean
    items?: Item[]
    urls?: { post: string; login: string }
    sections?: string[]
    labels?: Record<string, string>
}

// Stable defaults, so hooks do not see a new dependency on every render
const EMPTY_LIST: never[] = []
const EMPTY_MAP: Record<string, never> = {}
const DEFAULT_URLS = { post: '', login: '' }

export default function Questions({
    lazy,
    _key = '',
    enabled,
    product_id = 0,
    allow_guest = false,
    items = EMPTY_LIST,
    urls = DEFAULT_URLS,
    sections = EMPTY_LIST,
    labels = EMPTY_MAP,
}: Props) {
    const customerData = useCustomerDataStore()
    const formKey = useFormKeyStore()
    const messages = useMessagesStore()

    const [nickname, setNickname] = useState('')
    const [question, setQuestion] = useState('')
    const [error, setError] = useState('')
    const [busy, setBusy] = useState(false)

    useEffect(() => {
        if (lazy && enabled === undefined && _key) {
            router.reload({ only: [_key], preserveUrl: true })
        }
    }, [lazy, enabled, _key])

    useEffect(() => {
        setNickname(n => n || customerData.displayName)
    }, [customerData.displayName])

    const canAsk = allow_guest || customerData.isLoggedIn
    // The extension's own section, per visitor, so never part of the cached page
    const pending = ((customerData.section('product-questions')?.pending ?? []) as Pending[]).filter(
        p => p.product_id === product_id
    )
    const [loginBefore, loginAfter] = (labels.loginRequired ?? '').split('%1')

    async function submit(event: FormEvent<HTMLFormElement>) {
        event.preventDefault()
        const form = event.currentTarget
        // Read now: the reCAPTCHA widget resets once the submit event is over
        const captcha = (form.elements.namedItem('g-recaptcha-response') as HTMLTextAreaElement | null)?.value
        const message = question.trim() && nickname.trim() ? '' : (labels.required ?? '')
        setError(message)
        if (message || busy) return

        setBusy(true)
        try {
            const response = await fetch(urls.post, {
                method: 'POST',
                credentials: 'same-origin',
                headers: {
                    'X-Requested-With': 'XMLHttpRequest',
                    'Content-Type': 'application/x-www-form-urlencoded',
                },
                body: new URLSearchParams({
                    form_key: formKey.ensure(),
                    product_id: String(product_id),
                    nickname,
                    question,
                    ...(captcha ? { 'g-recaptcha-response': captcha } : {}),
                }),
            })
            const result: { success?: boolean; message?: string } = await response.json()
            messages.addServerMessage(result.success ? 'success' : 'error', result.message)
            if (result.success) {
                setQuestion('')
                await customerData.reload(sections)
            }
        } catch {
            messages.add('error', labels.error ?? '')
        } finally {
            setBusy(false)
        }
    }

    if (enabled === undefined) {
        return (
            <div className="grid w-full gap-3" aria-busy="true">
                {[1, 2, 3].map(n => <span key={n} className="l-skeleton block h-4 w-full"></span>)}
            </div>
        )
    }
    if (!enabled) return <div />

    return (
        <section className="grid w-full gap-8">
            <h2 className="text-lg font-semibold text-base-content">{labels.title}</h2>

            {!items.length ? (
                <p className="text-base-content/70">{labels.empty}</p>
            ) : (
                <ul className="divide-y divide-base-300">
                    {items.map(item => (
                        <li key={item.id} className="grid gap-2 py-4">
                            <p className="font-semibold text-base-content">{item.question}</p>
                            <p className="text-sm text-base-content/60">
                                {(labels.askedBy ?? '').replace('%1', item.nickname).replace('%2', item.date)}
                            </p>
                            {item.answer && (
                                <p className="border-s-(length:--border) border-base-300 ps-4 text-base-content/70">
                                    <span className="font-semibold text-base-content">{labels.answer}:</span> {item.answer}
                                </p>
                            )}
                        </li>
                    ))}
                </ul>
            )}

            {pending.map(p => (
                <div key={p.question} className="l-alert">
                    <span className="font-semibold">{labels.pending}:</span> {p.question}
                </div>
            ))}

            {canAsk ? (
                <form action={urls.post} data-recaptcha="vendor_product_question" noValidate className="grid gap-4" onSubmit={submit}>
                    <h3 className="text-lg font-semibold text-base-content">{labels.ask}</h3>
                    <label className="grid gap-1">
                        <span className="text-sm font-semibold text-base-content">{labels.nickname}</span>
                        <input value={nickname} onChange={e => setNickname(e.target.value)} name="nickname" className="l-input w-full" required />
                    </label>
                    <label className="grid gap-1">
                        <span className="text-sm font-semibold text-base-content">{labels.question}</span>
                        <textarea value={question} onChange={e => setQuestion(e.target.value)} name="question" rows={4} className="l-textarea w-full" required />
                    </label>
                    {error && <p className="text-sm text-error" role="alert">{error}</p>}
                    <div data-recaptcha-slot></div>
                    <div>
                        <button type="submit" className="l-btn l-btn-primary" disabled={busy}>{labels.submit}</button>
                    </div>
                </form>
            ) : (
                <p className="text-base-content/70">
                    {loginBefore}<Link href={urls.login} className="text-link">{labels.signIn}</Link>{loginAfter}
                </p>
            )}
        </section>
    )
}
```

Follows the Lynx rules: tokens and `l-` controls, logical classes for RTL, one root element in every state, no
`window`/`document` during render, no text of its own (all from `labels` or the server's JSON message).

## Install guide line for merchants

In the editor, open a product page, add a **Tab** to the product's Tabs and drop **Product Questions** into it (or
into **Below Add to Cart**).

## A public listing page (`/productquestions/index/index`)

`lynx_productquestions_index_index.xml` for title/meta, a component with a provider, a `pages` entry without
`sample_data` (the preview opens the real route), and a starting layout in the default design. Account page
`/productquestions/my/index`: see `account-tab`.
