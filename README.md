# TraceViz

**An interactive AI-powered educational animation system that uses deterministic execution traces as the source of truth and LLMs to generate visual explanations of Computer Science concepts in real time.**

---

## The Problem

Students struggle with process-based CS concepts — B-tree insertion, page replacement, CPU scheduling — because these involve **multiple states and transitions**, not a single fact to memorize.

A textbook says:

> "Insert the key into the B-tree, and if the node overflows, split it."

But it doesn't show *what the tree looked like before*, *where the key went*, *why the node became full*, *when the split happened*, or *which key moved upward*.

Existing AI animation tools can generate visually attractive animations — but an LLM can get the actual sequence wrong. An animation may look perfectly polished while showing an **incorrect** B-tree split.

---

## The Core Idea — Trace First

TraceViz does **not** work like this:

```
User → LLM → Manim → Animation
```

It works like this:

```
User → Deterministic Simulator → Execution Trace → LLM → Manim → Animation
```

The deterministic simulator is the source of truth.

**Python determines WHAT happens. The LLM determines HOW it is presented.**

This separation is the central design principle of the entire project.

---

## Architecture

```
                 USER REQUEST
                      |
                      v
             PRIMITIVE CACHE CHECK
                /          \
          HIT /              \ MISS
            /                  \
           v                    v
     Cached Output      Deterministic Python
                             Simulator
                                |
                                v
                        VERIFIED TRACE (JSON)
                                |
                  +-------------+-------------+
                  |                           |
                  v                           v
         Interactive Preview          LLM Trace-to-Manim
                  |                           |
                  |                           v
                  |                    Manim Renderer
                  |                           |
                  +-------------+-------------+
                                |
                                v
                        Student Interface
```

The trace is the central data structure — animation, UI, and explanation all derive from it.

Note the left branch: the interactive preview reads the trace **directly**, without waiting on the LLM or a Manim render. That shortcut is what makes step-through interaction responsive.

---

## What is an Execution Trace?

A structured record of how the concept changes over time — enough information to reconstruct the visual state at every important step.

```json
{
  "concept": "LRU Page Replacement",
  "input": [1, 2, 3, 1, 4],
  "steps": [
    { "step": 1, "reference": 1, "state": [1],       "action": "page_loaded" },
    { "step": 2, "reference": 2, "state": [1, 2],    "action": "page_loaded" },
    { "step": 3, "reference": 3, "state": [1, 2, 3], "action": "page_loaded" }
  ]
}
```

---

## Initial Concepts

| Domain | Concept | Demonstrates |
|---|---|---|
| DBMS | B-Tree insertion and node splitting | hierarchical structure, overflow, splitting, key promotion |
| OS | LRU page replacement | sequential execution, frame state, eviction decisions |

Two concepts are enough to prove the architecture. More follow once the core pipeline works.

---

## Features

- **Live drill-down** — pause at any step and ask why it happened; answered from the trace
- **Adjustable pace** — play, pause, previous, next, speed control, step-by-step
- **Personalized code tracing** — students supply their own keys or reference string, get their own trace
- **Side-by-side contrast** — compare two executions
- **Reusable animation primitives** — trace events map to a fixed primitive set (`split_node`, `evict_page`, ...) instead of generating visuals from scratch
- **Caching** — reusable primitives and rendered outputs, to avoid regenerating work

---

## Tech Stack

| Layer | Technology |
|---|---|
| Simulation / backend | Python |
| Animation | Manim |
| AI | LLM API (trace-to-Manim translation, explanations) |
| Cache | Reusable primitives + rendered output store |
| Frontend | Web frontend (framework not finalized) |

---

## Project Structure

```
traceviz/
├── backend/
│   ├── concepts/
│   │   ├── btree/
│   │   │   ├── simulator.py      # deterministic execution, owns correctness
│   │   │   ├── trace.py          # emits standard JSON trace
│   │   │   ├── primitives.py     # event -> animation primitive mapping
│   │   │   └── renderer.py       # Manim scene construction
│   │   └── lru/                  # same four-file layout
│   ├── trace/
│   │   ├── schema.py             # shared trace format — concept-agnostic
│   │   └── validator.py          # trace verification layer
│   ├── llm/
│   │   └── trace_to_manim.py
│   ├── cache/
│   └── api/
├── animations/
│   ├── primitives/
│   └── renderers/
├── frontend/
├── tests/
├── traces/
└── docs/
```

Each concept is self-contained. Adding a new one means adding a folder, not rewriting the system.

---

## Getting Started

```bash
git clone https://github.com/<username>/traceviz.git
cd traceviz

python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate

pip install -r requirements.txt
```

Run the tests:

```bash
pytest tests/
```

---

## Team

Four members, four connected layers of one pipeline — not four separate projects.

```
M1: Concept Input → Simulator → Raw Trace (JSON)
        |                              \  (direct, for instant preview)
M2: Verify + Enrich → Verified Trace     \
        |                                 \
M3: Trace → Primitives → LLM → Manim → Rendered Output (cached)
        |                                 /
M4: Trace Viewer + Playback  ←———————————
```

| Member | Role | Owns |
|---|---|---|
| M1 | Trace Engine Lead | Deterministic simulators, JSON trace schema, what-if replay |
| M2 | Trace Intelligence Lead | Trace verification, student code → trace, "why" explanations, prediction checking |
| M3 | Animation Lead | Animation primitives, LLM trace-to-Manim pipeline, primitive coverage validation, render cache |
| M4 | Interactive Frontend Lead | Trace viewer, playback controls, drill-down, predict-next-step UI, comparison view |

**Shared interfaces** (frozen early, owned by one member each):
- Trace JSON schema — M1 owns, everyone consumes; nobody invents their own format
- Verified trace schema — M2's output, a superset of M1's raw trace
- Primitive manifest — M3 owns; M2's event names must map to it 1:1
- Render output format — M3 → M4

---

## Development Priority

1. Core simulators (B-tree, LRU)
2. Trace format
3. Trace viewer
4. Animation primitives
5. Trace → Manim
6. Cache
7. Personalization
8. Drill-down
9. Evaluation

Don't jump ahead. The basic trace pipeline has to work first.

---

## Branching

```
m1/trace-engine
m2/trace-intelligence
m3/animation
m4/frontend
```

Work on your branch, open a PR into `main`, get one approval.

---

## Design Constraints

The project deliberately avoids:

- Making the LLM the source of truth for algorithm state
- Claiming deterministic correctness without actual deterministic execution
- Claiming real-time generation where interaction requires full regeneration
- Expanding to every CS subject before the core pipeline works
- Fabricating results or capabilities that aren't implemented

If something is technically difficult, the answer is a simpler version that still demonstrates the core idea.

---

## Evaluation (Proposed)

**System performance:** trace generation latency, animation generation latency, render time, cache-hit rate, interaction response time, primitive reuse rate.

**Student learning:** pre-test/post-test, comprehension questions, ability to predict the next state, ability to explain why a transition occurred.

These are proposed methods. No results have been collected yet.

---

## What Makes TraceViz Different

Not "an AI that creates educational videos." The combination is:

**Execute → Trace → Visualize → Interact → Learn → Evaluate**

Trace-first · deterministically correct · LLM-assisted (not LLM-authored) · interactive · personalized · responsive.

---

## License

MIT
