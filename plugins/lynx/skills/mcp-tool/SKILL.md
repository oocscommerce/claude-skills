---
name: mcp-tool
description: Adds a tool (and optionally an HTTP route or a layout source) to the Lynx MCP server (module OOCS_LynxMcp, bin/magento lynx:mcp:serve) from a third-party Magento module, so AI assistants connected over MCP can read or change that module's data. Covers the OOCS\LynxMcp\Api\ToolInterface class in Tool/<Object>/<VerbObject>.php, tool naming, registration in OOCS\LynxMcp\Model\ToolList in global etc/di.xml, the per-tool ACL resource under OOCS_LynxMcp::tools (permissions follow the approving admin's role), LocalizedException errors, store scope, the forked-worker server constraints (restart after changes), checking with bin/magento lynx:mcp:tools --details, and RouteInterface routes in RouteList. Use when a developer says "add an MCP tool", "expose my module to Claude via Lynx MCP", "new tool for lynx:mcp:serve", "MCP route", "assistant cannot see my tool". Use with the basics skill.
---

# Lynx MCP Tool

Produces an MCP tool in `Vendor_Module`: one class implementing `OOCS\LynxMcp\Api\ToolInterface`, one ACL resource,
one `di.xml` line. The tool runs Magento code directly (no REST/GraphQL) inside the Lynx MCP server and is visible
only to clients approved by an admin whose role holds its ACL resource.

## Before you start

- Read `basics`. `OOCS_LynxMcp` comes with `oocscommerce/lynx` (package `oocscommerce/module-lynx-mcp`; depends on
  `OOCS_LynxPageBuilder`).
- Look at the built-in tools under `vendor/oocscommerce/module-lynx-mcp/Tool/`:
  `Products/SearchProducts.php` (read-only, simple), `Layout/SaveLayout.php` (writes), plus `etc/di.xml` and
  `etc/acl.xml` of that module.
- Run `bin/magento lynx:mcp:tools` to see existing names (avoid collisions).
- Decide: tool name `<group>_<verb>_<object>` (e.g. `cms_read_block`), whether it changes data, whether it works per
  store view.

## Steps

### 1. `etc/module.xml`

```xml
<module name="Vendor_Module">
    <sequence>
        <module name="OOCS_LynxMcp"/>
    </sequence>
</module>
```

`composer.json`: require `oocscommerce/module-lynx-mcp`.

### 2. The tool class — `Tool/<Object>/<VerbObject>.php`

`OOCS\LynxMcp\Api\ToolInterface` has exactly five methods. The tool **name** is not a method: it is the item name in
`di.xml`. The only annotation sent to clients is `readOnlyHint`, taken from `isReadOnly()`.

| Method | Returns |
|---|---|
| `getDescription(): string` | What it does and returns, written for the model deciding whether to call it |
| `getInputSchema(): array` | JSON Schema, `type` `object` at the root |
| `getAclResource(): string` | ACL id the caller's role must hold: `Vendor_Module::tool_<tool name>` |
| `isReadOnly(): bool` | `false` if it changes data (sent as `readOnlyHint`, marked "changes data" on the approval screen) |
| `execute(array $arguments): array` | String-keyed array; sent as `structuredContent` and as JSON text |

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Tool\Blocks;

use Magento\Cms\Api\GetBlockByIdentifierInterface;
use Magento\Framework\Exception\LocalizedException;
use Magento\Framework\Exception\NoSuchEntityException;
use Magento\Store\Model\Store;
use Magento\Store\Model\StoreManagerInterface;
use OOCS\LynxMcp\Api\ToolInterface;

class ReadBlock implements ToolInterface
{
    /**
     * @param GetBlockByIdentifierInterface $getBlockByIdentifier
     * @param StoreManagerInterface $storeManager
     */
    public function __construct(
        private readonly GetBlockByIdentifierInterface $getBlockByIdentifier,
        private readonly StoreManagerInterface $storeManager,
    ) {
    }

    public function getDescription(): string
    {
        return 'Read a CMS block by identifier: id, title, whether it is active, and its HTML content.';
    }

