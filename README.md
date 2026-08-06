# Enterprise Security Incident Investigator (SOC AI Analyst)

An agentic AI SOC assistant that investigates security incidents by combining threat
intelligence, SIEM logs, endpoint telemetry, identity events, vulnerability data, and
a RAG-grounded knowledge base of policies/playbooks — then produces an explainable
investigation report (timeline, root cause, severity, tools used, confidence score).

This project is a **complete, runnable reference implementation** of the spec: a mock
security database, six mock product APIs, a RAG knowledge base, an agentic
plan → execute → ground → report pipeline, and a FastAPI conversational endpoint.

## Project layout

```
soc_assistant/
├── data/
│   ├── db/
│   │   ├── build_db.py          # creates & seeds the mock SQLite security DB
│   │   └── mock_security.db     # generated (run build_db.py)
│   └── knowledge_base/          # RAG corpus (markdown)
│       ├── security_policies.md
│       ├── incident_response_playbooks.md
│       ├── mitre_attack.md
│       ├── soc_runbooks.md
│       └── compliance_guidelines.md
├── backend/
│   ├── mock_apis.py             # Identity, SIEM, Firewall, EDR, Threat Intel, Vuln Scanner
│   ├── rag.py                   # TF-IDF retrieval over the knowledge base
│   ├── agent.py                 # Planner -> Executor -> ReportSynthesizer -> SOCAgent
│   ├── app.py                   # FastAPI conversational API (/chat)
│   └── run_demo.py              # zero-server CLI demo (no fastapi/uvicorn needed)
├── requirements.txt
└── README.md
```

## Quick start

```bash
cd data/db && python3 build_db.py && cd ../../backend

# Option A — no server needed, just see it work:
python3 run_demo.py

# Option B — run the real conversational API:
pip install -r ../requirements.txt
uvicorn app:app --reload --port 8000
curl -X POST localhost:8000/chat -H "Content-Type: application/json" \
  -d '{"message": "Why was user John locked out?"}'
```

## How it satisfies each requirement

**1. Conversational Assistant** — `SOCAgent.ask()` in `agent.py` handles free-text
questions like "Why was user John locked out?", "Is this IP malicious?", "Show failed
logins today.", "Explain yesterday's ransomware alert." `app.py`'s `/chat` endpoint
keeps a `session_id` → `SOCAgent` mapping so conversation history persists across
turns (`GET /sessions/{id}`).

**2. Knowledge Base (RAG)** — `data/knowledge_base/*.md` contains Security Policies,
Incident Response Playbooks, a MITRE ATT&CK reference, SOC Runbooks, and Compliance
Guidelines. `rag.py` chunks each doc by `##` section and retrieves the most relevant
chunks per query with TF-IDF cosine similarity (fully offline; swap in a real vector
DB + embeddings model for production without touching the calling code).

**3. Mock Security APIs** — `mock_apis.py` implements `IdentityAPI`, `SIEMAPi`,
`FirewallAPI`, `EDRAPi`, `ThreatIntelAPI`, and `VulnScannerAPI`, all backed by
`data/db/mock_security.db` (SQLite), seeded with a realistic, internally-consistent
incident scenario: a brute-force lockout on `jsmith` from a malicious Tor-exit IP, and
a ransomware/exfiltration incident on host `FIN-WKS-014` tied to an unpatched CVE.

**4. Agentic Workflow** — `agent.py` implements the exact
`Identity → SIEM → Threat Intel → Playbook → Report` flow from the spec via three
stages:
- `Planner.plan()` — extracts entities (username/IP/hostname) and decides which tools
  to call and in what order, based on intent (lockout, ransomware, IP reputation,
  failed-logins report, suspicious login).
- `Executor.execute()` — calls the mock APIs, including a **dynamic pivot** step
  (`threat_intel_lookup_from_events`) that looks up threat intel for every IP/hash
  surfaced by earlier tool calls — this is the "autonomously determine which tools to
  invoke" behavior, not a hardcoded fixed chain.
- `ReportSynthesizer.synthesize()` — grounds severity/root-cause in the retrieved
  policy/playbook text and assembles the final report.

This reference implementation plans with transparent, auditable rules so every
decision is inspectable end-to-end without an API key. **To upgrade the planner and
report-writer to be LLM-driven** (recommended for production breadth beyond the 5
scripted intents), replace `Planner.plan()` and `ReportSynthesizer._assess()` with
calls to the Claude API using tool-use/function-calling — pass `TOOL_REGISTRY` from
`mock_apis.py` as the `tools` schema, let Claude choose calls, execute them, and feed
results back for the narrative. The Executor/report data model doesn't need to
change. See the "LLM-driven upgrade" section below for a ready-to-adapt snippet.

**5. Explainability** — Every `report` dict returned by `SOCAgent.ask()` includes
`timeline`, `root_cause`, `severity`, `tools_used`, `confidence_score`,
`knowledge_base_citations` (which policy/playbook grounded the call), and
`raw_evidence` (full provenance: every tool call, its parameters, and its raw result).

**Bonus items:**
- *Remediation steps* — `recommended_remediation` in every report.
- *Human approval before disabling users* — `requires_human_approval` flag, driven by
  `POL-IAM-020` in `security_policies.md`; the agent never disables an account itself,
  it only ever reads telemetry.
- *Timeline visualization* — see the interactive chat artifact (rendered separately in
  this conversation), which renders the timeline as a visual sequence.
- *Multi-agent-style separation of concerns* — `Planner` / `Executor` /
  `ReportSynthesizer` are independent, swappable components rather than one monolithic
  function, which is the seam you'd cut along to run them as separate agents/services.

## LLM-driven upgrade (production path)

```python
# In agent.py, replace Planner.plan() with something like:
import anthropic
client = anthropic.Anthropic()

TOOLS_SCHEMA = [
    {"name": "identity_get_user_events", "description": "...", "input_schema": {...}},
    # one entry per TOOL_REGISTRY key in mock_apis.py
]

def plan_with_llm(question, history):
    resp = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        system="You are a SOC investigation planner. Decide which security tools to "
               "call, in order, to answer the analyst's question.",
        messages=history + [{"role": "user", "content": question}],
        tools=TOOLS_SCHEMA,
    )
    # loop on resp.content tool_use blocks, execute via mock_apis, feed tool_result
    # back until Claude returns a final text plan/report.
```

The `Executor`, the SQLite mock database, and the RAG knowledge base require no
changes to support this — they're the same "tools" either way.

## Scalability notes
- `mock_security.db` → in production, each mock API method becomes an HTTP call to the
  real product (Splunk/Sentinel, Okta/Azure AD, Palo Alto/Fortinet,
  CrowdStrike/Defender, a TI feed, Qualys/Nessus); the `TOOL_REGISTRY` contract stays
  the same.
- `SESSIONS` (in-memory dict in `app.py`) → swap for Redis or a database table for a
  multi-instance/horizontally-scaled deployment.
- `KnowledgeBase` (TF-IDF, in-process) → swap for a vector database (pgvector, Chroma,
  Pinecone) with a real embeddings model as the policy corpus grows past a few hundred
  documents.
