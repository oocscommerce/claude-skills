# Lynx skills for Claude Code

Skills that teach [Claude Code](https://claude.com/claude-code) how to build on [Lynx](https://docs.oocscommerce.com/lynx/), the single-page storefront for Magento 2. With them in your Magento project, ask Claude Code for the result, such as "add a Lynx payment renderer for my method `vendor_invoice`", and it writes the module the way the [Lynx developer guide](https://docs.oocscommerce.com/lynx/developer/) describes.

## Skills

| Skill | Use it to |
|---|---|
| `basics` | Shared foundation: module skeleton, dependencies, styling rules, CLI. The other skills build on it. |
| `component` | Create a PageBuilder component (Vue and React) and register it |
| `component-settings` | Add or change the settings a component shows in the editor |
| `provider` | Give a component its data from a PHP provider |
| `page` | Add a storefront page or route, built in PageBuilder or with its own page component |
| `forms-analytics` | Post forms, show server messages, refresh customer data, track analytics events |
| `account-tab` | Add a My Account tab |
| `payment-method` | Add a payment method renderer to the Lynx checkout |
| `shipping-method` | Add a shipping method renderer to the Lynx checkout |
| `checkout-step` | Add a checkout step |
| `product-type` | Support a new or third-party product type on the storefront |
| `theme` | Create a child theme or override a component |
| `ssr` | Make components render correctly with server-side rendering |
| `testing` | Write and run tests for components, stores and theme overrides |
| `extension-port` | Make an existing Luma extension work on Lynx |
| `mcp-tool` | Add a tool to the Lynx MCP server |

## Requirements

- Claude Code 2.1.275 or later
- A Magento project with Lynx installed (`oocscommerce/lynx`), see [Installation](https://docs.oocscommerce.com/lynx/developer/installation)

## Install

In Claude Code, run:

```text
/plugin install lynx --marketplace oocscommerce/claude-skills
```

This adds the `oocscommerce` marketplace and installs the `lynx` plugin in one step. The same in two steps, or from a shell:

```bash
claude plugin marketplace add oocscommerce/claude-skills
claude plugin install lynx@oocscommerce
```

### For a whole team

Run these from the Magento root folder, then commit `.claude/settings.json`:

```bash
claude plugin marketplace add oocscommerce/claude-skills --scope project
claude plugin install lynx@oocscommerce --scope project
```

The settings file declares the marketplace and enables the plugin for the project. Each teammate runs `claude plugin install lynx@oocscommerce --scope project` once to download it.

## Use

Describe what you want; Claude Code picks the matching skill:

```text
Add a Lynx component "Store Hours" with a title setting and a list of opening hours, for Vue and React.
```

You can also call a skill by name, such as `/lynx:component`. Review the code it writes like any other code, then build and test as described in [Development Setup](https://docs.oocscommerce.com/lynx/developer/setup).

## Update

```bash
claude plugin marketplace update oocscommerce
claude plugin update lynx@oocscommerce
```

Or turn on auto-update for the marketplace in `/plugin` › Marketplaces.

## License

The skills are released under the [MIT License](LICENSE). Lynx itself is commercial software with its own license.
