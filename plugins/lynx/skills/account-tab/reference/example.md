# My Account tab: full example

Module `Vendor_ProductQuestionsLynx`, tab key `product_questions`, path `productquestions_my_index`. The repository
`Vendor\ProductQuestions\Api\QuestionRepositoryInterface::getListForCustomer()` stands for whatever service the
extension offers.

## Provider

`Provider/AccountQuestions.php`. `resolve()` runs first even in the editor preview; `resolveSample()` then overlays
sample rows on its result.

```php
<?php
declare(strict_types=1);

namespace Vendor\ProductQuestionsLynx\Provider;

use Magento\Customer\Model\Session as CustomerSession;
use Magento\Framework\Stdlib\DateTime\TimezoneInterface;
use OOCS\LynxComponent\Api\SampleProviderInterface;
use OOCS\LynxComponent\Model\AbstractComponentProvider;
use Vendor\ProductQuestions\Api\QuestionRepositoryInterface;

class AccountQuestions extends AbstractComponentProvider implements SampleProviderInterface
{
    public function __construct(
        private readonly CustomerSession $customerSession,
        private readonly QuestionRepositoryInterface $questionRepository,
        private readonly TimezoneInterface $timezone,
    ) {
    }

    /**
     * The signed-in customer's questions
     *
     * @param array $props
     * @return array
     */
    public function resolve(array $props): array
    {
        $props['labels'] = $this->getLabels();
        // The controller redirects guests; the page is not cached, so the session is safe to read
        if (!$this->customerSession->isLoggedIn()) {
            return $props;
        }

        $props['items'] = array_map(fn ($question) => [
            'id'       => (int) $question->getId(),
            'product'  => (string) $question->getProductName(),
            'url'      => (string) $question->getProductUrl(),
            'question' => (string) $question->getQuestion(),
            'answer'   => (string) $question->getAnswer(),
            'date'     => $this->timezone->formatDate($question->getCreatedAt(), \IntlDateFormatter::MEDIUM),
        ], $this->questionRepository->getListForCustomer((int) $this->customerSession->getCustomerId()));

        return $props;
    }

    /**
     * Sample rows for the editor preview
     *
     * @param array $resolved
     * @param array $props
     * @return array
     */
    public function resolveSample(array $resolved, array $props): array
    {
        $resolved['items'] = [[
            'id'       => 1,
            'product'  => 'Sample Product',
            'url'      => '#',
            'question' => 'Is this machine washable?',
            'answer'   => 'Yes, at 30 degrees.',
            'date'     => $this->timezone->formatDate(null, \IntlDateFormatter::MEDIUM),
        ]];

        return $resolved;
    }

    /**
     * @return array<string, string>
     */
    private function getLabels(): array
    {
        return [
            'title'   => (string) __('My Product Questions'),
            'empty'   => (string) __('You have not asked any questions yet.'),
            'answer'  => (string) __('Answer'),
            'waiting' => (string) __('Awaiting moderation'),
        ];
    }
}
```

## Component

Vue, `view/frontend/lynx/vue/src/Components/AccountQuestions.vue`:

```vue
<script setup lang="ts">
import { Link } from '@inertiajs/vue3'

withDefaults(defineProps<Lynx.VendorAccountProductQuestions>(), {
    items: () => [],
    labels: () => ({}),
})
</script>

<template>
    <div class="grid w-full gap-6">
        <h1 class="text-2xl font-semibold text-base-content">{{ labels.title }}</h1>
        <p v-if="!items.length" class="text-base-content/70">{{ labels.empty }}</p>
        <ul v-else class="divide-y divide-base-300">
            <li v-for="item in items" :key="item.id" class="grid gap-1 py-4">
                <Link :href="item.url" class="text-sm text-link">{{ item.product }} · {{ item.date }}</Link>
                <p class="font-semibold text-base-content">{{ item.question }}</p>
                <p class="text-base-content/70">
                    {{ item.answer ? `${labels.answer}: ${item.answer}` : labels.waiting }}
                </p>
            </li>
        </ul>
    </div>
</template>
```

React, `view/frontend/lynx/react/src/Components/AccountQuestions.tsx`:

