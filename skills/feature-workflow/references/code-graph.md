# Code Graph — module architecture page conventions

Usable standalone: paste the prompt below as-is, filling `<module/folder>` and `<feature>`.
In feature-workflow Step 3, `<module/folder>` is the module(s) the feature touches and `<feature>` is the feature's output folder; types the design adds or changes carry a plain-text `new` / `changed` tag in their box name (text only — colour stays role, dashed border stays outside-module), and the commit note is the baseline commit the design was drawn against (after build, update the page and its commit note).

```text
Draw an architecture diagram for <module/folder>, as a self-contained HTML page saved under docs/plans/<feature>/architecture/.

Content
- For every type in the module: one sentence on its main purpose.
- List only its public interface: non-private functions and computed properties, each with one line on what it does. Skip declarative UI body/render functions and plain framework lifecycle overrides (e.g. viewDidLoad, onCreate) unless they carry behaviour.
- Show the relationships between types, including who writes each view's state and which objects load their own data (show their data source as its own Service box, and the owner only forwarding events to them).
- Add a short "key flows" section with the call path for the main user actions.
- Group types into layers (entry/controller, state & layout, views, sheets, domain, data, analytics) and mark types that live outside the module.

Diagram
- One relationship diagram as inline SVG at the top, grouped by layer.
- Colour boxes by architectural role only: Controller, ViewModel (stateful, observed), Model / pure value, View, Service (data, analytics, settings). Types outside the module keep their role colour with a dashed border. Groups show layer, line style shows relationship; colour means role and nothing else.
- Arrows follow one convention: solid blue = data flow (source → consumer, e.g. Model → ViewModel → View), grey dashed = user action / method call (caller → callee), grey dotted = structure (subclass, creates, owns). Label the important arrows.
- Name types by role: anything observed by a view is a ViewModel and is named `…ViewModel`; flag names that don't match their role.
- Include a legend for the colours and line styles.
- Show each box's full type name on one line.
- Diagram labels stay in English only.
- No overlapping text, nothing overflowing its box.

Page
- Left sticky table of contents; tables for the per-type function lists; light and dark themes.
- Add a 中文 / EN toggle for all prose, headings, tables and the nav (default English); class and function names stay in English.
- Note the commit the page reflects.

Only update the docs. Do not change code.
```
