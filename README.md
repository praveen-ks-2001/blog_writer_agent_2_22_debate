# Blog Writer Agent

An AI agent that transforms your raw brain dump into a polished, voice-consistent technical blog post. It uses a multi-step LangGraph pipeline with human-in-the-loop interrupts at key creative decision points.

## Structure

```
├── inputs/            # Your raw content (brain_dump.md, voice.md)
├── outputs/           # Agent-generated files (debate_full.md, debate_summary.md)
├── notebooks/         # Development notebook
├── assets/            # Static assets (graph.png, etc.)
├── .env               # API keys (git-ignored)
└── requirements.txt
```

## Setup

```bash
pip install -r requirements.txt
```

Create a `.env` file in the root:
```
OPENAI_API_KEY=your_key_here
ANTHROPIC_API_KEY=your_key_here
```

---

## How It Works

The agent runs as a LangGraph state machine. State is checkpointed after every node via `MemorySaver`, so you can pause, inspect, and resume at any interrupt point.

### The Pipeline

```
brain_dump → clean → reconstruct → debate_options ──► [INTERRUPT 1]
                                                             │
                                             user picks a debate topic
                                                             │
                                        debate → distill → story_arc ──► [INTERRUPT 2]
                                                                                │
                                                            user approves / rewrites arc
                                                                                │
                                                                            write ─────────────┐
                                                                              │                │
                                                                            judge ──► [INTERRUPT 3]
                                                                              │                │
                                                              user says "approve" or gives feedback
                                                                              │
                                                                         revise ──────────────►┘
                                                                              │
                                                                            END
```

### Node-by-Node Breakdown

| # | Node | Model | What it does |
|---|------|-------|--------------|
| 1 | `clean` | GPT (mini) | Fixes grammar & clarity in your brain dump. Does NOT restructure or summarize. |
| 2 | `reconstruct` | Claude | Writes a vivid first-person scene capturing the frustration/moment from your notes. Used as the cold open. |
| 3 | `debate_options` | GPT | Generates 3-4 debate topics — real tensions in the content (tradeoffs, "is this even worth it", alt approaches). **→ INTERRUPT** |
| 4 | `debate` | Claude | Two senior engineers argue casually about the topic you picked — blunt, specific, no conclusions. Saved to `outputs/debate_full.md`. |
| 5 | `distill` | GPT (mini) | Extracts 2-3 strongest points from each side + unresolved tensions. Saved to `outputs/debate_summary.md`. |
| 6 | `story_arc` | Claude | Writes a 6-8 line story flow (not headers — the emotional arc of the post). **→ INTERRUPT** |
| 7 | `write` | GPT (with memory) | Ghostwrites the full blog post using the scene, arc, debate points, and your voice guide. |
| 8 | `judge` | Claude | Ruthless editor — finds flat sections, AI-sounding prose, voice violations. Flags and suggests fixes. **→ INTERRUPT** |
| 9 | `revise` | GPT (with memory) | Applies only the feedback you gave. Does not touch unflagged sections. Loops back to `judge`. |

---

## Running the Agent

### Step 1 — Prepare inputs

Edit `inputs/brain_dump.md` with your raw notes (stream of consciousness is fine).  
Edit `inputs/voice.md` with your writing voice characteristics.

### Step 2 — Start the pipeline

```python
config = {"configurable": {"thread_id": "blog-thread-1"}}
result = app.invoke(initial_state, config=config)
```

The agent runs until the first interrupt inside `debate_options`.

### Step 3 — Inspect & resume: Debate topic selection *(Interrupt 1)*

```python
# See the generated debate topics
result["__interrupt__"][0].value["debate_options"]

# Resume by providing the topic you want (or write your own)
result = app.invoke(
    Command(resume="Your chosen debate topic here"),
    config=config
)
```

The agent runs debate → distill → story_arc, then pauses again.

### Step 4 — Inspect & resume: Story arc approval *(Interrupt 2)*

```python
# Read the generated story arc
print(result["__interrupt__"][0].value["story_arc_outline"])

# Resume: pass the arc back as-is to approve, or pass a rewritten version
result = app.invoke(
    Command(resume=result["__interrupt__"][0].value["story_arc_outline"]),
    config=config
)
```

The agent writes the full draft, judges it, then pauses again.

### Step 5 — Inspect & resume: Draft review loop *(Interrupt 3)*

```python
# Read the draft and judge feedback
print(result["__interrupt__"][0].value["current_draft"])
print(result["__interrupt__"][0].value["judge_feedback"])

# Option A: Give specific feedback → triggers a revision
result = app.invoke(
    Command(resume="Make the second paragraph sharper"),
    config=config
)

# Option B: Approve → ends the pipeline
result = app.invoke(
    Command(resume="approve"),
    config=config
)
```

The revision loop (`judge` → `revise`) repeats until you send `"approve"`.

---

## Inputs

| File | Purpose |
|---|---|
| `inputs/brain_dump.md` | Your raw notes about the topic — unfiltered |
| `inputs/voice.md` | Describes your writing voice, tone, and style rules |

## Outputs

| File | Purpose |
|---|---|
| `outputs/debate_full.md` | Full engineer debate transcript |
| `outputs/debate_summary.md` | Distilled points and tensions from the debate |
