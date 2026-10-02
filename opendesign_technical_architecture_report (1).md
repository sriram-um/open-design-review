# OpenDesign: Architecture & Workflow Guide

**Subject:** Technical Evaluation of OpenDesign vs. Standard LLM UI Generation

---

## 1. What Is OpenDesign & What Problem Does It Solve?

Prompting a standard LLM chat window or making raw API calls (`POST /v1/chat/completions`) for frontend code quickly breaks down:

- **Stylistic drift** — Models introduce arbitrary fonts, mismatched hex colors, and inconsistent paddings across turns.
- **Workflow disconnect** — Output arrives as a raw text string in a chat transcript, requiring manual copy-pasting, package management, and dependency wiring.
- **Ambiguous feedback loops** — An instruction like "make that card on the right slightly wider" forces the model to infer the target, often breaking other parts of the layout.

OpenDesign is an **agent runtime harness and live preview workspace**. Instead of asking a model for a text completion, it launches a real local coding agent (such as Claude Code or Cursor CLI) directly on your workspace files, constrains it with design tokens, and renders the result in a live browser canvas.

---

## 2. Core Architecture: Standard API vs. OpenDesign

**Standard API call:**

```text
User Prompt ──▶ [App Backend] ──▶ [LLM API] ──▶ Raw code string returned ──▶ Manual file write
```

**OpenDesign flow:**

```text
User Prompt ──▶ [OpenDesign Daemon]
                      │
                      ▼ (spawns / drives)
             [Local CLI Agent: e.g., Claude Code] ◀───┐
                      │                              │  Autonomous
                      ├─▶ Reads DESIGN.md & SKILL.md  │  execution
                      ├─▶ Creates/edits files on disk │  & fix loop
                      ├─▶ Checks terminal / lint / build
                      └─▶ Inspects state via MCP ─────┘
                              │
                              ▼
                     [Live Preview UI]
                              │ (user clicks element to comment)
                              └─────────▶ Targeted feedback fed back to agent
```

---

## 3. The Three Core Building Blocks

### 3.1 Guidance Layer — `DESIGN.md` & `SKILL.md`

- **`DESIGN.md` (visual contract)** — A declarative markdown contract specifying exact hex palettes, typography scales, border radii, and spacing systems (e.g., an 8px grid). It acts as guardrails so the agent doesn't guess styles.
- **`SKILL.md` (blueprints)** — Modular procedural instructions teaching the agent how to construct specific layouts (e.g., analytics dashboards, pitch decks, mobile onboarding flows).

### 3.2 Execution Layer — Local CLI Agent

- OpenDesign delegates execution to a local CLI agent on your machine (like Claude Code).
- The agent has native filesystem access: it creates actual files, patches multi-file component trees, executes terminal commands, and resolves syntax or build errors without operator intervention.

### 3.3 Runtime Layer — Daemon, Preview & MCP

- **Express daemon** — A lightweight local service that watches project files and coordinates the runtime.
- **Live sandbox preview** — Renders components inside an isolated `srcDoc` browser iframe.
- **MCP interface** — Exposes standard Model Context Protocol endpoints so external tools can discover projects and manipulate workspace files programmatically.

---

## 4. Primary Differentiator: "Click-to-Edit" Visual Grounding

The platform's most significant workflow advantage is linking visual critique directly to code changes, removing conversational ambiguity:

1. **Auto-annotation** — The preview renderer ensures structural elements contain tracking hooks (like `data-od-id`), or generates fallback DOM selectors.
2. **Click-to-comment** — A reviewer clicks directly on any element in the browser preview (e.g., a KPI card or button) and types an instruction.
3. **Structured payload** — The preview bridge captures the exact selector and note:

   ```json
   {
     "selector": "[data-od-id='stats-summary-card']",
     "note": "Swap badge background to slate-100 and reduce font size to 12px."
   }
   ```

4. **Scoped diff** — The agent receives this grounded target, locates the corresponding code on disk, and applies an isolated diff — removing guesswork and avoiding layout regressions.

---

## 5. Working with Real Data

Prototypes are not restricted to hardcoded placeholder text:

| Approach | How it works |
| --- | --- |
| **Workspace data files** *(recommended)* | Drop `metrics.json` or `records.csv` into a `./data/` folder. The agent reads the file on disk and dynamically renders tables and charts against real records. |
| **Schema mocking** | Supply a TypeScript interface or OpenAPI schema. The agent creates typed mock arrays adhering strictly to your data model, so engineers can swap mocks for live API hooks later. |
| **Live API fetching** | Because the output is standard browser code, the agent can write native `fetch()` calls to query local REST or GraphQL endpoints running on your workstation. |

---

## 6. Verification Checklist

To test OpenDesign on your workstation:

- [ ] Start the backend daemon (`pnpm exec od`).
- [ ] Connect your local agent CLI (e.g., Claude Code).
- [ ] Add a sample `DESIGN.md` into the project root.
- [ ] Provide a prompt referencing a `./data/sample.json` file.
- [ ] Open the browser preview, click an element, and submit an edit note to verify targeted code diffing.
