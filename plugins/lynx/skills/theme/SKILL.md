---
name: theme
description: Creates a child theme of Lynx Vue or Lynx React and customizes the Lynx storefront from it - overriding a module's component, store or util file-for-file (`<theme>/<Vendor_Module>/lynx/<vue|react>/src/<path>`), the theme's own Vite workspace (`lynx/package.json`, `vite.config.js`, `src/main.ts|main.tsx`, `ssr.ts`, `ssr-worker.ts`, `styles.css`), fonts and Tailwind tokens, image sizes, and building/assigning the theme. Explains what belongs in design settings versus the theme, and the `l-` prefixed daisyUI classes the build includes. Use when a developer says "create a Lynx child theme", "override a Lynx component", "change the Lynx product page markup", "replace Lynx header from my theme", "change the Lynx font", "custom Tailwind tokens on Lynx", or "my override is not picked up". Use with the basics skill.
---
# Lynx child theme and overrides

Produces a Magento theme that inherits from `OOCS/LynxVue` or `OOCS/LynxReact`, builds its own bundle, and replaces
selected module files (components, stores, utils) without touching the Lynx packages.

## Before you start

- Read **basics** (module layout, styling rules, CLI).
- Find the Lynx packages (`vendor/oocscommerce/`) and the parent you extend:
  `theme-frontend-lynx-vue` (`OOCS/LynxVue`) or `theme-frontend-lynx-react` (`OOCS/LynxReact`). A store that runs both
  needs two child themes, one per framework.
- `bin/magento lynx:workspace:setup --list` shows the Lynx themes; the root `package.json` `lynx.themes` shows those
  already built. Keep them all when you rerun the command.
- Before overriding, look for an extension point: slots of the built-in components (product, category, cart,
  checkout), an `after` plugin on the provider's `resolve()`, or a new component (skill component). Override only
  to change markup or behavior the component does not expose.
- Brand colors, corners, borders, widths and heading scale are **design settings** the merchant sets in the editor;
  they win over anything in the theme. The theme holds what designs do not set: font, type scale, extra tokens,
  global CSS, component overrides.

## Steps

### 1. Register the theme

`app/design/frontend/Vendor/storefront/registration.php`:

```php
<?php
declare(strict_types=1);

use Magento\Framework\Component\ComponentRegistrar;

ComponentRegistrar::register(ComponentRegistrar::THEME, 'frontend/Vendor/storefront', __DIR__);
```

`theme.xml`:

```xml
<?xml version="1.0"?>
<theme xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:noNamespaceSchemaLocation="urn:magento:framework:Config/etc/theme.xsd">
    <title>Vendor Storefront</title>
    <parent>OOCS/LynxVue</parent>
</theme>
```

`composer.json` (when installed as a package; `require` keeps the parent installed; React parent:
`oocscommerce/theme-frontend-lynx-react`):

```json
{
    "name": "vendor/theme-frontend-storefront",
    "type": "magento2-theme",
    "require": {
        "oocscommerce/theme-frontend-lynx-vue": "^1.0"
    },
    "autoload": {
        "files": ["registration.php"]
    }
}
```

### 2. Copy the Vite workspace from the parent

The child builds its own bundle, so it needs its own `lynx/` folder:

```bash
T=app/design/frontend/Vendor/storefront
P=vendor/oocscommerce/theme-frontend-lynx-vue
mkdir -p $T/lynx/src $T/web
cp $P/lynx/package.json $P/lynx/vite.config.js $P/lynx/tsconfig.json $P/lynx/.gitignore $T/lynx/
cp $P/lynx/src/main.ts $P/lynx/src/ssr.ts $P/lynx/src/ssr-worker.ts $P/lynx/src/styles.css $T/lynx/src/
cp $P/web/.gitignore $T/web/
# React parent: main.tsx instead of main.ts
# Linting: also eslint.config.mjs. SSR on a Cloudflare Worker: also wrangler.toml (rename the Worker)
```

Edit `$T/lynx/package.json`: give it a **unique `name`** (`"name": "vendor-storefront"`); the parents use `oocs-lynx`
and `oocs-lynx-react`, and npm workspaces cannot share a name. Keep the scripts and the `@oocs/lynx-vue` /
`@oocs/lynx-react` dev dependency (the build kit, shipped inside `OOCS_LynxTheme`).

