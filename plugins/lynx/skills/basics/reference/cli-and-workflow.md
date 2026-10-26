# CLI and day-to-day workflow

## Commands

| Command | Arguments and options | What it does |
|---|---|---|
| `lynx:theme:setup` | `[theme...]` themes to set up (none = every theme set up before); `--list` list Lynx themes and stop; `--npm=<binary>` (default `LYNX_NPM` env, then `npm` on `PATH`) | Writes `<theme>/lynx/.lynx/package.json` (generated meta-package `lynx-deps`: `dependencies` of the build kit and of every enabled module with `view/frontend/lynx/vue/package.json`; `lynx.modules`, `lynx.parents`), runs `npm install` in `<theme>/lynx/` when it changed, records it in `node_modules/.lynx-deps`, writes `<theme>/lynx/dist/props.d.ts` and `dist/extensions.json`. Two sources pinning one package to different versions stop it. |
| `lynx:static:deploy` | `[theme...]` (none = every theme set up); `--ssr` also `npm run build:ssr`; `--list`, `--npm` | Installs nothing: stops with "Not set up for the enabled modules and the kit" when the meta-package is stale. Writes `<theme>/lynx/dist/props.d.ts` (the `Lynx` types from `etc/components/*.graphqls`, also done by `lynx:theme:setup` and `lynx:static:test`), then `npm run build` in each theme's `lynx/` folder |
| `lynx:pagebuilder:build` | `--npm=<binary>` | `npm ci` + `npm run build` for the admin editor (`view/adminhtml/pagebuilder` of `OOCS_LynxPageBuilder` and of every enabled editor extension). The editor ships prebuilt: only after changing its sources, then commit the output |
| `lynx:compile` | `[type...]` (none = all; `--help` lists them) | Writes merged module XML, translations and the merged `etc/components/*.graphqls` (`components`) to `generated/lynx_compile` (needs `OOCS_LynxCompiler`) |
| `lynx:mcp:serve`, `lynx:mcp:tools` | see skill mcp-tool | MCP server (needs `OOCS_LynxMcp`) |

Theme argument = theme code with `_` for `/` (`OOCS_LynxVue`, `Vendor_storefront`). A theme is
buildable when it has `lynx/package.json`; that file declares `"workspaces": [".lynx"]`.
Each theme's `lynx/` is its own npm project with its own `node_modules`; there is no root `package.json`.
A theme-only package: `npm install <pkg>` in `<theme>/lynx/`, commit its `package.json`.

## Theme scripts (`<theme>/lynx/`)

| Script | Does |
|---|---|
| `npm run build` | `type-check`, then Vite → `../web/js/lynx/*.min.js`, `../web/css/lynx/styles.min.css` |
| `npm run type-check` | `lynx-tsconfig-vue && vue-tsc -p tsconfig.modules.json --noEmit` |
| `npm run dev` | Vite dev server (port 5173); picks up added/removed files |
| `npm run build:ssr` | Node SSR bundle → `dist/ssr/nodejs/server.mjs` |
| `npm run build:ssr:worker` | Cloudflare Worker bundle → `dist/ssr/worker/worker.mjs` |
| `npm run ssr` | Runs the Node renderer |
| `npm run lint` | ESLint (see below); `npm run lint -- --fix` fixes |
| `npm run format` / `format:check` | Prettier with the kit's config over the theme and each source's `lynx/vue` folder |

`tsconfig.modules.json` is generated (git-ignored): one `paths` entry per module, pointing to the theme's override
folder, then parent themes' override folders, then the module. A missing module or export is a type error.

## Lint

`npm run lint` in a theme's `lynx/` runs ESLint with the theme's `lynx/eslint.config.mjs` over the theme's `src/`,
its parent themes and every module in `lynx.modules`:

```bash
cd vendor/oocscommerce/theme-frontend-lynx-vue/lynx && npm run lint
```

Notable rules: no DOM globals at module scope (`ssr-friendly/…`), `no-eval`,
`@typescript-eslint/no-deprecated`, `no-console` except `warn`/`error`, `vue/no-mutating-props`,
`vue/require-explicit-emits`.

## What to run after a change

| You changed | Run |
|---|---|
| `.vue`, `.ts` | `npm run build` (or `bin/magento lynx:static:deploy`), reload. Production: also `setup:static-content:deploy` |
| `etc/components/*.graphqls` (components, settings, slots, props) | `bin/magento cache:clean lynx_component full_page`, plus `lynx:compile components` if used; `lynx:static:deploy` to regenerate the `Lynx.*` types |
| `di.xml` (providers' dependencies, pools, plugins) | `bin/magento cache:clean config full_page`; production: `setup:di:compile` |
| `events.xml`, `routes.xml`, `config.xml`, other module XML | `bin/magento cache:clean config`, plus `lynx:compile` if used |
| A `lynx_*.xml` layout file | `bin/magento cache:clean layout full_page` |
| Layout rows from a data patch or the DB | `bin/magento cache:clean lynx_component full_page` |
| What a provider returns | `bin/magento cache:clean full_page` (FPC holds the HTML and the Inertia JSON) |
| A CSV translation | `bin/magento cache:clean translate full_page`, plus `lynx:compile translate` if used |
| Enabled/disabled/added a module with frontend sources, `composer update`, a module's npm packages, the theme's own `package.json` (pulled) | `bin/magento lynx:theme:setup <themes>`, then build |
| New build in use on the checkout (SRI) | `bin/magento cache:clean block_html` (the import-map hashes are cached there) |

Deployment order: `setup:upgrade` → `setup:di:compile` → `lynx:compile` (if used) → `lynx:theme:setup` /
`lynx:static:deploy` → `setup:static-content:deploy`. Static deploy before the Lynx build publishes the old bundle.

## Store configuration a Lynx store view needs

- **Content › Design › Configuration**: Lynx Vue or a child theme.
- Merge CSS, Merge JS, JS Bundling, Move JS code to the bottom: all **No**
  (`dev/css/merge_css_files`, `dev/js/merge_files`, `dev/js/enable_js_bundling`, `dev/js/move_script_to_bottom`).
- Inline Translation off. `Magento_Wishlist` enabled.
- **Stores › Configuration › Lynx › Theme Settings** › Active Design.
- Built JS must be served unchanged (no CDN minify/rewrite): the checkout verifies chunk hashes.
