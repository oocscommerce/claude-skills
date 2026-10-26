# Controller-declared page component

Controller (`Controller/Ask/Index.php`):

```php
<?php
declare(strict_types=1);

namespace Vendor\Faq\Controller\Ask;

use Magento\Framework\App\Action\HttpGetActionInterface;
use Magento\Framework\App\RequestInterface;
use Magento\Framework\View\Result\Page;
use Magento\Framework\View\Result\PageFactory;
use OOCS\LynxInertia\Service\InertiaService;

class Index implements HttpGetActionInterface
{
    /**
     * @param InertiaService $inertiaService
     * @param PageFactory $pageFactory
     * @param RequestInterface $request
     */
    public function __construct(
        private readonly InertiaService $inertiaService,
        private readonly PageFactory $pageFactory,
        private readonly RequestInterface $request,
    ) {
    }

    /**
     * Answer with the Ask page component
     *
     * @return Page
     */
    public function execute(): Page
    {
        $this->inertiaService->setPage(
            'Vendor_Faq::Ask',
            [
                'heading' => (string) __('Ask a question'),
                'topics'  => [(string) __('Shipping'), (string) __('Returns'), (string) __('Payment')],
            ],
            $this->request->getRequestUri()
        );

        $page = $this->pageFactory->create();
        $page->getConfig()->getTitle()->set(__('Ask a question'));

        return $page;
    }
}
```

`Vendor_Faq::Ask` → `Vendor_Faq/Pages/Ask` → file `Pages/Ask.vue` under the module's
`view/frontend/lynx/vue/src/`. A theme can override it at the same relative path.

`view/frontend/lynx/vue/src/Pages/Ask.vue`:

```vue
<script setup lang="ts">
defineProps<{
    heading: string
    topics: string[]
}>()
</script>

<template>
    <div class="max-w-narrow mx-auto py-12 space-y-4">
        <h1 class="text-3xl font-semibold text-base-content">{{ heading }}</h1>
        <ul class="space-y-2">
            <li v-for="topic in topics" :key="topic">{{ topic }}</li>
        </ul>
    </div>
</template>
```

Notes:

- The page renders inside the shell `Layouts/Default`. It carries no header/footer/backstage zones: on a full load
  those areas are empty; after an Inertia visit from another page the zones already on screen stay.
- The module must be enabled and the themes set up again (`bin/magento lynx:theme:setup <themes>`) for `Pages/*` to be
  bundled; rebuild with `bin/magento lynx:static:deploy` or `npm run build`.
- Styling: design tokens only (`text-base-content`, `bg-base-100`, `rounded-box`, `max-w-site`/`max-w-narrow`…),
  font sizes from the theme scale and never below `text-sm`, daisyUI classes with the `l-` prefix.
