# Agent Orchestration

A collection of multi-agent pipelines built with **LangGraph** that turn a single natural-language topic into a finished artifact — either a complete software project or a publishable blog post. Each notebook is a self-contained, runnable Colab demo.

---

## Pipeline Position

```
User Topic
    │
    ▼
 Router Agent          ← decides: skip or search?
    │
    ├──(needs research)──▶  Research Agent  ─┐
    │                                         │
    └──(no research needed)──────────────────▶ Orchestrator / Architect
                                                       │
                                              Parallel Worker Agents  ← fan-out via Send()
                                                       │
                                              Reviewer / Reducer
                                                       │
                                              Assembler / Editor
                                                       │
                                             Final Artifact (project or blog post)
```

---

## Notebooks

### 1. `Developer_agent.ipynb`

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Janardan-thapaliya/Agent_Orchestration/blob/main/Developer_agent.ipynb)

A **code-generation pipeline** that accepts a software project description and produces production-ready, modular Python code with tests.

#### Pipeline

| Step | Node | Model | Role |
|------|------|-------|------|
| 1 | `router` | `gpt-4.1-mini` | Decides if external research is needed; emits Tavily queries |
| 2 | `research` *(optional)* | Tavily | Fetches docs, repos, papers; filters to `EvidenceItem` list |
| 3 | `architect` | `gpt-4o` | Designs a `ProjectPlan`: name, domain, tech stack, 4–8 `DevTask`s |
| 4 | `developer` ×N | `gpt-4.1` | Fan-out — one worker per task; each returns a Markdown module with code + tests |
| 5 | `review` | `gpt-4.1` | Structured code review → `ReviewResult(has_issues, issues[])` |
| 6 | `autofix` *(conditional)* | `gpt-4.1` | Applies minimal targeted fixes if reviewer found issues |
| 7 | `assembler` | `gpt-4.1-mini` | Merges all modules, generates `README.md` + `PROJECT.md`, saves to disk |

<img width="212" height="917" alt="image" src="https://github.com/user-attachments/assets/263301fe-dfba-47b2-8114-741b3acfdc1f" />

#### Key Schemas

- **`RouterDecision`** — `needs_research: bool`, `queries: List[str]`
- **`ProjectPlan`** — `project_name`, `domain` (`NLP / Image Processing / ML / General Software`), `tech_stack`, `tasks: List[DevTask]`
- **`DevTask`** — `id`, `title`, `objective`, `deliverables`, `target_files`, `requires_tests`
- **`ReviewResult`** — `has_issues: bool`, `issues: List[str]`
- **`State`** — LangGraph TypedDict; `modules` uses `Annotated[List[str], operator.add]` for fan-out merge

#### Setup

```bash
pip install langchain_community langchain_openai langgraph langchain_core langchain-google-genai python-dotenv
```

```python
import os
os.environ["OPENAI_API_KEY"] = "..."
os.environ["TAVILY_API_KEY"] = "..."
```

#### Usage

```python
result = app.invoke({"topic": "Build a FastAPI service with a vector search endpoint using FAISS"})
print(result["final_project"])   # generated README
# project files saved to disk under the project name
```

---

### 2. `Blog_writer_psychology_specific.ipynb`

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Janardan-thapaliya/Agent_Orchestration/blob/main/Blog_writer_psychology_specific.ipynb)

A **psychology blog writing pipeline** tailored for curious, non-expert readers. The router classifies every topic into one of three research modes before writing begins, ensuring evergreen topics stay grounded in theory while evidence-heavy topics are backed by real citations.

#### Research Modes

| Mode | When triggered | Research? |
|------|---------------|-----------|
| `closed_book` | Classic, evergreen psychology (attachment, habits, memory) | No |
| `hybrid` | Evergreen + benefits from recent studies or modern context | Yes |
| `open_book` | Time-sensitive: new studies, DSM updates, statistics | Yes |

#### Pipeline

| Step | Node | Role |
|------|------|------|
| 1 | `router` | Classifies mode; generates 3–10 Tavily queries if needed |
| 2 | `research` *(optional)* | Fetches and deduplicates evidence; prefers peer-reviewed/academic sources |
| 3 | `orchestrator` | Plans the blog: title, audience, tone, 5–9 sections as `Task` objects |
| 4 | `worker` ×N | Fan-out — one worker per section; respects `requires_citations` flag |
| 5 | `reducer` | Sorts sections by `task.id`, stitches the final Markdown, saves `{blog_title}.md` |

#### Key Schemas

- **`RouterDecision`** — `needs_research`, `mode`, `queries`
- **`Plan`** — `blog_title`, `audience`, `tone`, `blog_kind`, `constraints`, `tasks`
- **`Task`** — `id`, `title`, `goal`, `bullets[3–6]`, `target_words`, `requires_research`, `requires_citations`, `requires_code`
- **`EvidenceItem`** — `title`, `url`, `published_at`, `snippet`, `source`

#### Usage

```python
result = run("Self sabotage by our own brain")
print(result["final"])   # full Markdown blog post
# also saved to "{blog_title}.md"
```

<img width="203" height="697" alt="image" src="https://github.com/user-attachments/assets/bcd607d5-2636-4508-9520-f223580cdef0" />

### 3. `Blog_writing_agent_with_images.ipynb`

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Janardan-thapaliya/Agent_Orchestration/blob/main/Blog_writing_agent_with_images.ipynb)

An **extended blog pipeline** that adds two new roles on top of the psychology writer:

- **Lead Writer** (`gpt-4.1`) — writes a long-form introductory lead section before the fan-out workers start
- **Editor** (`gpt-4.1`) — does a final editorial pass on the assembled draft for tone, flow, and coherence

The result is a more polished, magazine-grade post with a strong opening and consistent voice throughout.

#### Extended State

Adds `lead_section: Optional[str]` to the shared state; the reducer inserts the lead before the parallel sections.

#### Model Assignment

| Role | Model |
|------|-------|
| Router | `gpt-4.1-mini` |
| Research | `gpt-4.1-mini` |
| Orchestrator | `gpt-4.1` |
| Lead Writer | `gpt-4.1` |
| Section Workers | `gpt-4.1-mini` |
| Editor | `gpt-4.1` |

---

## Key LangGraph Patterns Used

| Pattern | Where |
|---------|-------|
| **Conditional routing** (`add_conditional_edges`) | Router → research or architect/orchestrator; reviewer → autofix or assembler |
| **Parallel fan-out** (`Send`) | Architect/orchestrator → multiple workers; one `Send` per task |
| **Reducer accumulation** (`Annotated[List, operator.add]`) | Workers write to `modules` / `sections` without clobbering each other |
| **Structured output** (`.with_structured_output(Schema)`) | Router, architect, reviewer — every decision is a typed Pydantic model |

---

## Dependencies

```
langchain_community
langchain_openai
langchain_core
langchain-google-genai   # Developer agent only
langgraph
python-dotenv
```

**API Keys required:**

| Key | Used by |
|-----|---------|
| `OPENAI_API_KEY` | All notebooks |
| `TAVILY_API_KEY` | All notebooks (research nodes) |
| `GOOGLE_API_KEY` | `Developer_agent.ipynb` only |
