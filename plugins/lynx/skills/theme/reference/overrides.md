# Overrides: how the build picks a file

## Theme layout

```
<theme>/
├── registration.php, theme.xml, composer.json
├── Magento_Theme/layout/lynx_default.xml       stylesheet and font preloads (parent's is inherited)
├── OOCS_LynxInertia/templates/inertia.phtml     script tags of the bundle (parent's is inherited)
├── <Vendor_Module>/lynx/<vue|react>/src/…      overrides, per module
├── lynx/                                        Vite workspace
│   ├── package.json (unique name), vite.config.js, tsconfig.json, .gitignore
│   └── src/
│       ├── main.ts | main.tsx                   browser entry
│       ├── ssr.ts, ssr-worker.ts                SSR entries (Node, Cloudflare Worker)
│       └── styles.css                           Tailwind entry, fonts, tokens
└── web/                                         build output only: js/lynx/, css/lynx/
```

Generated, git-ignored: `lynx/tsconfig.modules.json`, `lynx/dist/` (`sources.css`, SSR bundles), `web/js/lynx/*`,
`web/css/lynx/*`.

## Resolution order

Every file is known by a key: module name + path in `src/` (`OOCS_LynxCatalog/Components/Product/Simple.vue`). When
several folders have the same key, the first wins (build kit `resolveSourceDirs()`):

1. The theme's `lynx/src/` (keys without a module prefix, e.g. `Components/Promo.vue`, so it never replaces a module
   file; keep it for entry files)
2. The theme's module overrides `<theme>/<Vendor_Module>/lynx/<framework>/src/`
3. Each parent theme, nearest first: its `lynx/src/`, then its module overrides
4. The modules `<module>/view/frontend/lynx/<framework>/src/` (only modules in `lynx.modules`)

So a parent's override also applies to its children, and a child can override it again. The losing file is not
bundled. Every import of the key gets the winner: the component registry, `import … from 'OOCS_LynxCatalog/…'` from
other modules, and relative imports inside the module. Vue maps every `.vue` and the `.ts` of prefixed folders; React
maps `.tsx` everywhere and `.ts` of prefixed folders; `.d.ts` files and `ssr/` folders are not mapped.

Type checking follows the same order: `tsconfig.modules.json` points each `OOCS_Module/*` path to the theme's override
folder, then parents' override folders, then the module.

## Path examples

| Module file | Override |
|---|---|
| `module-lynx-catalog/view/frontend/lynx/vue/src/Components/Product/Simple.vue` | `<theme>/OOCS_LynxCatalog/lynx/vue/src/Components/Product/Simple.vue` |
| `module-lynx-theme-component/view/frontend/lynx/vue/src/Components/Header.vue` | `<theme>/OOCS_LynxThemeComponent/lynx/vue/src/Components/Header.vue` |
| `module-lynx-theme/view/frontend/lynx/vue/src/stores/global/messagesStore.ts` | `<theme>/OOCS_LynxTheme/lynx/vue/src/stores/global/messagesStore.ts` |
| `Vendor/Module/view/frontend/lynx/react/src/Components/Highlight.tsx` | `<theme>/Vendor_Module/lynx/react/src/Components/Highlight.tsx` (React theme) |

Finding the file behind a component: its `components` entry in a module's `etc/di.xml` has
`<item name="component">OOCS_LynxCatalog::Components/Product/Simple</item>`; `::` becomes `/` and the file is
`Components/Product/Simple.vue` / `.tsx` under that module's `lynx/<framework>/src/`.

## Override not picked up?

- Folder name not exactly the module name, or wrong framework folder (`vue` vs `react`).
- The module is disabled or not in `lynx.modules`: rerun `bin/magento lynx:workspace:setup <themes>`.
- The theme you built is not the one assigned to the store view, or `setup:static-content:deploy` ran before the build
  (production), or the browser serves the old bundle.
- Check the build log line for the chunk: the path must be inside your theme.

## React node contract

A component receives its props and, when it has children, a `slots` prop with each slot already rendered
(`NodeSlots`, a global type; `LayoutItem` too, from `OOCS_LynxTheme/types/inertia.d.ts`). Nothing needs to be attached
to the root; `align` is applied by a wrapper element. Headless components return `null`.

```tsx
type Props = { title: string; slots?: NodeSlots }

export default function Panel({ title, slots }: Props) {
    return (
        <section className="w-full">
            <h2 className="text-xl font-semibold">{title}</h2>
            {slots?.default}
        </section>
    )
}
```

Each node is wrapped in its own `Suspense` and error boundary: a component that fails to load or throws removes only
its own node.

## Porting a Vue override to React

- `<slot name="x" />` → `slots?.x`.
- `router.reload()` always preserves scroll and state on React; do not pass `preserveScroll`.
- `<Deferred data="x">` takes its fallback as the `fallback` prop, not a slot.
- In callbacks after `await`, read fresh state through store actions or a ref, not the render closure.
- Imperative widgets (payment fields, maps) go in a `useRef`, mounted/destroyed in a `useEffect` with cleanup.
- Vue `@change` fires on blur, React `onChange` on every keystroke: clamp numeric input in `onBlur`.
- `config()` sometimes needs an explicit generic: `config<string>('storeCode', '')`.
- Vue `defineExpose({...})` → `useImperativeHandle` with a `ref` prop.
