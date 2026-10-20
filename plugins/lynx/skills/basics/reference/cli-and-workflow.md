# CLI and day-to-day workflow

## Commands

| Command | Arguments and options | What it does |
|---|---|---|
| `lynx:workspace:setup` | `[theme...]` themes to build (replaces those in `package.json`); `--list` list Lynx themes and stop; `--npm=<binary>` (default `LYNX_NPM` env, then `npm` on `PATH`) | Writes `workspaces`, `lynx.themes`, `lynx.modules` of the root `package.json`, then runs `npm install` there and writes `<theme>/lynx/dist/props.d.ts`. First run adds `"build": "npm run build --workspaces --if-present"`. Other keys are kept. |
| `lynx:static:deploy` | `[theme...]` (none = every theme in `package.json`); `--ssr` also `npm run build:ssr`; `--list`, `--npm` | Writes `<theme>/lynx/dist/props.d.ts` (the `Lynx` types from `etc/components/*.graphqls`, also done by `lynx:workspace:setup` and `lynx:static:test`), then `npm run build` in each theme's `lynx/` folder |
| `lynx:pagebuilder:build` | `--npm=<binary>` | `npm ci` + `npm run build` for the admin editor (`view/adminhtml/pagebuilder` of `OOCS_LynxPageBuilder`) |
| `lynx:compile` | `[type...]` (none = all; `--help` lists them) | Writes merged module XML, translations and the merged `etc/components/*.graphqls` (`components`) to `generated/lynx_compile` (needs `OOCS_LynxCompiler`) |
| `lynx:mcp:serve`, `lynx:mcp:tools` | see skill mcp-tool | MCP server (needs `OOCS_LynxMcp`) |

Theme argument = theme code with `_` for `/` (`OOCS_LynxVue`, `OOCS_LynxReact`, `Vendor_storefront`). A theme is
buildable when it has `lynx/package.json`.

## Theme workspace scripts (`<theme>/lynx/`)

| Script | Does |
|---|---|
| `npm run build` | `type-check`, then Vite → `../web/js/lynx/*.min.js`, `../web/css/lynx/styles.min.css` |
| `npm run type-check` | `lynx-tsconfig-vue && vue-tsc -p tsconfig.modules.json --noEmit` (React: `lynx-tsconfig-react && tsc …`) |
| `npm run dev` | Vite dev server (port 5173); picks up added/removed files |
| `npm run build:ssr` | Node SSR bundle → `dist/ssr/nodejs/server.mjs` |
| `npm run build:ssr:worker` | Cloudflare Worker bundle → `dist/ssr/worker/worker.mjs` |
| `npm run ssr` | Runs the Node renderer |

`tsconfig.modules.json` is generated (git-ignored): one `paths` entry per module, pointing to the theme's override
folder, then parent themes' override folders, then the module. A missing module or export is a type error.

## Lint

Each theme ships `lynx/eslint.config.mjs` that lints the theme's `src/` plus every module in `lynx.modules`:

```bash
npx eslint --config vendor/oocscommerce/theme-frontend-lynx-vue/lynx/eslint.config.mjs \
  'app/code/Vendor/Module/view/frontend/lynx/vue/src/**/*.{ts,vue}'
npx eslint --config vendor/oocscommerce/theme-frontend-lynx-react/lynx/eslint.config.mjs \
  'app/code/Vendor/Module/view/frontend/lynx/react/src/**/*.{ts,tsx}'
```

Notable rules: no DOM globals at module scope (Vue `ssr-friendly/…`, React `better-ssr/…`), `no-eval`,
`@typescript-eslint/no-deprecated`, `no-console` except `warn`/`error`; Vue: `vue/no-mutating-props`,
`vue/require-explicit-emits`; React: `@eslint-react/no-missing-key`, `no-nested-component-definitions`.

## What to run after a change

| You changed | Run |
|---|---|
| `.vue`, `.ts`, `.tsx` | `npm run build` (or `bin/magento lynx:static:deploy`), reload. Production: also `setup:static-content:deploy` |
| `etc/components/*.graphqls` (components, settings, slots, props) | `bin/magento cache:clean lynx_component full_page`, plus `lynx:compile components` if used; `lynx:static:deploy` to regenerate the `Lynx.*` types |
| `di.xml` (providers' dependencies, pools, plugins) | `bin/magento cache:clean config full_page`; production: `setup:di:compile` |
| `events.xml`, `routes.xml`, `config.xml`, other module XML | `bin/magento cache:clean config`, plus `lynx:compile` if used |
| A `lynx_*.xml` layout file | `bin/magento cache:clean layout full_page` |
| Layout rows from a data patch or the DB | `bin/magento cache:clean lynx_component full_page` |
| What a provider returns | `bin/magento cache:clean full_page` (FPC holds the HTML and the Inertia JSON) |
| A CSV translation | `bin/magento cache:clean translate full_page`, plus `lynx:compile translate` if used |
| Enabled/disabled/added a module with frontend sources | `bin/magento lynx:workspace:setup <themes>`, then build |
| New build in use on the checkout (SRI) | `bin/magento cache:clean block_html` (the import-map hashes are cached there) |

Deployment order: `setup:upgrade` → `setup:di:compile` → `lynx:compile` (if used) → `lynx:workspace:setup` /
`lynx:static:deploy` → `setup:static-content:deploy`. Static deploy before the Lynx build publishes the old bundle.

## Store configuration a Lynx store view needs

- **Content › Design › Configuration**: Lynx Vue, Lynx React or a child theme.
- Merge CSS, Merge JS, JS Bundling, Move JS code to the bottom: all **No**
  (`dev/css/merge_css_files`, `dev/js/merge_files`, `dev/js/enable_js_bundling`, `dev/js/move_script_to_bottom`).
- Inline Translation off. `Magento_Wishlist` enabled.
- **Stores › Configuration › Lynx › Design** › Active Design.
- Built JS must be served unchanged (no CDN minify/rewrite): the checkout verifies chunk hashes.
