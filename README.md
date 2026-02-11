# powerbi-agentic-plugins

## Pre-Requisites

- Install [Fabric CLI](https://microsoft.github.io/fabric-cli/)
- Install [Copilot CLI](https://github.com/features/copilot/cli)
- Download or Install [Power BI Modeling MCP](https://github.com/microsoft/powerbi-modeling-mcp)

## Copilot CLI setup

```
# 1. Open GitHub Copilot CLI
copilot

#2. Register the Power BI Modeling MCP
/mcp add
    - name: powerbi-modeling-mcp
    - type: stdio
    - command: [path to mcp server download folder]\powerbi-modeling-mcp.exe --start

#3. Register the marketplace (one-time setup)
/plugin marketplace add RuiRomano/powerbi-agentic-plugins

#4. Install plugins

/plugin install powerbi-plugin@powerbi-agentic-plugins
/plugin install fabric-plugin@powerbi-agentic-plugins

#5. Restart Copilot/mcp

```

Other helpful commands:
```
# Remove the marketplace and plugins

/plugin marketplace remove powerbi-agentic-plugins --force

# Update plugin to latest version

/plugin update powerbi-plugin
```

## VS Code setup

Configure [VS Code skills](vscode://settings/chat.agentSkillsLocations) to the skill folder:

![vs-code-settings-skills](assets/images/vs-code-settings-skills.png)

## Scenarios

### New Direct Lake semantic model on top of Lakehouse tables

- Create a new Lakehouse in Fabric
- Load it with sample retail data
- Prompt
    ```
    Create a new direct lake semantic model in workspace [workspace] that uses the tables from lakehouse [lakehouse]
    ```

## No Warranty / Limitation of Liability

This software is provided “as is” without warranties or conditions of any kind, either express or implied. Microsoft shall not be liable for any damages arising from use, misuse, or misconfiguration of this software.

## Code of Conduct

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/). For more information, see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or contact [open@microsoft.com](mailto:open@microsoft.com) with any additional questions or comments.