`vite.config.js` works unchanged: it finds the project root, enabled modules and the parent chain from the root
`package.json`. Do not copy `inertia.phtml`, `lynx_default.xml`, `root.phtml` or `etc/view.xml`: Magento theme
fallback inherits them, and `inertia.phtml` loads `js/lynx/main.min.js` from the active theme, i.e. your build.

Files under `lynx/src/` are entry points only (`main.ts`/`main.tsx`, `ssr.ts`, `ssr-worker.ts`, `styles.css`). Do not
add components or logic there; override module files instead (step 4). If you change `main.*`, keep its hydration
behavior (skill ssr).

### 3. Set up the workspace

```bash
bin/magento setup:upgrade
bin/magento lynx:workspace:setup OOCS_LynxVue Vendor_storefront   # name every theme you build
```

The theme name is the code with `_` for `/`. The command writes the theme and its parents into the root
`package.json` (`lynx.themes`) and runs `npm install`. Without it the build stops with "package.json … does not list
<theme dir>".

### 4. Override a module file

```
Module file:    <module>/view/frontend/lynx/<vue|react>/src/<path>
Theme override: <theme>/<Vendor_Module>/lynx/<vue|react>/src/<path>
```

Example: change the simple/virtual product page (`OOCS_LynxCatalog::Components/Product/Simple`):

```bash
mkdir -p $T/OOCS_LynxCatalog/lynx/vue/src/Components/Product
cp vendor/oocscommerce/module-lynx-catalog/view/frontend/lynx/vue/src/Components/Product/Simple.vue \
   $T/OOCS_LynxCatalog/lynx/vue/src/Components/Product/
# React theme: lynx/react/src/…/Simple.tsx
```

Then edit the copy. Keep its props and the slots it renders (`price_after`, `product.info.additional`,
`info_after`): the provider still sends the same props and merchants may have placed components in those slots.
Treat `urls.add_to_wishlist` and `urls.add_to_compare` as optional and render their buttons only when present: the
provider sends them only while the node's `show_wishlist` / `show_compare` is on and Magento allows it. The gallery
is `Components/Product/Media` (slot `gallery_after`), the breadcrumbs `Components/Product/Page`.

Stores and utils work the same way: `$T/OOCS_LynxTheme/lynx/vue/src/stores/global/messagesStore.ts` replaces the store
for every importer. More path examples and the resolution order: [reference/overrides.md](reference/overrides.md).

To show data the provider does not send, add an `after` plugin on the provider's `resolve()` in a module (providers
are created through the object manager, so plugins run), declare the new prop in the module's `.graphqls` (the server
drops undeclared props), then read it in the override (`Lynx.<TypeName>` has it after `lynx:static:deploy`):

```graphql
# Vendor_Module etc/components/Simple.graphqls (<sequence> OOCS_LynxCatalog)
extend type ProductDetailSimple {
  delivery_note: String! @optional
}
```

```xml
<!-- Vendor_Module etc/di.xml -->
<type name="OOCS\LynxCatalog\Provider\Product\Detail\Simple">
    <plugin name="vendor_module_delivery_note" type="Vendor\Module\Plugin\AddDeliveryNote"/>
</type>
```

```php
<?php
declare(strict_types=1);

namespace Vendor\Module\Plugin;

use OOCS\LynxCatalog\Provider\Product\Detail\Simple;

class AddDeliveryNote
{
    /**
     * Add a translated delivery note to the product detail props
     *
     * @param Simple $subject
     * @param array $result
     * @param array $props
     * @return array
     */
    public function afterResolve(Simple $subject, array $result, array $props): array
    {
        $result['delivery_note'] = (string) __('Ships within 24 hours');

        return $result;
    }
}
```

### 5. Styles, tokens, fonts, image sizes

