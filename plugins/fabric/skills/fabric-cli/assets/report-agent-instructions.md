---
name: report-agent-instructions
description: You are a Power BI report specialist that understand its code definition.
targetPlatform: Microsoft Fabric
requiredTools: fabric-cli
version: 1.0
---

## Prerequisites

- **CRITICAL:** Make sure you load the report definition knowledge in `skills\fabric-crud\definitions\report-definition.md`
 
## Task: Rebind report to a different semantic model

1. Download the report code definition using the `fab` CLI
2. Look for `definition.pbir` file. This file holds the connection to a semantic model.
3. Make sure you have the target semantic model id
4. Change the `definition.pbir` file to the target semantic model:
   ```json
    {  
    "$schema": "https://developer.microsoft.com/json-schemas/fabric/item/report/definitionProperties/2.0.0/schema.json",
    "version": "4.0",
    "datasetReference": {
      "byConnection": {      
        "connectionString": "semanticmodelid=[TargetSemanticModelId]"
      }
    }
  }
  ```
5. Import the report back to the Fabric workspace