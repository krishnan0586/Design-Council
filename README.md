[README.md]
# Design Council

An LLM-powered multi-agent debate system for architectural design optimisation. Specialist AI agents argue over competing design priorities — cost, carbon, and glazing — and reach a consensus on objective weights that drive a Grasshopper parametric optimisation algorithm.

<img width="2559" height="1418" alt="image (1)" src="https://github.com/user-attachments/assets/f4fce3e5-45da-46a1-ac5b-5f04d8a487df" />

---

## What it does

Traditional parametric optimisation in architecture requires designers to manually assign weights to competing objectives—how much does cost matter versus sustainability versus spatial quality? This is inherently a multi-disciplinary judgement that involves quantity surveyors, sustainability consultants, and design architects pulling in different directions.
This project replaces that manual weighting process with a structured AI debate. Three specialist agents — each loaded with their own domain documents — argue their case, challenge each other's positions, and reach a compromise. A chairperson agent synthesises the debate into a final set of objective weights, which are written directly to a file that Grasshopper reads in real time.

---

## System overview

```
Client Brief (xlsx)          Costing Information (xlsx)
Sustainability (xlsx)        Building Regulations (xlsx)
        |                               |
        v                               v
+-----------------------------------------------+
|            LLM Council (FastAPI backend)       |
|                                                |
|  Stage 1: Each agent reads its documents and  |
|           proposes weights independently       |
|                                                |
|  Stage 2: Agents see each other's arguments,  |
|           make concessions, challenge claims  |
|                                                |
|  Human:   Chairperson asks stakeholder one    |
|           focused question before deciding    |
|                                                |
|  Stage 3: Chairperson synthesises debate and  |
|           delivers final objective weights    |
+-----------------------------------------------+
        |
        v
weights_output.json + weights_output.txt
        |
        v
+-----------------------------------------------+
|           Grasshopper (Rhino 3D)              |
|                                                |
|  FileSystemWatcher reads weights in real time |
|  Fitness function: f = w1*cost + w2*carbon   |
|                        + w3*glazing           |
|  Optimiser (Galapagos/Octopus) varies         |
|  geometry to maximise/minimise fitness        |
+-----------------------------------------------+
```

---

## The three objective weights

The council debates and produces weights for three parameters that feed the optimisation algorithm:

| Parameter | Definition |
|---|---|
| `cost_gbp` | Total construction cost priority — floor plate cost (£/m²) + facade cost (£/m²) |
| `carbon_eq` | Embodied carbon minimisation — kg CO₂e/m³ across concrete, timber, and steel |
| `glazing_ratio` | Facade glazing optimisation — % of facade that is transparent vs opaque |

All weights sum to 1.0. A higher weight means the optimiser prioritises that objective more strongly.

---

## The agents

### Design Brief Agent
Loads the client brief (Excel or PDF) and advocates for faithful interpretation of the client's spatial requirements, programme, and architectural quality targets. Pushes back on cost and carbon arguments that would compromise the brief.

### Sustainability Agent
Loads BREEAM criteria and embodied carbon data. Argues for low-carbon material choices and optimal glazing ratios for daylighting performance. Cites specific kg CO₂e/m³ figures for concrete (405), timber (104), and steel (1,500).

### Cost Consultancy Agent
Loaded with benchmarked cost data — floor plate rates (£/m²) and facade costs split by glass (£1,000/m²) vs solid (£500/m²). Acts as a pragmatic QS, pushing back on sustainability and design arguments that risk breaching the budget envelope.

### Chairperson
Receives the full debate transcript and synthesises a final verdict. Before deciding, asks the human stakeholder one focused question. Produces the final weight output that is written to disk.

---

## Debate structure

**Stage 1 — First opinions**
Each agent independently reviews its documents and the project context, then proposes weights and argues its case in 100–150 words citing specific figures.

**Stage 2 — Structured debate**
Each agent sees the other agents' Stage 1 arguments. They must respond with four specific sections:
- **Concession** — one specific threshold they are willing to relax (e.g. "I accept concrete up to 30% of structural volume")
- **Challenge** — one specific claim from another agent they dispute, citing the exact figure
- **Revised weights** — updated weights reflecting the negotiation
- **Reasoning** — how the concession and challenge justify the revision

**Human input**
The chairperson asks the stakeholder one focused question based on the debate (e.g. "Is the net-zero carbon target a hard constraint or a preference?"). The stakeholder's answer is expanded into a professional instruction and factored into the final verdict.

**Stage 3 — Chairperson verdict**
The chairperson reads the condensed debate transcript and the stakeholder instruction, then outputs final weights and a 3–4 sentence rationale.

---

## Optimisation validation

After Grasshopper runs the optimisation and produces a result, the system validates the output against the original document ranges:

| Parameter | Source document |
|---|---|
| Glazing ratio | Sustainability.xlsx |
| Floor height | Building Regulations.xlsx |
| Occupancy density | Building Regulations.xlsx |
| Total area | Client Brief.xlsx |
| Total budget | Client Brief.xlsx |

If any parameter falls outside the permitted range, the chairperson explains what failed and why. The user can choose to rerun the full council debate with the validation failures injected as context — so agents adjust their weight proposals to steer the optimiser away from constraint violations.

---

## Tech stack

