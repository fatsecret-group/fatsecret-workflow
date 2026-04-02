---
name: figma-driven-implementation
description: Use when implementing UI components that have a Figma design reference with known nodeIds, to ensure layout, sizing, spacing, and colors match the design exactly
---

## When to Use

- A Figma design was analyzed earlier (via `figma-design-brief`) and component nodeIds are available
- You are about to write or modify UI code for a component that has a Figma reference
- Do NOT invoke if the current task has no Figma design association

## The Rule

When implementing ANY visual component from a Figma design, query Figma for that specific component. Do NOT rely on memory, the plan, or approximate values.

## Per-Component Workflow

For each UI element you are implementing:

1. **Query**: Call `mcp__figma__get_design_context` with the specific component's nodeId (from the Design Brief)
2. **Extract**: Pull exact values from the returned code/metadata:
   - Dimensions (width, height — use fixed or flexible as design indicates)
   - Padding and margins
   - Font: family, size, weight, line height
   - Colors (map to FSColor tokens when possible, otherwise use literal with a `// Figma: #hex` comment)
   - Corner radius
   - Shadows and opacity
3. **Implement**: Write SwiftUI code using these exact values
4. **Next**: Move to the next component

```
For each component:
  get_design_context(nodeId) → extract values → write SwiftUI → next
```

## Mapping to Project Design System

| Figma value | Project equivalent |
|-------------|-------------------|
| Colors | FSColor (check FSColor.swift first) |
| Fonts | FSFont (check FSFont.swift first) |
| Components | DSComp* series, existing SwiftUI views |
| Spacing | Use literal values if no spacing tokens exist |

If an exact FSColor/FSFont match exists, use it. If not, use the literal value.

## After All Components Are Implemented

1. Take a simulator screenshot: `mcp__xcodebuildmcp__screenshot`
2. Get the Figma screenshot: `mcp__figma__get_screenshot`
3. Present both to the user for visual comparison
4. Fix any discrepancies by re-querying the specific component's nodeId

## Red Flags

| Thought | Reality |
|---------|---------|
| "I remember the spacing from the plan" | Query Figma. Memory drifts. |
| "This looks about right" | Check exact values. "About right" ≠ pixel-correct. |
| "I'll verify at the end" | Too late — check per component. |
| "The plan says 16pt padding" | Plans approximate. Figma is source of truth. |
| "I'll batch all components first" | No. One at a time, query each. |

## This Is a Rigid Skill

Follow exactly. Do not skip the per-component query step for any reason.
