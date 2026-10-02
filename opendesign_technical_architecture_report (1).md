# OpenDesign Architecture & Evaluation Report

**Audience:** AI Engineers, Frontend Architects, and Product Teams  
**Subject:** Technical Evaluation of OpenDesign vs. Standard LLM UI Generation  
**Status:** Approved for Team Review  

---

## 1. Executive Summary: What is OpenDesign?

**OpenDesign** is an open-source, local-first runtime harness and live preview workspace that turns autonomous CLI coding agents (e.g., Claude Code, Cursor, Codex) into disciplined, brand-compliant frontend design systems.

Rather than acting as another closed SaaS canvas (like Figma) or an isolated chat wrapper (like standard ChatGPT/Claude code prompts), OpenDesign combines **modular skills**, **strict design token contracts (`DESIGN.md`)**, and a **Model Context Protocol (MCP)** interface with a local daemon and interactive browser sandbox.

---

## 2. The Core Problem: Why "Just Prompting an LLM" Fails for UI

In traditional LLM-assisted frontend development, engineers prompt a foundation model via a web UI or a standard API endpoint (`POST /v1/chat/completions`). This approach introduces significant architectural and visual bottlenecks:

| Problem in Standard Prompting | The Underlying Cause | How OpenDesign Solves It |
| :--- | :--- | :--- |
| **"AI Slop" & Inconsistent Styling** | The model hallucinates arbitrary hex codes, margin sizes, and fonts on zero-shot completions. | Injects a declarative **`DESIGN.md`** token contract into the agent's context, constraining the probabilistic design space. |
| **Monolithic Code Regressions** | Prompting *"make the button blue"* causes the model to regenerate 500 lines of code, breaking adjacent elements. | Uses **DOM-grounded anchors** (`data-od-id`) so the agent applies targeted, surgical diffs to specific components. |
| **Stateless Text Execution** | Standard API calls return raw strings. If there are syntax or runtime errors, human intervention is required to debug. | Drives a **local CLI Agent** with full filesystem and terminal tools to autonomously compile, lint, and self-heal. |
| **Static Mockups vs. Runnable Artifacts** | Designers build static frames in Figma; engineers must rewrite them from scratch in code. | The prototype **is** real, runnable code (HTML/Tailwind/React) saved directly on disk. |

---

## 3. High-Level Architecture: API Calls vs. Agentic Runtime

Below is the architectural contrast between standard text-in/text-out LLM invocation and the OpenDesign execution lifecycle.

### Architecture Comparison Diagram

```
Standard API Call:
User Prompt ──▶ [App Backend] ──▶ [LLM API] ──▶ Raw Code String returned ──▶ Manual file write

OpenDesign Flow:
User Prompt ──▶ [OpenDesign Daemon]
                      │
                      ▼ (spawns / drives)
             [Local CLI Agent: e.g., Claude Code] ◀───┐
                      │                               │  Autonomous
                      ├─▶ Reads DESIGN.md & SKILL.md   │  Execution
                      ├─▶ Creates/Edits files on disk  │  & Fix Loop
                      ├─▶ Checks terminal / lint / build│
                      └─▶ Inspects state via MCP ──────┘
                              │
                              ▼
                     [Live Preview UI]
                              │ (User clicks element to comment)
                              └─────────▶ Targeted feedback fed back to Agent
```

> *Note: A high-resolution Draw.io / vector diagram based on this flow can be generated and embedded directly into your internal documentation repository.*

---

## 4. Key Architectural Building Blocks

OpenDesign coordinates four primary primitives:

```
┌────────────────────────────────────────────────────────┐
│             1. Guidance & Guardrails                   │
│   • SKILL.md: Procedural layouts (Dashboard, Deck)     │
│   • DESIGN.md: Design token contracts (Hex, Grid, Font)│
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│             2. Execution Engine                        │
│   • Local CLI Agent (Claude Code / Cursor CLI / Codex) │
│   • Direct filesystem read/write/patch tools           │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│             3. Sandbox & Preview Layer                 │
│   • Local background daemon (Node / SQLite)            │
│   • Live browser canvas rendering sandboxed iframes    │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│             4. Feedback & Bridge Layer                 │
│   • Model Context Protocol (MCP) server integration    │
│   • DOM node anchoring (`data-od-id`) for click-to-edit│
└──────────────────────────┘
```

### 1. The Design Contract (`DESIGN.md`)
Unlike vague prompt guidelines, `DESIGN.md` acts as a deterministic boundary layer for the agent:
* **Color Palette:** Strict foreground, background, muted, border, and accent hex codes.
* **Layout Grid:** Strict 8px spacing rules (e.g., Tailwind `p-2`, `p-4`, `p-6`).
* **Typography:** Pre-selected fonts, heading scales, and line heights.
* **Posture & Prohibitions:** Explicit negative constraints (e.g., "no gradient buttons", "no borders on card headers").

