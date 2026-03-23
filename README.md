# AI Research Agent

An autonomous research agent that takes any topic, gathers information from the web and Wikipedia, generates visual diagrams using Mermaid, and compiles everything into a styled PDF report.

## Architecture

```
User Input (topic)
      │
      ▼
┌─────────────────────────────────────────────┐
│           Tool-Calling Agent Loop           │
│  (LangChain AgentExecutor + gpt-4o-mini)   │
│                                             │
│  ┌─────────┐   The LLM decides which tool  │
│  │   LLM   │──►to call next based on the   │
│  │ (Brain) │   current state and goal.      │
│  └────┬────┘   Uses native function calling │
│       │        (not text-based ReAct).      │
│       ▼                                     │
│  ┌──────────────────────────────────┐       │
│  │         4 Available Tools        │       │
│  │                                  │       │
│  │  1. duckduckgo_search            │       │
│  │     └─ Live web search           │       │
│  │                                  │       │
│  │  2. wikipedia                    │       │
│  │     └─ Structured knowledge      │       │
│  │                                  │       │
│  │  3. create_mermaid_diagram       │       │
│  │     └─ Renders Mermaid → PNG     │       │
│  │       via mermaid.ink API        │       │
│  │                                  │       │
│  │  4. save_research_section        │       │
│  │     └─ Persists title + content  │       │
│  │       to sections.json           │       │
│  └──────────────────────────────────┘       │
│                                             │
│  Loop continues until agent produces a      │
│  final answer (max 25 iterations).          │
└─────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────┐
│              PDF Generation                 │
│                                             │
│  Reads sections.json + diagram_meta.json    │
│  and compiles a styled A4 PDF with:        │
│   - Title page (from user topic)           │
│   - Research sections with **bold** text   │
│   - Mermaid diagrams interleaved inline    │
│     after their related sections           │
│   - Auto page breaks and image scaling     │
└─────────────────────────────────────────────┘
      │
      ▼
  output/Research_Report.pdf
```

## Agent Execution Flow

```
1. User enters research topic via input()
                │
2. Agent receives system prompt with instructions:
   - Research the topic (search + wikipedia)
   - Save each finding as a section
   - Create Mermaid diagrams after each section
                │
3. Agent loop begins (up to 25 iterations):
   │
   ├──► LLM THINKS  → Decides next action
   ├──► TOOL CALL   → Executes chosen tool
   ├──► OBSERVATION  → Tool returns result
   └──► Repeat until all research + diagrams done
                │
4. Agent returns final answer summary
                │
5. PDF cell compiles everything into report
```

## Tool Details

### `duckduckgo_search`
- **Source**: `langchain_community`
- **Purpose**: Live web search for current information
- **Input**: Search query string
- **Output**: Search result snippets

### `wikipedia`
- **Source**: `langchain_community`
- **Purpose**: Structured encyclopedic knowledge
- **Config**: Top 1 result, max 1000 chars
- **Output**: Wikipedia page summary

### `create_mermaid_diagram`
- **Purpose**: Converts Mermaid syntax into PNG images
- **How it works**:
  1. Receives raw Mermaid code from the LLM
  2. Strips markdown fences if present
  3. Converts semicolons to newlines (mermaid.ink requires real newlines)
  4. Base64-encodes the code and sends to `mermaid.ink/img/` API
  5. Saves the returned PNG to `output/diagrams/`
  6. Records metadata (label, position) in `diagram_meta.json` so the PDF knows where to place each diagram
- **Input**: `mermaid_code` (str), optional `label` (str)
- **Output**: File path of saved PNG

### `save_research_section`
- **Purpose**: Persists research findings for PDF compilation
- **How it works**: Appends `{title, content}` to `output/sections.json`
- **Input**: `title` (str), `content` (str)
- **Output**: Confirmation with section count

## Logging System

Every action is logged with timestamps to both the console and `output/agent_log.txt`:

| Tag | Meaning |
|-----|---------|
| `SETUP` | Tools initialized |
| `START` | User topic captured |
| `THINKING` | LLM reasoning |
| `THOUGHT` | LLM produced response |
| `DECIDING` | Agent chose a tool |
| `INPUT` | Tool input (truncated to 200 chars) |
| `RESULT` | Tool output (truncated to 200 chars) |
| `MERMAID` | Diagram render status |
| `SAVED` | Section persisted |
| `DONE` | Agent finished |

Implemented via `AgentLogger` (a LangChain `BaseCallbackHandler`) and inline `log()` calls in custom tools.

## PDF Generation

The `ResearchPDF` class (extends `fpdf2.FPDF`) handles:

- **Title page**: Wraps long titles using `multi_cell()`, centered
- **Markdown rendering**: Converts `**bold**` markers to actual bold font via regex splitting
- **Unicode sanitization**: Replaces em dashes, curly quotes, etc. with ASCII equivalents
- **Image scaling**: Fits diagrams to page width (160mm max), scales down tall images, auto page-breaks
- **Diagram placement**: Uses `diagram_meta.json` to interleave diagrams after their related sections. Falls back to even distribution if metadata is missing.

## Output Structure

```
output/
├── sections.json          # Research findings [{title, content}, ...]
├── diagram_meta.json      # Diagram placement metadata [{path, label, after_section}, ...]
├── agent_log.txt          # Timestamped execution log
├── diagrams/
│   ├── diagram_1.png      # Mermaid-rendered PNGs
│   ├── diagram_2.png
│   └── ...
└── Research_Report.pdf    # Final compiled report
```

## Setup

```bash
# Clone and install
git clone <repo-url>
cd AI-IA-1
uv sync

# Add your OpenAI API key to .env
echo 'OPENAI_API_KEY="sk-..."' > .env
```

## Usage

Run the 3 cells in `agents.ipynb` in order:

1. **Cell 1** — Loads LLM, defines tools, sets up logger
2. **Cell 2** — Prompts for topic, runs the agent (research + diagrams)
3. **Cell 3** — Compiles everything into `output/Research_Report.pdf`

## Dependencies

| Package | Purpose |
|---------|---------|
| `langchain` | Core agent framework |
| `langchain-openai` | OpenAI LLM integration |
| `langchain-community` | DuckDuckGo + Wikipedia tools |
| `langchain-classic` | `create_tool_calling_agent` + `AgentExecutor` |
| `fpdf2` | PDF generation |
| `Pillow` | Image dimension reading for PDF layout |
| `requests` | HTTP calls to mermaid.ink API |
| `python-dotenv` | `.env` file loading |

## Design Decisions

- **Tool-calling agent over text-based ReAct**: The ReAct agent uses a `stop` parameter that newer models (gpt-5-mini) don't support. Tool-calling uses the model's native function calling API — no text parsing, no stop sequences, no hallucinated observations.

- **mermaid.ink over npm/mmdc**: Pure Python, no Node.js dependency. The tool handles semicolons-to-newlines conversion since LLMs tend to generate single-line Mermaid with semicolons, but the API requires real newlines.

- **Diagram metadata for inline placement**: Rather than dumping all diagrams at the end, `diagram_meta.json` tracks which section each diagram was created after, allowing the PDF to interleave visuals with text.

- **Robust input parsing**: The `save_research_section` tool uses structured `title` + `content` params (works cleanly with tool-calling agents) instead of the single-string JSON hack needed for text-based ReAct.