`lynx/src/styles.css` is the Tailwind entry. Keep its first lines (they import Tailwind, the shared
`OOCS_LynxTheme/css/lynx.css` with the daisyUI setup and design tokens, the theme's own `@source`, and
`../dist/sources.css` with every module's `@source`). Below them, change tokens in `@theme`:

```css
@theme {
  --font-sans: "Inter", ui-sans-serif, system-ui, sans-serif;
  /* Fallback for pages without a design; a design's setting wins */
  --radius-button: 9999px;
}
```

- **Font**: the parent self-hosts Mulish via `@fontsource/mulish`. Add your font package as a dependency of the
  theme's `lynx/package.json` (pin the version), replace the `@font-face` rules (reference files by package name,
  e.g. `url("@fontsource/inter/files/inter-latin-400-normal.woff2")`, never a `../node_modules` path) and
  `--font-sans`, and replace the preloads in `$T/Magento_Theme/layout/lynx_default.xml`. Full example:
  [reference/fonts-and-styles.md](reference/fonts-and-styles.md).
- **Module CSS**: a module can ship `view/frontend/lynx/css/x.css`; import it in `styles.css` with
  `@import "Vendor_Module/css/x.css";`. CSS files are not overridden file by file.
- **daisyUI**: `lynx.css` configures daisyUI 5 with `prefix: l-` and `include: button, input, select, textarea,
  checkbox, radio, alert, card, table, skeleton`. Only those `l-*` classes produce CSS; do not add a second
  `@plugin "daisyui"`. Build other widgets with Tailwind and the tokens (basics).
- **Image sizes**: `$T/etc/view.xml` with only the sizes you change; keep the image ids Lynx components use
  (`category_page_grid`, `product_page_image_large`…). The rest comes from Lynx Blank.
- **Magento templates** (`Magento_Theme/templates/root.phtml`…): copy from Lynx Blank into the same path and edit.

### 6. Build, assign, deploy

```bash
bin/magento lynx:static:deploy Vendor_storefront     # or: cd $T/lynx && npm run build
bin/magento setup:static-content:deploy               # production mode, always after the Lynx build
bin/magento cache:clean
```

Assign in **Content › Design › Configuration** › store view › **Applied Theme**, then clean the cache. Designs are not
tied to a theme: the store view keeps its Active Design; switching themes only changes the code that draws the
layouts. A new override file needs no `lynx:workspace:setup`, only a rebuild.

## Rules

- **An override replaces the whole file.** A relative import of the module's original from your override resolves to
  your override; copy the file and change the copy.
- **Keep the contract**: props, slot names, and every export other files import (stores/utils). Type checking catches
  a missing export or changed type, not changed behavior.
- **Vue node components keep one root element**: the editor's preview markers and the `align` attribute
  (`data-align`) fall through onto it. React components need nothing on the root (`align` is applied by a wrapper).
- **Override per framework**: a Vue theme reads only `lynx/vue/src` overrides, a React theme only `lynx/react/src`.
- **Overrides of a disabled module, or one missing from `lynx.modules`, are skipped.** The override folder name must
  be exactly the module name (`Vendor_Module`).
- **Stay SSR-safe** (skill ssr): no `window`/`document` during setup or render; on React never resolve a
  component lazily during render.
- **Styling rules still apply**: design tokens and `l-` classes, sizes from `text-sm` up, no hardcoded colors,
  radius, borders, shadows or widths (basics). A hardcoded brand color in a theme ignores the merchant's design.
- **Labels** in an override come from props (`labels`) or `config('labels')`, never hardcoded text.
- **Recheck overrides after `composer update`**: compare each copy with the new original.
- Never put sources in `web/`: it holds build output only (`web/js/lynx/`, `web/css/lynx/`).

## Check

1. `cd $T/lynx && npm run type-check`, then `npm run build`.
2. The build log lists every chunk with its source; your override path must appear:
   ```
   OOCS_LynxCatalog/Components/Product/Simple.min.js  ←  app/design/frontend/Vendor/storefront/OOCS_LynxCatalog/lynx/vue/src/Components/Product/Simple.vue
   ```
3. Lint with the theme's `eslint.config.mjs` if copied.
4. Open the page on the store view running the theme; open it in the PageBuilder preview and change Button color,
   Corners, Border width: the override must follow.
5. If SSR is used: rebuild the SSR bundle from the child theme's `lynx/` and restart the renderer (skill ssr).

## Reference

- [reference/overrides.md](reference/overrides.md): resolution order, path table, React node contract, porting notes
- [reference/fonts-and-styles.md](reference/fonts-and-styles.md): font replacement, `styles.css` layout, tokens
- https://docs.oocscommerce.com/lynx/developer/themes, https://docs.oocscommerce.com/lynx/developer/frontend
