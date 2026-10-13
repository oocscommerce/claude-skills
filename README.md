# Lynx skills for Claude Code

Skills that teach [Claude Code](https://claude.com/claude-code) how to build on [Lynx](https://docs.oocscommerce.com/lynx/), the single-page storefront for Magento 2. With them in your Magento project, ask Claude Code for the result, such as "add a Lynx payment renderer for my method `vendor_invoice`", and it writes the module the way the [Lynx developer guide](https://docs.oocscommerce.com/lynx/developer/) describes.

## Skills

| Skill | Use it to |
|---|---|
| `lynx-basics` | Shared foundation: module skeleton, dependencies, styling rules, CLI. The other skills build on it. |
| `lynx-component` | Create a PageBuilder component (Vue and React) and register it |
| `lynx-component-settings` | Add or change the settings a component shows in the editor |
| `lynx-provider` | Give a component its data from a PHP provider |
| `lynx-page` | Add a storefront page or route, built in PageBuilder or with its own page component |
| `lynx-forms-analytics` | Post forms, show server messages, refresh customer data, track analytics events |
| `lynx-account-tab` | Add a My Account tab |
| `lynx-payment-method` | Add a payment method renderer to the Lynx checkout |
| `lynx-shipping-method` | Add a shipping method renderer to the Lynx checkout |
| `lynx-checkout-step` | Add a checkout step |
| `lynx-product-type` | Support a new or third-party product type on the storefront |
| `lynx-theme` | Create a child theme or override a component |
| `lynx-ssr` | Make components render correctly with server-side rendering |
| `lynx-extension-port` | Make an existing Luma extension work on Lynx |
| `lynx-mcp-tool` | Add a tool to the Lynx MCP server |

## Requirements

- Claude Code
- A Magento project with Lynx installed (`oocscommerce/lynx`), see [Installation](https://docs.oocscommerce.com/lynx/developer/installation)

## Install

Copy the skills into the Magento project, so everyone working on it gets them. Run from the Magento root folder:

```bash
git clone --depth 1 https://github.com/oocscommerce/claude-skills.git /tmp/lynx-skills
mkdir -p .claude/skills
cp -r /tmp/lynx-skills/skills/* .claude/skills/
rm -rf /tmp/lynx-skills
```

Commit `.claude/skills` with the project. To have the skills in every project on your machine instead, copy them to `~/.claude/skills/`.

Start a new Claude Code session in the project. `/skills` lists the installed skills.

## Use

Describe what you want; Claude Code picks the matching skill:

```text
Add a Lynx component "Store Hours" with a title setting and a list of opening hours, for Vue and React.
```

You can also call a skill by name, such as `/lynx-component`. Review the code it writes like any other code, then build and test as described in [Development Setup](https://docs.oocscommerce.com/lynx/developer/setup).

## Update

Run the install commands again. They replace the `lynx-*` skills with the latest version and leave your other skills alone.

## License

The skills are released under the [MIT License](LICENSE). Lynx itself is commercial software with its own license.