    public function getInputSchema(): array
    {
        return [
            'type'       => 'object',
            'properties' => [
                'identifier' => ['type' => 'string', 'description' => 'Block identifier'],
                'store'      => ['type' => 'string', 'description' => 'Store view code; default all store views'],
            ],
            'required'   => ['identifier'],
        ];
    }

    public function getAclResource(): string
    {
        return 'Vendor_Module::tool_cms_read_block';
    }

    public function isReadOnly(): bool
    {
        return true;
    }

    public function execute(array $arguments): array
    {
        $identifier = trim((string) ($arguments['identifier'] ?? ''));
        if ($identifier === '') {
            throw new LocalizedException(__('identifier must not be empty.'));
        }
        try {
            $storeId = isset($arguments['store'])
                ? (int) $this->storeManager->getStore((string) $arguments['store'])->getId()
                : Store::DEFAULT_STORE_ID;
            $block = $this->getBlockByIdentifier->execute($identifier, $storeId);
        } catch (NoSuchEntityException) {
            throw new LocalizedException(__('No block or store view matches.'));
        }

        return [
            'id'        => (int) $block->getId(),
            'title'     => (string) $block->getTitle(),
            'is_active' => (bool) $block->isActive(),
            'content'   => (string) $block->getContent(),
        ];
    }
}
```

A tool that writes: [reference/examples.md](reference/examples.md#a-tool-that-writes).

### 3. ACL resource — `etc/acl.xml`

Place it under **Tools** (`Magento_Backend::admin › OOCS_LynxPageBuilder::manage › OOCS_LynxMcp::mcp ›
OOCS_LynxMcp::tools`):

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:Acl/etc/acl.xsd">
    <acl>
        <resources>
            <resource id="Magento_Backend::admin">
                <resource id="OOCS_LynxPageBuilder::manage">
                    <resource id="OOCS_LynxMcp::mcp">
                        <resource id="OOCS_LynxMcp::tools">
                            <resource id="Vendor_Module::tool_cms_read_block" title="Read CMS Block" translate="title"/>
                        </resource>
                    </resource>
                </resource>
            </resource>
        </resources>
    </acl>
</config>
```

### 4. Register — global `etc/di.xml`

```xml
<?xml version="1.0"?>
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:noNamespaceSchemaLocation="urn:magento:framework:ObjectManager/etc/config.xsd">
    <type name="OOCS\LynxMcp\Model\ToolList">
        <arguments>
            <argument name="tools" xsi:type="array">
                <item name="cms_read_block" xsi:type="object">Vendor\Module\Tool\Blocks\ReadBlock\Proxy</item>
            </argument>
        </arguments>
    </type>
</config>
```

- Item name = tool name, must match `^[a-zA-Z0-9_-]{1,64}$`; a later module using the same name replaces the tool.
- Always `\Proxy`: `ToolList` is built in the master process at server start; without a proxy your tool and its
  dependencies are constructed there, before fork, and listing tools builds every tool.
- `ToolList` throws `InvalidArgumentException` for a bad name or a class not implementing `ToolInterface`; the server
  (and `lynx:mcp:tools`) fail to start.

### 5. Activate

```bash
bin/magento setup:upgrade            # ACL + DI (setup:di:compile in production)
bin/magento cache:clean
bin/magento lynx:mcp:tools --details # name, read/write, ACL, full description and schema (--json for all)
# restart the server process (systemd unit, or Ctrl+C and run again):
php -d opcache.enable_cli=1 bin/magento lynx:mcp:serve --host=127.0.0.1 --port=7361
```

Grant the resource in **System › Permissions › User Roles › Role Resources** to the roles that should use it
(roles with **All** already have it). Connected assistants see the tool on their next `tools/list`; no reconnect.

## Rules

- **Permissions = the approving admin's role.** Every Bearer token resolves to an ACL role (OAuth token → admin who
  approved it; admin token → that admin; integration token → integration resources). Checked on every request.
  `tools/list` hides tools the role lacks; calling one answers "Unknown tool". OAuth scopes are ignored.
- **Global DI only.** The server runs in area `global` and loads only `etc/di.xml`. Never register tools, routes, or
  any pool your tool reads in `etc/adminhtml/di.xml` / `etc/frontend/di.xml`: they are not loaded, and an area
  declaration of the same array argument replaces the whole list.
