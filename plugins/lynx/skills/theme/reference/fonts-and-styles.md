# Fonts, styles.css and tokens

## `lynx/src/styles.css` layout (copied from the parent)

```css
@import "tailwindcss";
/* Shared by every Lynx theme: daisyUI setup (prefix l-), design tokens, frame grid */
@import "OOCS_LynxTheme/css/lynx.css";
@source "./**/*.{vue,ts}";
/* Module and override sources of this build, written by the Lynx Vite config */
@import "../dist/sources.css";

/* @font-face rules … */

@theme {
  /* the theme's tokens, defaults commented out */
}
```

Keep the four lines at the top. Never add `@source` paths that count `../` to a module folder: the depth differs
between `app/code`, `vendor/` and path repositories, and a wrong path still builds, just without most of the CSS.

## What goes where

| Want | Where |
|---|---|
| Brand palette, corners, border width, control height, widths, header height, heading scale, depth | Design settings in the PageBuilder editor (per design; they win over the theme) |
| Font family, `@font-face`, preloads | Theme `styles.css` + `Magento_Theme/layout/lynx_default.xml` |
| Fallback values for pages without a design | `@theme` in `styles.css` (e.g. `--radius-button`) |
| Extra tokens (a new shadow, a breakpoint) | `@theme` in `styles.css` |
| Global CSS (prose styles, a utility) | `styles.css`, or a module's `view/frontend/lynx/css/*.css` imported with `@import "Vendor_Module/css/file.css";` |
| Component markup and classes | Override the component file |

Tokens defined by Lynx (`lynx.css`): `--color-*` palette and `--radius-box/field/selector`, `--border`, `--size-field`,
`--size-selector`, `--depth`, `--heading-scale` in `@layer base` (written by design settings); in `@theme`: the text
scale (`text-lg`+ multiplied by `--heading-scale`), `--color-link`, `--radius-button`, `--shadow-thin/menu/dialog/card`,
`--width-site/content/narrow` (`max-w-site`…), `--spacing-header` (`min-h-header`), `--breakpoint-xs` (480px).

## Replacing the font (example: Inter)

1. Dependency of the theme, pinned (`$T/lynx/package.json`):
   ```json
   "dependencies": { "@fontsource/inter": "<exact latest version>" }
   ```
   Install it with `npm install @fontsource/inter@<version>` in `$T/lynx/` (writes the line above); commit
   `package.json`. Remove `@fontsource/mulish` if unused.
2. In `styles.css`, replace the Mulish `@font-face` blocks. Reference files by package name; the build copies them
   into `web/js/lynx/` under their own names:
   ```css
   @font-face {
     font-family: "Inter";
     font-style: normal;
     font-display: swap;
     font-weight: 400;
     src: url("@fontsource/inter/files/inter-latin-400-normal.woff2") format("woff2");
   }
   @theme {
     --font-sans: "Inter", ui-sans-serif, system-ui, sans-serif;
   }
   ```
   Copy the `unicode-range` values from the package's `<weight>.css` when you ship several subsets.
3. Replace the parent's preloads, `$T/Magento_Theme/layout/lynx_default.xml`:
   ```xml
   <?xml version="1.0"?>
   <page xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="urn:magento:framework:View/Layout/etc/page_configuration.xsd">
       <head>
           <remove src="js/lynx/mulish-latin-400-normal.woff2"/>
           <remove src="js/lynx/mulish-latin-600-normal.woff2"/>
           <link src="js/lynx/inter-latin-400-normal.woff2" rel="preload" as="font" crossorigin="anonymous"/>
       </head>
   </page>
   ```
   Do not set `layout="…"` on `<page>` (basics).
4. Rebuild, then `bin/magento cache:clean layout full_page`.

## daisyUI

Configured once in `OOCS_LynxTheme/css/lynx.css`:

```css
@plugin "daisyui" {
  themes: false;
  prefix: l-;
  include: button, input, select, textarea, checkbox, radio, alert, card, table, skeleton,
    collapse, badge;
}
```

Use `l-btn`, `l-input`, `l-select`, `l-textarea`, `l-checkbox`, `l-radio`, `l-alert`, `l-card`, `l-table`,
`l-skeleton`, `l-collapse`, `l-badge` and their modifiers; any other `l-*` class produces no CSS. `hover:` applies on every device (custom
variant in `lynx.css`), so hover-driven menus also open on touch devices in desktop view.
