# 🧩 Power BI Agentic Plugins

Plugins that turn GitHub Copilot into a specialist for Power BI and Microsoft Fabric development. Built for [GitHub Copilot CLI](https://github.com/features/copilot/cli) and [Claude Code](https://claude.com/product/claude-code), also compatible with [VS Code](https://code.visualstudio.com/).

## 💡 Why Plugins

GitHub Copilot helps you write code. Plugins let you go further: teach Copilot how Power BI semantic models should be structured, which Fabric CLI commands to use, how to author reports in PBIR format, and what best practices to follow — so you get accurate, domain-specific help instead of generic suggestions.

Each plugin bundles the skills, tools, and agents for a specific area of the Microsoft data platform. Out of the box, they give Copilot a strong starting point for Power BI and Fabric work. The real power comes when you customize them for your organization — your naming conventions, your workspace structure, your modeling patterns.

## 📦 Plugins

| Plugin                           | What it does                                                                                                                      | 
| -------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- | 
| **[powerbi](./plugins/powerbi)** | Create semantic models, author reports in PBIR, write DAX queries, explore published datasets, and apply modeling best practices. | 
| **[fabric](./plugins/fabric)**   | Navigate workspaces, import/export item definitions, call Fabric & Power BI REST APIs, run jobs, and manage OneLake files.        | 

Every plugin follows the same structure:

```
plugin-name/
├── .claude-plugin/plugin.json   # Manifest
├── .mcp.json                    # Tool connections
├── agents/                      # Agent personas with role-specific instructions
└── skills/                      # Domain knowledge Copilot draws on automatically
```

- **Skills** encode domain expertise, best practices, command references, and step-by-step workflows. Copilot draws on them automatically when relevant.
- **Agents** define personas with specific responsibilities (e.g., a Power BI creator vs. consumer) and declare which skills and tools to use.
- **Connectors** wire Copilot to external tools — the Fabric CLI and Power BI Modeling MCP — via [MCP servers](https://modelcontextprotocol.io/).

**These plugins are starting points.** They become much more useful when you customize them for how your team actually works:

- **Add company context** — Add your naming conventions, workspace structure, and modeling patterns into skill files so Copilot understands your world.
- **Adjust workflows** — Modify skill instructions to match how your team does things (e.g., your deployment pipeline, your BPA rules).
- **Swap connectors** — Edit `.mcp.json` to point at your specific MCP servers.
- **Build new plugins** — Follow the structure above to create plugins for additional scenarios.
  
## 🚀 Getting Started

Make sure you complete the prerequisites, set up your preferred AI assistant, and try one of the [scenarios](#scenarios).

### ✅ Pre-requisites

- Install [Fabric CLI](https://microsoft.github.io/fabric-cli/)

### Copilot CLI

- Install [GitHub Copilot CLI](https://github.com/features/copilot/cli)
- Open Copilot and run the following commands:

    ```bash
    # Open GitHub Copilot CLI
    copilot

    # Add the marketplace (one-time setup)
    /plugin marketplace add RuiRomano/powerbi-agentic-plugins

    # Install plugins
    /plugin install powerbi@powerbi-agentic-plugins
    /plugin install fabric@powerbi-agentic-plugins

    # Restart Copilot to activate
    ```

Once installed, plugins activate automatically. Skills fire when relevant — for example, asking Copilot to create a semantic model automatically pulls in the `powerbi-semantic-model` skill.

### VS Code

- Install [Visual Studio Code](https://code.visualstudio.com/download)
- Install [GitHub Copilot Chat extension](https://marketplace.visualstudio.com/items?itemName=GitHub.copilot-chat)
- Clone this repo and configure [VS Code skills](vscode://settings/chat.agentSkillsLocations) to point at the skill folders:

![vs-code-settings-skills](assets/images/vs-code-settings-skills.png)

### 📊 Scenarios

#### New Direct Lake semantic model on top of Lakehouse tables

```
# 1. Create a Lakehouse in Microsoft Fabric
# 2. Load it with some sample data, e.g. Retail sample data
# 3. Prompt:
    Create a new direct lake semantic model in workspace [workspace] that uses the tables from lakehouse [lakehouse]
```

#### Semantic Model on top of CSV data

```
# Prompt:
    Create a new semantic model based on the CSV files located in `https://github.com/RuiRomano/powerbi-agentic-plugins/tree/main/assets/sample-data` use Power Query HTTP connector. Apply standard modeling best practices throughout (e.g., proper relationships, naming conventions, data types, and star schema design).
    After creating the semantic model, create a Power BI report on top of it.
```

#### Align Report visuals

- Save a report as PBIP
- Close Desktop
- Open the PBIP using Copilot CLI
- Run the prompt
- Reopen PBIP in Power BI Desktop

```
# Prompt:
    Align the Power BI report visuals in the PBIR folder `Path to the PBIP *.Report\ folder`
    
```



## No Warranty / Limitation of Liability

This software is provided "as is" without warranties or conditions of any kind, either express or implied. Microsoft shall not be liable for any damages arising from use, misuse, or misconfiguration of this software.

## Code of Conduct

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/). For more information, see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or contact [open@microsoft.com](mailto:open@microsoft.com) with any additional questions or comments.