- **Errors.** Throw `Magento\Framework\Exception\LocalizedException` with a message the assistant can act on (what
  was wrong, valid values). Any other `Throwable` is logged to `var/log/` and the assistant only sees "The tool
  failed. See the server log." Do not echo secrets in messages.
- **Validate arguments yourself.** The server does not check them against the schema; cast, trim, clamp limits.
- **Store scope.** Each request starts at store 0 (admin, All Store Views). Per-store-view tools take a `store`
  argument. `OOCS\LynxMcp\Helper\PageBuilderFinder::findStore($value)` accepts a code or a domain/URL the store view
  serves (empty → default store view) and throws a `LocalizedException` listing valid codes. Use
  `Magento\Store\Model\App\Emulation` when you need frontend URLs/config of that store.
- **Long-running server, forked workers.** Workerman master boots Magento once, then forks a worker per connection;
  the worker serves one request and exits. Consequences:
  - Config, DI and code are read at start: **restart `lynx:mcp:serve` after any code, DI, ACL-tree or Lynx MCP
    config change**, and after `setup:upgrade`.
  - Do no work in constructors (no DB reads, no config caching): anything built in the master is frozen from boot.
  - In-memory caches/statics live for one request only; persist what must survive (DB, cache).
  - Each worker opens its own DB/Redis connections; do not hold connections or file handles across calls.
  - A request over `--timeout` (default 120 s) is killed: wrap multi-step writes in a DB transaction, keep tools fast,
    paginate (cap `limit`).
  - Request body max 1 MB (base64 uploads count).
- **Output**: plain data (ids, codes, paths, booleans); media paths relative to `pub/media` as image settings take
  them; counts with lists (`total`, `items`). Return `[]` for "done" rather than prose.
- `getDescription()` says what the tool returns and when to use it; mention related tools by name.
- `isReadOnly()` must be honest: admins see write tools marked on the approval screen.

## Optional: an HTTP route

`OOCS\LynxMcp\Api\RouteInterface`: `getMethods(): array` (others get 405 with `Allow`) and
`handle(array $request): array` receiving `method`, `path`, `query`, `headers` (lower-case), `body`, returning
`[status, headers, body]` (array body → JSON). Register in global `etc/di.xml` on `OOCS\LynxMcp\Model\RouteList`
argument `routes`, item name = exact path starting with `/` (no parameters). Routes are **not authenticated** by the
server: check `Authorization` yourself if they expose data. Reusing a built-in path (`/mcp`, `/token`, `/authorize`,
`/authorize/status`, `/revoke`, `/register`, `/.well-known/…`) replaces it. Example:
[reference/examples.md](reference/examples.md#an-http-route).

## Optional: a layout source

Storing CMS page or category layouts elsewhere: implement `OOCS\LynxPageBuilder\Api\LayoutSourceInterface` and
register it under `cms_page` or `category` in `OOCS\LynxPageBuilder\Model\LayoutSources` → `sources` (global
`etc/di.xml`, sequence after `OOCS_LynxCms` / `OOCS_LynxCatalog`). Other types are readable only from your own tools
via `LayoutSources::get('<type>')`. See [reference/examples.md](reference/examples.md#a-layout-source).

## Check

- `bin/magento lynx:mcp:tools` lists the tool with the right ACL and read/write flag (this list is not filtered by
  role).
- Restart the server; with an admin whose role lacks the resource the tool is absent from `tools/list`; after
  granting it, it appears on the next call.
- Call it from a client (e.g. `claude mcp add --transport http shop https://mcp.example.com/mcp`): valid input returns
  data; bad input returns your `LocalizedException` message with `isError: true`.
- Exceptions other than `LocalizedException` appear in `var/log/`; the server log is `var/log/lynx_mcp_server.log`.
- Write tools: a failure halfway leaves no partial data (transaction).

## Reference

- [reference/examples.md](reference/examples.md) — write tool, HTTP route, layout source, server facts
- https://docs.oocscommerce.com/lynx/developer/mcp
- Related: `extension-port` (rebuilding a design with an assistant over MCP)
