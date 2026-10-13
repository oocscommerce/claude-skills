# Starting layout for a page

Layouts are rows of `lynx_pagebuilder_layout` (`design_id`, `path`, `value` = JSON list of nodes), unique per design
and path. Lynx installs its own pages into the default design (`OOCS\LynxPageBuilder\Model\Design::DEFAULT_ID`), which
is read-only in the editor; merchants duplicate it, and a design inherits every path it has not saved from its parents.

## Data patch (insert only)

`Setup/Patch/Data/InstallFaqLayout.php`:

```php
<?php
declare(strict_types=1);

namespace Vendor\Faq\Setup\Patch\Data;

use Magento\Framework\Serialize\Serializer\Json;
use Magento\Framework\Setup\ModuleDataSetupInterface;
use Magento\Framework\Setup\Patch\DataPatchInterface;
use OOCS\LynxPageBuilder\Model\Design;
use OOCS\LynxPageBuilder\Setup\Patch\Data\CreateDefaultDesign;

class InstallFaqLayout implements DataPatchInterface
{
    private const PATH = 'faq_index_index';

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
     * Write the page's layout to the default design unless a row is already there
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
     * A constrained group holding the page heading
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
                'name'      => 'default_heading',
                'component' => 'OOCS_LynxThemeComponent/Components/Heading',
                'label'     => 'Heading',
                'visible'   => true,
                'props'     => ['text' => ['default' => 'Frequently Asked Questions'], 'level' => 'h1'],
            ]]],
        ]];
    }
}
```

The same check-then-insert is what `OOCS\LynxCatalog\Setup\Patch\Data\AddProductAlertLayouts` does for a page it adds.
Lynx's own install patches use `insertOnDuplicate(..., ['value'])` because they own those paths; a third-party module
should not overwrite a row someone else may have written.

## Node shape

| Key | Meaning |
|---|---|
| `name` | The component's key in the `components` registry (`default_group`, `vendor_faq_list`) |
| `component` | `Module_Name/Path` with a slash, e.g. `OOCS_LynxThemeComponent/Components/Group` (the registry entry writes `::`, the node writes `/`) |
| `label` | Shown in the editor's tree |
| `visible` | `true` |
| `props` | Saved settings; leave a setting out to get its default. `*_i18n` settings are objects keyed by locale with a `default` key |
| `slots` | For components with slots: slot name → list of nodes |

No `id` on stored nodes; ids are assigned at render. Useful built-ins: `default_group` (`OOCS_LynxThemeComponent/Components/Group`;
`layout` constrained/full, `width` narrow/default/site, `padding_y` none/sm/md/lg, `padding_x` default/none, `gap`
none/sm/md/lg), `default_heading` (`text` text_i18n, `level` h1–h6). To see what a built-in page stores, read the
layout patches under `vendor/oocscommerce/module-lynx-*/Setup/Patch/Data/`.

## After the fact

- A patch runs once per install. To change the starting layout on stores that already ran yours, add a new patch class
  (depending on the first) that updates only the default design's row; merchant designs that saved the path keep
  theirs.
- Writing rows outside the editor: clean the `lynx_component` cache type (layouts are cached there).

## Supplying the content zone from code

To build the content yourself before the design is read (as Lynx does for CMS pages that keep their layout on the
entity), observe `lynx_pagebuilder_layout_resolve_before` in `etc/frontend/events.xml`. Zones already set on the
transport are not read from the design:

```php
<?php
declare(strict_types=1);

namespace Vendor\Faq\Observer;

use Magento\Framework\Event\Observer;
use Magento\Framework\Event\ObserverInterface;

class FaqContent implements ObserverInterface
{
    /**
     * Put the FAQ nodes in the content zone
     *
     * @param Observer $observer
     * @return void
     */
    public function execute(Observer $observer): void
    {
        if ($observer->getData('action') !== 'faq_index_index') {
            return;
        }

        $transport = $observer->getData('transport');
        $layout = $transport->getData('layout');
        $layout['content'] = [/* nodes */];
        $transport->setData('layout', $layout);
    }
}
```

`lynx_pagebuilder_layout_resolve_after` fires with the same arguments once every zone is read, for code that needs to
see or adjust the final layout.