```tsx
import { Link } from '@inertiajs/react'

export default function AccountQuestions({ items = [], labels = {} }: Lynx.VendorAccountProductQuestions) {
    return (
        <div className="grid w-full gap-6">
            <h1 className="text-2xl font-semibold text-base-content">{labels.title}</h1>
            {!items.length ? (
                <p className="text-base-content/70">{labels.empty}</p>
            ) : (
                <ul className="divide-y divide-base-300">
                    {items.map(item => (
                        <li key={item.id} className="grid gap-1 py-4">
                            <Link href={item.url} className="text-sm text-link">{item.product} · {item.date}</Link>
                            <p className="font-semibold text-base-content">{item.question}</p>
                            <p className="text-base-content/70">
                                {item.answer ? `${labels.answer}: ${item.answer}` : labels.waiting}
                            </p>
                        </li>
                    ))}
                </ul>
            )}
        </div>
    )
}
```

The section sits inside the frame: root `w-full`, no outer spacing. A section with actions (remove, post) returns
early when the URL is `'#'` so the sample preview never sends anything.

## Starting layout

`Setup/Patch/Data/InstallAccountLayout.php`: the My Account frame, told which tab is current, with the section in its
slot, written to the default design only when that path has no row yet.

```php
<?php
declare(strict_types=1);

namespace Vendor\ProductQuestionsLynx\Setup\Patch\Data;

use Magento\Framework\Serialize\Serializer\Json;
use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\Patch\DataPatchInterface;
use OOCS\LynxPageBuilder\Model\Design;
use OOCS\LynxPageBuilder\Setup\Patch\Data\CreateDefaultDesign;

class InstallAccountLayout implements DataPatchInterface
{
    private const PATH = 'productquestions_my_index';

    /**
     * @param ModuleDataSetupInterface $moduleDataSetup
     * @param Json $json
     */
    public function __construct(
        private readonly ModuleDataSetupInterface $moduleDataSetup,
        private readonly Json $json,
    ) {
    }

    /**
     * The default design is created there
     *
     * @return string[]
     */
    public static function getDependencies(): array
    {
        return [CreateDefaultDesign::class];
    }

    /**
     * @return string[]
     */
    public function getAliases(): array
    {
        return [];
    }

    /**
     * Write the tab's layout to the default design unless a row is already there
     *
     * @return self
     */
    public function apply(): self
    {
        $connection = $this->moduleDataSetup->getConnection();
        $table = $this->moduleDataSetup->getTable('lynx_pagebuilder_layout');

        $saved = $connection->fetchOne(
            $connection->select()
                ->from($table, ['path'])
                ->where('design_id = ?', Design::DEFAULT_ID)
                ->where('path = ?', self::PATH)
        );
        if (!$saved) {
            $connection->insert($table, [
                'design_id' => Design::DEFAULT_ID,
                'path'      => self::PATH,
                'value'     => $this->json->serialize($this->getNodes()),
            ]);
        }

        return $this;
    }

    /**
     * Group > My Account frame (this tab current) > the section
     *
     * @return array
     */
    private function getNodes(): array
    {
        return [[
            'name'      => 'default_group',
            'component' => 'OOCS_LynxThemeComponent/Components/Group',
            'label'     => 'Group',
            'visible'   => true,
            'props'     => ['layout' => 'constrained', 'width' => 'site', 'padding_y' => 'sm', 'padding_x' => 'none'],
            'slots'     => ['default' => [[
                'name'      => 'default_account',
                'component' => 'OOCS_LynxCustomer/Components/MyAccount',
                'label'     => 'My Account',
                'visible'   => true,
                // The AccountTabs key to highlight
                'props'     => ['active_tab' => 'product_questions'],
                'slots'     => ['default' => [[
                    'name'      => 'vendor_account_product_questions',
                    'component' => 'Vendor_ProductQuestionsLynx/Components/AccountQuestions',
                    'label'     => 'My Product Questions',
                    'visible'   => true,
                    'props'     => [],
                ]]],
            ]]],
        ]];
    }
}
```

`OOCS_LynxThemeComponent` is installed with Lynx (the Group component); the patch only writes JSON, so it needs no
module dependency on it beyond `OOCS_LynxPageBuilder` for the classes used.
