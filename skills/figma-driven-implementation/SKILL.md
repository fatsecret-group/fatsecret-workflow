---
name: figma-driven-implementation
description: Use when implementing UI components that have a Figma design reference with known nodeIds, to ensure layout, sizing, spacing, and colors match the design exactly
---

## When to Use

- Invoked from `writing-plans` Step 1 Implement when a task creates or modifies UI
- Component nodeIds are available — typically listed in the task's Files section, or carried over from `story-analysis` output (Source / Key UI Specs)
- Do NOT invoke if the current task has no Figma design association

## The Rule

When implementing ANY visual component from a Figma design, query Figma for that specific component. Do NOT rely on memory, the plan, or approximate values.

## Per-Component Workflow

For each UI element you are implementing:

1. **Query**: Call `mcp__figma__get_design_context` with the specific component's nodeId (from the task's Files section or `story-analysis` output)
2. **Extract**: Pull exact values from the returned code/metadata:
   - Dimensions (width, height — use fixed or flexible as design indicates)
   - Padding and margins
   - Font: family, size, weight, line height
   - Colors (map to `FSColorExtension` tokens when possible, otherwise use literal with a `// Figma: #hex` comment — do NOT use `FSColor`, it is outdated and being replaced)
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
| Colors | `FSColorExtension` (check `FSColorExtension.swift` first). **Do NOT use `FSColor` — it is outdated and being replaced.** |
| Fonts (SwiftUI) | `FSFont` (check `FSFont.swift` first) |
| Fonts (UIKit / `NSAttributedString`) | Prefer the `NSAttributedString` extension in `Extensions.swift` that initializes with `FSFont`, instead of assembling attribute dictionaries by hand |
| Components | `DSComp*` series, existing SwiftUI views |
| Spacing | Use literal values if no spacing tokens exist |

If an exact `FSColorExtension`/`FSFont` match exists, use it. If not, use the literal value with a `// Figma: #hex` or `// Figma: <font>/<size>/<weight>` comment.

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
