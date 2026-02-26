---
name: spec
description: Guide to create development specs for Microsoft Fabric data projects. Use this skill whenever asked to create a spec document (/spec command) for a Microsoft Fabric project or to implement a project based on a spec document (/implement command). 
metadata:
  author: microsoft
  version: "1.0"
---

# Spec Skill

## Critical

- Look for a `standards.md` file in your working folder. If exists, make sure it's respected for both spec and implementation.
- Specs should be created in a `specs` folder in the working folder.
- Do not guess data source schemas or structures. If you are asked to build something that requires knowledge of data source ask a clarifying question to get the necessary details.
- Considering the user intent, load the necessary skills

## Commands

### /spec command

When the **/spec** command is present in the prompt, you should create or change a spec document using markdown language.

**Syntax:**

```
/spec [high level user intent for a Microsoft Fabric project]
```

```
/spec [spec file name] [high level user intent for a Microsoft Fabric project]
```

**Process:**

1. **Research:** Analyze the user input and make sure you clearly understand his intent and you are capable of translating that to a development spec. 
  - Consider all attached documents to ensure you have all you need to ellaborate a development plan.
  - It's important to analyze the data source schemas as it will impact the design.
  - For unclear aspects:
    - Make informed guesses based on context and industry standards
    - Ask clarifying questions to the user to fill any gaps in understanding
2. **Create OR Modify Spec Document** 
  - **Create Spec Document:** 
    - Load `/assets/spec-template.md` to understand the expected structure of the spec.    
    - Create a new spec file in `specs/[Name].spec.md`. Do not overwrite any existing spec.
    - Fill the spec document sections based on the template structure. 
      - Look into the comments in the template for guidance on what to include in each section. 
      - Remove the comments once you have filled the sections.
  - **Modify existing spec:**
    - Understand the spec sections
    - Incorporate the user input goal into the spec: design and task plan (if exists)


### /implement command

When the **/implement** command is present in the prompt, you should read the spec document and elaborate a plan to implement it.

**Syntax:**

```
/implement [path to the spec] [tasks:optional]
```

**Process:**

1. **Check if spec document exists** - Make sure the spec document in the path exists. If not stop.
2. **Review spec** - Analyze the spec document to understand the context of what to be implemented
3. **Check for Plan** - Check if the spec includes a Plan
   - If there is a plan, make sure you start execution from the last unchecked task
   - If there is no plan, ellaborate a plan in a separate document and execute on it.
4. **Implement** - Execute the plan tasks
  - As you complete each task, update the plan document to mark the task as done.
  - User may ask to implement only a subset of the tasks with reference to the task number
5. **Execution summary** - After implementation produce a summary of the work done in the document `specs/[SpecName].ExecutionSummary.md`.


