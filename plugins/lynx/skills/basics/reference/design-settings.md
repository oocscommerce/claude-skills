# Design settings → CSS variables → classes

Each design setting (PageBuilder editor › design settings) writes one CSS variable on `:root` of the storefront. A
component follows the setting only through classes that read that variable. Defaults live in
`OOCS_LynxTheme/css/lynx.css` (module `module-lynx-theme`, `view/frontend/lynx/css/lynx.css`); the settings are
declared in `OOCS_LynxPageBuilder`'s `etc/di.xml`.

| Design setting | CSS variable | Classes that follow it |
|---|---|---|
| Colors › Button, Button text | `--color-primary`, `--color-primary-content` | `bg-primary`, `text-primary`, `border-primary`, `text-primary-content`; `l-btn-primary`, `l-checkbox-primary`, `l-radio-primary`; focus rings of `l-btn` and fields |
| Colors › Secondary button, Secondary button text | `--color-secondary`, `--color-secondary-content` | `bg-secondary`, `text-secondary-content`, `l-btn-secondary` |
| Colors › Link | `--color-link` | `text-link`, `hover:text-link` |
| Colors › Text | `--color-base-content` | `text-base-content` and its opacities (`/70`, `/60`); the page's default text |
| Colors › Page background | `--color-base-100` | `bg-base-100`; background of `l-input`, `l-select`, `l-textarea`, `l-card` |
| Colors › Surface | `--color-base-200` | `bg-base-200` |
| Colors › Border | `--color-base-300` | `border-base-300`, `divide-base-300`; border of `l-input`, `l-select`, `l-textarea` |
| Colors › Dark surface, Dark surface text | `--color-neutral`, `--color-neutral-content` | `bg-neutral`, `text-neutral-content` |
| Colors › Info, Success, Warning, Error (and their text) | `--color-info` … `--color-error-content` | `bg-success`, `text-error`, `border-warning`…, `l-alert`, `l-input-error` |
| Typography › Body font scale | root `font-size` | every `rem` size: the `text-*` scale, spacing and widths in `rem` |
| Typography › Heading font scale | `--heading-scale` | `text-lg` to `text-5xl` |
| Layout › Site, Content, Narrow width | `--width-site`, `--width-content`, `--width-narrow` | `max-w-site`, `max-w-content`, `max-w-narrow`; the frame of `Group` and `align` wide/full |
| Layout › Header height | `--spacing-header` | `min-h-header`, `h-header` |
| Shape › Corners — boxes | `--radius-box` | `rounded-box`; `l-card`, `l-alert` |
| Shape › Corners — buttons | `--radius-button` | `rounded-button`; `l-btn` |
| Shape › Corners — fields | `--radius-field` | `rounded-field`; `l-input`, `l-select`, `l-textarea` |
| Shape › Corners — small controls | `--radius-selector` | `rounded-selector`; `l-checkbox` (radios and `l-badge` stay round) |
| Shape › Border width | `--border` | `border-(length:--border)`; borders of `l-btn`, `l-input`, `l-select`, `l-textarea`, `l-checkbox`, `l-radio` |
| Shape › Control height | `--size-field` | height of `l-btn`, `l-input`, `l-select` |
| Shape › Small control size | `--size-selector` | size of `l-checkbox`, `l-radio` |
| Shape › Depth | `--depth` | `shadow-card`, `shadow-dialog` |

`accent` exists as a token (`bg-accent text-accent-content`) and defaults to the primary color; no setting writes it.

## Token classes by use

| Use | Classes |
|---|---|
| Text | `text-base-content`, muted `text-base-content/70`, `text-base-content/60` |
| Backgrounds | `bg-base-100` (page, cards, fields), `bg-base-200` (sunken panels), `bg-base-300` |
| Borders | `border-base-300`, width `border-(length:--border)` |
| Brand colors | `bg-primary text-primary-content`, `secondary`, `accent`, `neutral`, each with `-content` |
| States | `info`, `success`, `warning`, `error` (with `-content`) |
| Links | `text-link`, `hover:text-link`; keep `text-primary` for icons and states |
| Corners | `rounded-box` (cards, panels), `rounded-field` (inputs), `rounded-selector` (checkbox, badges), `rounded-button` |
| Shadows | `shadow-card`, `shadow-menu`, `shadow-dialog`, `shadow-thin` |
| Widths | `max-w-site`, `max-w-content`, `max-w-narrow`; header `min-h-header` |

Lay text on a color with its `-content` partner (`bg-primary text-primary-content`) so it stays readable when the
merchant changes the palette.

## daisyUI controls (prefix `l-`)

Built components only: button, input, select, textarea, checkbox, radio, alert, card, table, skeleton, collapse, badge
(`@plugin "daisyui" { prefix: l-; include: … }` in `lynx.css`).

| Control | Classes |
|---|---|
| Button | `l-btn l-btn-primary`; secondary `l-btn l-btn-outline l-btn-secondary`; sizes `l-btn-sm`, `l-btn-lg` |
| Text field | `l-input`, with an error `l-input-error` |
| Select, textarea | `l-select`, `l-textarea` |
| Checkbox, radio | `l-checkbox l-checkbox-primary`, `l-radio l-radio-primary` |
| Card, table | `l-card l-card-border`, `l-table` |
| Loading placeholder | `l-skeleton` |
| Inline notice | `l-alert` |

Any other daisyUI class (`l-modal`, `l-menu`, `l-dropdown`, `l-badge`…) produces no CSS.

## What does not follow the settings

Tailwind's fixed palette and scales: `bg-blue-600`, `text-gray-500`, `rounded-lg`, `border-2`, `shadow-md`, `h-10` on
a button, `max-w-7xl`, `text-[15px]`. Use fixed values only for what must be identical in every design (icon size,
spacing inside the component).

## Font sizes

`text-sm` (14px), `text-base`, `text-lg` … `text-5xl`. `text-sm` is the floor for storefront text; `text-xs`/`text-2xs`
exist in the CSS but are not for storefront components; no arbitrary sizes.

## Quick check

Open the component in the editor preview and change Button color, Corners and Border width: everything that should
follow must change immediately.
