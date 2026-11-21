# Cable Ampacity

MCP tools for Claude that rate power cables. Ask in plain words and Claude calls the CDK AI System
cable-ampacity tools for you: steady-state ampacity by IEC 60287 or Neher-McGrath, the smallest cable for a
load, studies across cables, layouts and load cases, cyclic, emergency and transient ratings, short-circuit
limits, economic conductor size, ratings section by section along a route, which circuit goes in which
duct-bank position, and the design ambient soil temperature at a US site.

## What is in this plugin

- **A connector** (`.mcp.json`) pointing at the hosted server `https://cable-ampacity-mcp.onrender.com/mcp`.
- **A skill** (`skills/cable-ampacity`) that tells Claude when to use the tools, how to ask for missing inputs
  without guessing them, which tool fits each job, and how to report the results, warnings and errors
  faithfully.

The plugin contains only Markdown and JSON. It runs no code on your computer and installs nothing.

## Use it

In Claude Code, install it from its marketplace:

```
/plugin marketplace add chedy028-cdkaisystem/cable-ampacity
/plugin install cable-ampacity@cdkai-cable-tools
```

On claude.ai and in Cowork, add it from Customize > Plugins. Then connect the Cable Ampacity connector from the plugin's Connectors tab (claude.ai and
Cowork) or check it with `/mcp` (Claude Code). Then ask, for example, for the rating of a cable you describe
with its datasheet values and installation. Claude asks once for anything missing, calls the tools, and
replies with the ampacity, conductor temperature, losses, thermal resistances, method, verification status and
every warning the server returned.

## What it sends

The inputs you give for a calculation (cable construction, installation, site, system and load values, and a
site latitude and longitude if you ask for a site-ambient check) are sent to the hosted server at
`cable-ampacity-mcp.onrender.com`, operated by CDK AI System, which computes the result and returns it to
Claude. There is no account, no sign-in and no API key. No local code runs. The plugin sends nothing anywhere
else.

## More

- Documentation: https://docs.cdkaisystem.com
- Support: chedy028@cdkaisystem.com
- Privacy policy: https://docs.cdkaisystem.com/privacy
- Licence: Proprietary (`LicenseRef-Proprietary`). Copyright CDK AI System.

Results are screening results from an automated IEC 60287 or Neher-McGrath calculation. The tools return
calculations only, not the text of IEC or CIGRE publications.
