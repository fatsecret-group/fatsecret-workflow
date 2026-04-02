---
name: figma-design-brief
description: Use when a Figma URL or design reference is provided, before writing implementation plans, to fetch and analyze the design via Figma MCP and produce a design brief
---

## When to Use

- A Figma URL (figma.com/design/...) is present in the conversation or story
- BEFORE invoking `writing-plans`
- Do NOT invoke if no Figma link or design reference is provided

## Steps

1. Parse the Figma URL to extract `fileKey` and `nodeId`
   - `figma.com/design/:fileKey/:fileName?node-id=:nodeId` → convert `-` to `:` in nodeId
   - If node-id is absent, use the root file

2. Call `mcp__figma__get_design_context` with the fileKey and nodeId
   - This returns code, screenshot, and contextual hints

3. Call `mcp__figma__get_screenshot` for visual reference

4. Analyze the design and produce a **Design Brief** containing:
   - **Layout structure**: hierarchy, flex/stack direction, alignment
   - **Components list**: each distinct UI element with its nodeId (save these — implementation will need them)
   - **Design values**: colors, fonts, spacing, corner radius, shadows
   - **Existing project mappings**: map colors to FSColor, fonts to FSFont, components to DSComp* or existing views
   - **Interactions**: tap targets, navigation, animations if noted

5. Present the Design Brief to the user for confirmation before proceeding to `writing-plans`

## Output Format

```markdown
## Design Brief

### Layout
[hierarchy description]

### Components
| Component | Figma nodeId | Existing project match |
|-----------|-------------|----------------------|
| ...       | ...         | FSColor.x / DSComp.y / new |

### Design Values
- Colors: [hex → FSColor mapping]
- Typography: [font/size/weight → FSFont mapping]
- Spacing: [padding/margin values]
- Corner radius: [values]

### Notes
[anything unusual or requiring clarification]
```

## Important

- The Design Brief is the bridge between Figma and the implementation plan
- Save component nodeIds — `figma-driven-implementation` will use them per-component
- If the design uses components with Code Connect mappings, note those for direct reuse