### 2. Procedural Skills (`SKILL.md`)
Skills are modular blueprints that teach the agent *what* to build. When prompted to generate a "SaaS Analytics Dashboard," the dashboard skill injects layout patterns, recommended component hierarchies, responsive viewport considerations, and interactive state patterns.

### 3. The Local CLI Agent Harness
OpenDesign does not simply make a completion request to a proprietary API. It invokes a persistent CLI coding agent (such as **Claude Code**) running locally on your workstation:
* **Native Tooling:** The agent utilizes `read_file`, `write_file`, and `patch_file` directly inside the workspace directory (`.od/`).
* **Autonomous Self-Correction:** When build scripts fail or syntax errors occur, the agent receives the terminal error stream, adjusts the code, and re-tests without human intervention.

### 4. The Live Daemon & MCP Communication Bridge
A lightweight background daemon monitors file events on disk:
* **Instant Hot Reloading:** As soon as the agent saves changes, the preview canvas in the browser updates.
* **Model Context Protocol (MCP):** The daemon exposes an MCP server that the agent connects to. This allows the model to query the running preview, read user critique logs, and inspect DOM states.

---

## 5. The "Click-to-Edit" Visual Grounding Loop

The defining feature of OpenDesign for collaborative product teams is **spatially grounded feedback**:

1. **DOM Annotation:** When rendering components, OpenDesign injects unique identifiers into DOM nodes (e.g., `data-od-id="stats-summary-card"` or `data-od-id="hero-cta-button"`).
2. **Visual Inspection & Selection:** Instead of writing a vague text prompt ("make the second card on the right wider"), a designer or engineer clicks directly on the target element in the live browser preview canvas.
3. **Structured Context Dispatch:** The UI packages the user comment into a structured instruction payload tied directly to the element selector:
   ```json
   {
     "target_element": "stats-summary-card",
     "selector": "[data-od-id='stats-summary-card']",
     "instruction": "Swap the position of 'Pending' and 'Resolved', and reduce badge font size to 12px"
   }
   ```
4. **Surgical Patching on Disk:** The local CLI agent receives this pinpointed context via MCP/CLI and applies a targeted diff to the corresponding component lines, preventing unintended side effects across the rest of the application.

---

## 6. Real-World Data Ingestion

OpenDesign is not limited to static placeholder text. Because the CLI agent operates in a real local filesystem, teams can bind actual data into prototypes through three mechanisms:

1. **File-Grounded Workspace Data (Recommended):**
   * Place existing datasets (e.g., `data/metrics.json`, `sales.csv`, or `schema.ts`) in the workspace.
   * Instruct the agent to read the file and dynamically populate charts, tables, and metric cards using the local dataset.
2. **Schema-Constrained Mock Generation:**
   * Supply a TypeScript interface or JSON schema.
   * The agent generates realistic, typed mock arrays directly within the component code, ensuring seamless handoff when swapping mock data for production API endpoints.
3. **Live Client-Side API Fetching:**
   * Because the preview runs in a real browser context, the agent can implement standard client-side `fetch()` or `axios` calls targeting local backend endpoints (e.g., `http://localhost:8000/api`) or staging environments.

---

## 7. How OpenDesign Fits into the Team Workflow

| Stage | Traditional Workflow | OpenDesign Workflow |
| :--- | :--- | :--- |
| **Ideation** | PM writes a PRD; abstract user stories. | PM/Engineer drafts intent; selects an existing brand `DESIGN.md`. |
| **Exploration** | UI designer creates multiple static Figma frames (days/weeks). | CLI agent generates multiple interactive HTML/React variants (minutes). |
| **Critique** | Async comments left on static Figma rectangles or Slack screenshots. | Team clicks directly on live DOM nodes in the browser to log targeted fixes. |
| **Engineering Handoff** | Engineers inspect Figma styles and rewrite UI from scratch. | **The prototype is production-grade code.** It commits directly to Git. |

---

## 8. Key Takeaways for AI Engineers

* **Agent Harness > Raw API:** Relying on an autonomous CLI agent with filesystem tools outperforms single-turn text completions by enabling autonomous build checks, linting, and self-healing.
* **Deterministic Guardrails:** `DESIGN.md` solves the "AI slop" problem by turning the brand's visual identity into a machine-enforceable contract.
* **Spatial Feedback:** Attaching DOM-level anchors (`data-od-id`) eliminates ambiguity in iterative multi-turn design workflows.