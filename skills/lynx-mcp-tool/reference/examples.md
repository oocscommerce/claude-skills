# Lynx MCP: more examples and server facts

## A tool that writes

`cms_save_block` replaces the content of CMS blocks, all or nothing. ACL `Vendor_Module::tool_cms_save_block`
under `OOCS_LynxMcp::tools`; registered as `<item name="cms_save_block" xsi:type="object">Vendor\Module\Tool\Blocks\SaveBlock\Proxy</item>`.

`Tool/Blocks/SaveBlock.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Tool\Blocks;

use Magento\Cms\Api\BlockRepositoryInterface;
use Magento\Cms\Api\GetBlockByIdentifierInterface;
use Magento\Framework\App\ResourceConnection;
use Magento\Framework\Exception\LocalizedException;
use Magento\Framework\Exception\NoSuchEntityException;
use Magento\Store\Model\Store;
use OOCS\LynxMcp\Api\ToolInterface;

class SaveBlock implements ToolInterface
{
    private const MAX_BLOCKS = 20;

    /**
     * @param GetBlockByIdentifierInterface $getBlockByIdentifier
     * @param BlockRepositoryInterface $blockRepository
     * @param ResourceConnection $resource
     */
    public function __construct(
        private readonly GetBlockByIdentifierInterface $getBlockByIdentifier,
        private readonly BlockRepositoryInterface $blockRepository,
        private readonly ResourceConnection $resource,
    ) {
    }

    public function getDescription(): string
    {
        return 'Replace the HTML content of one or more CMS blocks (all store views), identified by identifier. '
            . 'Either every block is saved or none. Read a block first with cms_read_block.';
    }

    public function getInputSchema(): array
    {
        return [
            'type'       => 'object',
            'properties' => [
                'blocks' => [
                    'type'     => 'array',
                    'maxItems' => self::MAX_BLOCKS,
                    'items'    => [
                        'type'       => 'object',
                        'properties' => [
                            'identifier' => ['type' => 'string'],
                            'content'    => ['type' => 'string', 'description' => 'New HTML content'],
                        ],
                        'required'   => ['identifier', 'content'],
                    ],
                ],
            ],
            'required'   => ['blocks'],
        ];
    }

    public function getAclResource(): string
    {
        return 'Vendor_Module::tool_cms_save_block';
    }

    public function isReadOnly(): bool
    {
        return false;
    }

    public function execute(array $arguments): array
    {
        $blocks = $arguments['blocks'] ?? null;
        if (!\is_array($blocks) || $blocks === [] || \count($blocks) > self::MAX_BLOCKS) {
            throw new LocalizedException(__('blocks must hold 1 to %1 items.', self::MAX_BLOCKS));
        }

        // Validate everything before the first write
        $toSave = [];
        foreach ($blocks as $i => $item) {
            $identifier = trim((string) ($item['identifier'] ?? ''));
            if ($identifier === '' || !isset($item['content'])) {
                throw new LocalizedException(__('blocks[%1] needs identifier and content.', $i));
            }
            try {
                $block = $this->getBlockByIdentifier->execute($identifier, Store::DEFAULT_STORE_ID);
            } catch (NoSuchEntityException) {
                throw new LocalizedException(__('No block with identifier "%1".', $identifier));
            }
            $toSave[] = $block->setContent((string) $item['content']);
        }

        // A worker past --timeout is killed: the transaction keeps a half-done batch out of the database
        $connection = $this->resource->getConnection();
        $connection->beginTransaction();
        try {
            foreach ($toSave as $block) {
                $this->blockRepository->save($block);
            }
            $connection->commit();
        } catch (\Throwable $e) {
            $connection->rollBack();
            throw $e;
        }

        return ['saved' => array_map(static fn ($block) => $block->getIdentifier(), $toSave)];
    }
}
```

## An HTTP route

`Service/Route/Health.php`

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Service\Route;

use OOCS\LynxMcp\Api\RouteInterface;