| Component | Technology |
|---|---|
| LLM backend | OpenAI API / Gemini API / LM Studio (local) |
| API server | FastAPI + Uvicorn |
| Agent framework | Custom multi-agent loop (no LangChain) |
| Document parsing | openpyxl (Excel), PyMuPDF (PDF) |
| Frontend | Next.js 16, React 19, Tailwind CSS |
| Real-time streaming | Server-Sent Events (SSE) |
| Grasshopper bridge | FileSystemWatcher (C#) or GHPython |
| Parametric software | Rhino 3D + Grasshopper |

---

## Project structure

```
project/
├── council_backend/               # FastAPI backend
│   ├── main.py                    # API endpoints + SSE streaming
│   ├── council.py                 # Agent logic, debate orchestration
│   └── requirements.txt
│
└── frontend/                      # Next.js frontend
    ├── app/
    │   ├── layout.tsx             # Wraps app in CouncilProvider
    │   └── page.tsx
    ├── components/
    │   ├── chat/                  # Chat interface
    │   ├── roundtable/            # Agent visualisation + weight cards
    │   └── workspace/             # Three-panel layout shell
    ├── lib/
    │   ├── api.ts                 # All fetch + SSE calls to backend
    │   └── hooks/
    │       ├── useChat.ts         # Chat state + pipeline orchestration
    │       └── useCouncilStore.tsx # Global agent state (React context)
    └── .env.local                 # API keys + backend URL
```

---

## Setup

### Prerequisites
- Python 3.10+
- Node.js 18+
- Rhino 3D + Grasshopper (for the optimisation step)
- An API key for OpenAI, Gemini, or a local LM Studio install

### 1. Backend

```bash
cd council_backend
pip install -r requirements.txt
python -m uvicorn main:app --reload --port 8000
```

### 2. Frontend

```bash
cd frontend
npm install
npm run dev
```

Open `http://localhost:3001` (or whichever port Next.js assigns).

### 3. Configure your API provider

In `frontend/.env.local`:

```
# Gemini
NEXT_PUBLIC_GEMINI_API_KEY=AIza...

# or OpenAI
NEXT_PUBLIC_OPENAI_API_KEY=sk-...

NEXT_PUBLIC_API_URL=http://localhost:8000
```

In `council_backend/council.py`, set the provider:

```python
# OpenAI
MODEL = "gpt-4o"
def _make_client(api_key): return OpenAI(api_key=api_key)

# Gemini
MODEL = "gemini-2.0-flash"
def _make_client(api_key): return OpenAI(
    api_key=api_key,
    base_url="https://generativelanguage.googleapis.com/v1beta/openai/",
)

# LM Studio (local)
MODEL = "your-local-model-name"
def _make_client(api_key): return OpenAI(api_key="lm-studio", base_url="http://localhost:1234/v1")
```

### 4. Input files

Place your project documents in a folder. The frontend will ask you to select the `Client Brief.xlsx` — the other three files are expected in the same folder:

```
Input/
├── Client Brief.xlsx          # Project type, area range, budget range
├── Costing Information.xlsx   # Floor plate costs, facade costs, material costs
├── Sustainability.xlsx        # Glazing ratios, embodied carbon per material
└── Building Regulations.xlsx  # Floor height range, occupancy range
```

**Excel format** — each file uses a simple two-column key/value layout with an optional unit column:

```
| project_type        | Grade A Commercial Office |        |
| min_area            | 23000                     | m2     |
| max_area            | 25300                     | m2     |
| min_budget          | 75000000                  | £      |
| max_budget          | 82500000                  | £      |
```

### 5. Grasshopper bridge

In a GHPython component, point to the output text file:

```python
import os

# Input: txt_path (string)
# Outputs: cost_gbp, carbon_eq, glazing_ratio

if not txt_path or not os.path.exists(txt_path):
    cost_gbp = carbon_eq = glazing_ratio = 0.0
else:
    weights = {}
    with open(txt_path, "r") as f:
        for line in f:
            if "=" in line:
                key, val = line.strip().split("=", 1)
                weights[key.strip()] = float(val.strip())

    cost_gbp      = weights.get("cost_gbp",      0.0)
    carbon_eq     = weights.get("carbon_eq",     0.0)
    glazing_ratio = weights.get("glazing_ratio", 0.0)
```

Connect a Timer component to poll the file. Wire the three weight outputs into your fitness expression:

```
fitness = (cost_score * cost_gbp) + (carbon_score * carbon_eq) + (glazing_score * glazing_ratio)
```

---

## Chat commands

Once the frontend is running, the chat interface accepts these commands:

| Command | Action |
|---|---|
| `start` | Initialise and run the full council (Stage 1 + 2 + Chairperson) |
| `skip` | Skip human input, let chairperson decide independently |
| `continue` | Pass your human input to the chairperson without rerunning the debate |
| `rerun` | Restart the debate with current context injected into agent prompts |
| `validate` | Validate optimisation results against document ranges |
| `summary` | Generate a final project summary |
| `status` | Show current pipeline state |

Any other message is routed to the LLM as a general question about the project.

---

## Inspiration

The multi-agent debate structure is inspired by [Andrej Karpathy's llm-council](https://github.com/karpathy/llm-council), adapted here for a domain-specific architectural design context where each agent represents a professional discipline rather than a different LLM provider.

---

## Developed at

ATN AEC Hackathon — Architecture, Engineering & Construction AI Challenge
