# Career Roadmap — Interview Prep

A structured 30-week self-study system for senior frontend engineering interviews, built around a **brain-learning model** with deep focus phases, continuous DSA practice, and AI engineering as a key differentiator.

---

## About

**Candidate:** Mriganka Sekhar Sarkar  
**Role target:** Senior Frontend Engineer | Frontend Architect | Tech Lead | AI-Augmented Engineering  
**Experience:** 10 years — Angular, React, TypeScript, Node.js, AI/Gen-AI  
**Current role:** Senior Software Developer (UI Tech Lead) at Boeing India

> Full career context is in [`career_context.md`](./career_context.md).

---

## Repository Structure

```
career-roadmap/
├── career_context.md                        # Full profile, skills, target companies, prep phases
├── generate_schedule.py                     # Generates the 30-week Excel tracker
├── adjust_schedule.py                       # Shifts future rows forward for missed days
├── interview_prep_tracker.xlsx              # Generated tracker (open in Excel / Google Sheets)
├── 6_month_interview_prep_schedule.xlsx     # Legacy schedule artifact
└── lessons/
    └── javascript/
        ├── 01-execution-context-call-stack.md
        ├── 02-scope-chain-lexical-scope-hoisting.md
        └── 02-web-workers.md
```

---

## The 30-Week Learning Plan

### Phase Structure

| Phase | Weeks | Topic |
|---|---|---|
| 1 | 1–4 | JavaScript (deep dive) |
| 2 | 5–6 | TypeScript |
| 3 | 7–12 | Angular (Components → RxJS → NgRx → SSR → Testing) |
| 4 | 13–16 | Frontend System Design |
| 5 | 17–21 | Backend + HTTP + Security |
| 6 | 22–26 | AI Engineering |
| 7 | 27–30 | Mock Loops + Revision + Apply |

### Weekly Pattern (Brain-Learning Model)

| Day | Focus | Hours |
|---|---|---|
| Monday | Core topic — subtopic A | 3 |
| Tuesday | Core topic — subtopic B | 3 |
| Wednesday | Core topic — subtopic C + DSA problem | 3 |
| Thursday | Core topic — subtopic D + System Design / AI sprinkle | 3 |
| Friday | Core coding questions + LeetCode + Behavioral | 3 |
| Saturday | Mock Interview + Revision + Project | 5 |
| Sunday | Reading + Portfolio + Rest | 2 |

Three parallel tracks run throughout all 30 weeks:
- **DSA** — 3× per week, progressing from fundamentals to timed mock sets
- **LeetCode** — 2× per week, progressing from mediums to timed hards
- **Behavioral** — 1× per week, STAR stories and mock rounds

---

## Lessons

Each lesson follows a consistent 19-section structure:

> Introduction → Why It Exists → Fundamentals → Internal Working → Visual Architecture → Syntax & API → Practical Examples → Real-world Usage → Performance → Common Mistakes → Best Practices → Debugging → Advanced Concepts → Interview Questions → Hands-on Exercises → Mini Project → Revision → Cheat Sheet → References

### JavaScript

| # | Topic | Phase | Week |
|---|---|---|---|
| 01 | [Execution Context, Call Stack & Variable Environment](./lessons/javascript/01-execution-context-call-stack.md) | JS | Week 1 |
| 02 | [Scope Chain, Lexical Scope & Hoisting](./lessons/javascript/02-scope-chain-lexical-scope-hoisting.md) | JS | Week 1 |
| 03 | [Microtasks vs Macrotasks (Event Loop Deep Dive)](./lessons/javascript/03-microtasks-vs-macrotasks.md) | JS | Week 3 |
| 04 | [Web Workers](./lessons/javascript/02-web-workers.md) | JS | Week 4 |

More lessons are added weekly as the schedule progresses.

---

## Tooling

### `generate_schedule.py`

Generates a colour-coded Excel workbook with 5 sheets:

- **Dashboard** — live stats
- **Master Roadmap** — all lessons ordered
- **Daily Planner** — day-by-day tracker (main sheet)
- **Revision Tracker** — spaced repetition (1d / 7d / 30d / mastered)
- **Mock Interviews** — log with scores
- **Behavioral Stories** — STAR story tracker

```bash
pip install openpyxl
python generate_schedule.py
```

Output: `interview_prep_tracker.xlsx`

### `adjust_schedule.py`

Detects days in the past with no "Done" entry (gap days) and shifts all future unfinished rows forward, reassigning dates and day names.

```bash
python adjust_schedule.py
```

Useful for re-syncing the tracker after a vacation, illness, or busy sprint at work.

---

## Target Companies

| Tier | Companies |
|---|---|
| Tier 1 (Heavy DSA) | Google, Microsoft |
| Tier 2 (Balanced) | Atlassian, Adobe, Intuit, Salesforce, ServiceNow, LinkedIn, Uber |
| Tier 3 (Indian Product) | Flipkart, PhonePe, Razorpay, Walmart Global Tech |

---

## Key Differentiators

1. **AI Engineering experience** — MCP, LLM workflows, Claude integration at Boeing; building `convopro_chatgpt` (Streamlit + Ollama + LlamaIndex + MongoDB).
2. **Enterprise scale** — led AngularJS → Angular 12 migration across 25 engineers.
3. **TypeScript leadership** — defined org-wide TypeScript design practices adopted across multiple teams at Boeing.

---

## Progress Milestones

| Month | Target |
|---|---|
| 1 | JS + Angular foundation solid, first 25 LeetCode problems |
| 2 | 50 LeetCode done, AI project deployed / demo-able |
| 3 | System design comfortable, backend revision starts, begin applying |
| 4 | 100+ LeetCode, backend done, 3–5 applications/week |
| 5 | Active interviewing, 2 full mock loops done |
| 6 | Offer received 🎉 |