class Health implements RouteInterface
{
    public function getMethods(): array
    {
        return ['GET'];
    }

    /**
     * @param array $request method, path, query, headers (lower-case names), body
     * @return array [status, headers, body]; an array body is sent as JSON
     */
    public function handle(array $request): array
    {
        return [200, [], ['status' => 'ok']];
    }
}
```

`etc/di.xml` (global):

```xml
<type name="OOCS\LynxMcp\Model\RouteList">
    <arguments>
        <argument name="routes" xsi:type="array">
            <item name="/health" xsi:type="object">Vendor\Module\Service\Route\Health</item>
        </argument>
    </arguments>
</type>
```

`RouteList` throws at start when a path does not start with `/` or the class does not implement `RouteInterface`.
Routes are constructed in the master at start (no proxy in the built-in list); keep their constructors free of work.
The server answers 403 for an `Origin` not in **Allowed Origins** before any route runs, and 404 for unknown paths.

## A layout source

Implement `OOCS\LynxPageBuilder\Api\LayoutSourceInterface` when a module stores CMS page or category layouts its
own way. Every method taking an id throws `NoSuchEntityException` for an unknown entity.

| Method | Purpose |
|---|---|
| `getPagePath(): string` | Full action name the entity renders under (`cms_page_view`); filters components |
| `resolveId(string $key, int $storeId): int` | Entity id from an id or another key (CMS identifier) |
| `getLabel(int $id, int $storeId): string` | Title in the store view |
| `getNodes(int $id, int $storeId): array` | Saved layout nodes |
| `saveNodes(int $id, int $storeId, array $nodes): void` | Store nodes; empty list removes the layout |
| `isEnabled(int $id, int $storeId): bool` / `setEnabled(int $id, int $storeId, bool $enabled): void` | **Render with Lynx** switch |
| `isAvailable(int $id, int $storeId): bool` | Store view shows the entity (assigned and active) |
| `getPreviewRoute(int $id, int $storeId): string` | Storefront route used for preview URLs |
| `isStoreScoped(): bool` | Layout and switch differ per store view |

```xml
<type name="OOCS\LynxPageBuilder\Model\LayoutSources">
    <arguments>
        <argument name="sources" xsi:type="array">
            <item name="cms_page" xsi:type="object">Vendor\Module\Model\PageLayout</item>
        </argument>
    </arguments>
</type>
```

Built-ins: `cms_page` → `OOCS\LynxCms\Model\PageLayout`, `category` → `OOCS\LynxCatalog\Model\CategoryLayout`.
The built-in layout tools accept only these two types; anything else is reachable from your own tools through
`OOCS\LynxPageBuilder\Model\LayoutSources::get('<type>')`.

## Server facts worth knowing

| Item | Value |
|---|---|
| Start | `php -d opcache.enable_cli=1 bin/magento lynx:mcp:serve --host=127.0.0.1 --port=7361 [--max-children=8] [--timeout=120]` |
| Needs | PHP `pcntl` + `posix`; module `OOCS_LynxPageBuilder` |
| Transport | Streamable HTTP, stateless: one JSON response per `POST /mcp`, no SSE, no session id |
| Limits | 16 KB headers, 1 MB body; `--timeout` seconds per request, then the worker is killed and logged |
| Files | PID/status `var/lynx_mcp/`, server log `var/log/lynx_mcp_server.log`, tool errors in `var/log/` |
| Config | **Stores › Configuration › Lynx › MCP Server** (default scope); restart after changing it |
| Tool list CLI | `bin/magento lynx:mcp:tools` (`--details`, `--json`); shows every registered tool regardless of role |
| Built-in tools | `pagebuilder_read_{designs,pages,components,layout}`, `pagebuilder_create_{design,preview_url,share_preview_url}`, `pagebuilder_save_layout`, `catalog_search_{products,categories}`, `cms_search_pages`, `media_upload_image` |

Access cannot be narrowed per client: every client an admin approves can call every tool of that admin's role.
For assistants that should do less, create a dedicated admin with a restricted role.
