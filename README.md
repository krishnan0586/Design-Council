# Design Council

An LLM-powered multi-agent debate system for architectural design optimisation. Specialist AI agents argue over competing design priorities, cost, carbon, and glazing, and reach a consensus on objective weights that drive a Grasshopper parametric optimisation algorithm.

<img width="2559" height="1418" alt="Design Council UI" src="https://github.com/user-attachments/assets/f4fce3e5-45da-46a1-ac5b-5f04d8a487df"/>

---

## What it does

Traditional parametric optimisation in architecture requires designers to manually assign weights to competing objectives, how much does cost matter versus sustainability versus spatial quality? This is inherently a multi-disciplinary judgement that involves quantity surveyors, sustainability consultants, and design architects pulling in different directions.

This project replaces that manual weighting process with a structured AI debate. Three specialist agents, each loaded with their own domain documents, argue their case, challenge each other's positions, and reach a compromise. A chairperson agent synthesises the debate into a final set of objective weights, which are written directly to a file that Grasshopper reads in real time.

---

## Multi-Agent Pipeline
<img width="1240" height="1754" alt="Design Council_Workflow Diagram" src="https://github.com/user-attachments/assets/a2ef5d90-2acb-43fb-a52e-c41ca59dd6b4"/>

---

## Sample Scenario - Three Objectives

The council debates and produces weights for three parameters that feed the optimisation algorithm:

| Parameter | Definition |
|---|---|
| `cost_gbp` | Total construction cost priority, floor plate cost (£/m²) + facade cost (£/m²) |
| `carbon_eq` | Embodied carbon minimisation, kg CO₂e/m³ across concrete, timber, and steel |
| `glazing_ratio` | Facade glazing optimisation, % of facade that is transparent vs opaque |

All weights sum to 1.0. A higher weight means the optimiser prioritises that objective more strongly.

---

## The Agents

### Design Brief Agent
Loads the client brief (Excel or PDF) and advocates for faithful interpretation of the client's spatial requirements, programme, and architectural quality targets. Pushes back on cost and carbon arguments that would compromise the brief.

### Sustainability Agent
Loads BREEAM criteria and embodied carbon data. Argues for low-carbon material choices and optimal glazing ratios for daylighting performance. Cites specific kg CO₂e/m³ figures for concrete (405), timber (104), and steel (1,500).

### Cost Consultancy Agent
Loaded with benchmarked cost data, floor plate rates (£/m²) and facade costs split by glass (£1,000/m²) vs solid (£500/m²). Acts as a pragmatic QS, pushing back on sustainability and design arguments that risk breaching the budget envelope.

### Chairperson
Receives the full debate transcript and synthesises a final verdict. Before deciding, asks the human stakeholder one focused question. Produces the final weight output that is written to disk.

---

## Processing Structure

**Stage 1, First opinions**
Each agent independently reviews its documents and the project context, then proposes weights and argues its case in 100–150 words citing specific figures.

**Stage 2, Structured debate**
Each agent sees the other agents' Stage 1 arguments. They must respond with four specific sections:
- **Concession**, one specific threshold they are willing to relax (e.g. "I accept concrete up to 30% of structural volume")
- **Challenge**, one specific claim from another agent they dispute, citing the exact figure
- **Revised weights**, updated weights reflecting the negotiation
- **Reasoning**, how the concession and challenge justify the revision

**Human input**
The chairperson asks the stakeholder one focused question based on the debate (e.g. "Is the net-zero carbon target a hard constraint or a preference?"). The stakeholder's answer is expanded into a professional instruction and factored into the final verdict.

**Stage 3, Chairperson verdict**
The chairperson reads the condensed debate transcript and the stakeholder instruction, then outputs final weights and a 3–4 sentence rationale.

---

## Optimisation Validation

After Grasshopper runs the optimisation and produces a result, the system validates the output against the original document ranges:

| Parameter | Source document |
|---|---|
| Glazing ratio | Sustainability.xlsx |
| Floor height | Building Regulations.xlsx |
| Occupancy density | Building Regulations.xlsx |
| Total area | Client Brief.xlsx |
| Total budget | Client Brief.xlsx |

If any parameter falls outside the permitted range, the chairperson explains what failed and why. The user can choose to rerun the full council debate with the validation failures injected as context, so agents adjust their weight proposals to steer the optimiser away from constraint violations.

---

## Tech Stack

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

## Project Structure

```
Cell 1, Install & imports
Cell 2, Config: API key, model, output path
Cell 3, Document parsers (PDF + Excel)
Cell 4, BaseAgent class
Cell 5, Four specialist agents
Cell 6, chairperson Agent
Cell 7, Council orchestrator (Stage 1 → Stage 2 → chairperson)
Cell 8, Run the council (streaming transcript)
Cell 9, Inspect output + write weights_output.json
```

---

## Setup

### Prerequisites
- Python 3.10+
- Node.js 18+
- Rhino 3D + Grasshopper (for the optimisation step)
- An API key for OpenAI, Gemini, or a local LM Studio install

### 1. Input files
```
Input/
├── Client Brief.xlsx          # Project type, area range, budget range
├── Costing Information.xlsx   # Floor plate costs, facade costs, material costs
├── Sustainability.xlsx        # Glazing ratios, embodied carbon per material
└── Building Regulations.xlsx  # Floor height range, occupancy range
```

### 2. Output files
```
Output/
├── Design Council_Workflow Diagram.jpg            # Diagram image for reference
├── Optimization Results.xlsx                      # Final results from Grasshopper after optimisations
├── Weights_Input.gh                               # Grasshopper script (sync)
└── weights_output.json                            # Weights prescribed by the council (JSON)
└── weights_output.txt                             # Weights prescribed by the council (TXT)
```

### 3. Working files
```
Working/
└── Design Council.ipynb      # Notebook file for local runs
```

## Inspiration

The multi-agent debate structure is inspired by [Andrej Karpathy's llm-council](https://github.com/karpathy/llm-council), adapted here for a domain-specific architectural design context where each agent represents a professional discipline rather than a different LLM provider.

---

## Developed at 
ATN AEC Hackathon London - March 2026
Heatherwick Studios, London
https://www.aectech.us/london-hackathon

Access all projects here , 
https://www.aectech.us/hackathon-archive
