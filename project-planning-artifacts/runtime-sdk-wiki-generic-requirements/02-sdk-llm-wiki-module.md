# Requirement: SDK LLM Wiki Module

## Goal

Provide CrewAgent with an internal SDK-aware knowledge module that lets the main
LLM use imported SDK Wiki content to select APIs, infer dependencies, and create
small-step execution plans.

## Internal Abilities

The module should expose internal tools or equivalent context functions:

```text
sdk_wiki.list_sdks
sdk_wiki.search_pages
sdk_wiki.read_page
sdk_wiki.resolve_symbol
sdk_wiki.expand_relations
sdk_wiki.plan_api_usage
```

These are CrewAgent internal abilities, not MCP tools.

## Required Behavior

- Use imported index files for page discovery.
- Read structured Markdown pages on demand.
- Support API, workflow, concept, and relation page types.
- Return source references with API recommendations.
- Require plans to reference existing Wiki pages.
- Prevent silent use of APIs not present in the SDK Wiki.

## `sdk_wiki.plan_api_usage`

Input should include:

```json
{
  "sdkId": "sam",
  "intent": "给当前板架模型施加向下均布压力",
  "modelState": {},
  "constraints": {
    "preferSmallSteps": true,
    "requireSourceRefs": true
  }
}
```

Output should include:

```json
{
  "ok": true,
  "taskType": "apply_pressure_load",
  "requiredApis": [],
  "executionPlan": [],
  "missingInformation": [],
  "confidence": 0.0
}
```

## LLM Boundary

All LLM reasoning happens inside CrewAgent. SAM MCP Server and SAM Bridge must not
call LLMs for SDK understanding.

## Acceptance Criteria

- Given "施加压力载荷", the module can surface Pressure, Surface, and StaticStep
  pages from the SAM SDK Wiki.
- Plans include source references.
- If the relevant Wiki pages are missing, the module reports missing knowledge
  instead of inventing API names.
