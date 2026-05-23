# Architectural Synthesis: CORTEX Cognitive OS

This document proposes a unified, production-grade software architecture named **CORTEX (Cognitive Operating Runtime and Tool Execution engine)**. CORTEX extracts, refines, and combines the strongest concepts from the 8 surveyed AI system proposals into a concrete, implementation-ready design.

```
+---------------------------------------------------------------------------------+
|                                 CORTEX RUNTIME                                  |
|                                                                                 |
|   +-------------------+     +-------------------------+     +---------------+   |
|   |   Input Stream    | --> |  System 1 Parser (vLLM) | --> |   Task Queue  |   |
|   +-------------------+     +-------------------------+     +---------------+   |
|                                                                     |           |
|                                                                     v           |
|   +-------------------+     +-------------------------+     +---------------+   |
|   |  Causal Network   | <-- | System 2 Planner (MCTS) | <-- | Plan Executor |   |
|   |  (World Model)    |     +-------------------------+     +---------------+   |
|   +-------------------+                                             |           |
|                                                                     v           |
|   +-------------------+     +-------------------------+     +---------------+   |
|   | Cryptographic Log | <-- | Safety Gate (LlamaGuard)| --> | Tool Sandbox  |   |
|   |  (pg_audit_ledger)|     +-------------------------+     | (MicroVM/LXD) |   |
|   +-------------------+                                     +---------------+   |
|                                                                                 |
+---------------------------------------------------------------------------------+
```

## 1. Concrete System Specifications

### 1.1 Memory Architecture (Hybrid Context/Index Store)
CORTEX rejects pure RAG and pure large-context storage. It implements a **Hybrid sliding-window context with transactional state indexing**:
* **Active Execution Context:** Up to 128k tokens containing the recent conversation, system state, execution traces, and active workspace files.
* **Vector Semantic Store:** ChromaDB with hierarchical document indexing, using query expansion.
* **Factual & Invariant Ledger:** Relational PostgreSQL schemas representing system settings, tool schemas, and workspace structures.

```sql
CREATE TABLE agent_state (
    session_id UUID PRIMARY KEY,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    context_tokens INT[],
    world_state JSONB NOT NULL
);

CREATE TABLE system_audit_ledger (
    entry_id BIGSERIAL PRIMARY KEY,
    timestamp TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    session_id UUID NOT NULL,
    action_type VARCHAR(50) NOT NULL,
    action_payload JSONB NOT NULL,
    previous_hash BYTEA NOT NULL,
    entry_hash BYTEA NOT NULL
);
```

### 1.2 Reasoning and Planning Loop
* **System 1 (Reflexive Mode):** Direct generation of structured JSON steps for simple, high-confidence operations (confidence > 0.85).
* **System 2 (Verification/Search Mode):** Monte Carlo Tree Search (MCTS) combined with Tree-of-Thought (ToT) when confidence is low or safety-critical invariants are involved.
* **Self-Correction:** Any parser or validation errors automatically trigger a correction step, sending the error trace and schema requirements back to the system planner.

### 1.3 Safe Tool Execution Sandbox
Tools are written as structured Python modules and executed inside ephemeral **LXD containers** or **Firecracker MicroVMs** with strict network egress policies.
* **Egress Masking Layer:** A mandatory out-of-band proxy parses outgoing data, masking PII and checking against security blacklists before dispatch.

```python
import subprocess
import json

def execute_sandbox_tool(container_id: str, command: list[str]) -> dict:
    # Restrict cpu and memory usage on runtime
    prefix = ["lxc", "exec", container_id, "--", "sudo", "-u", "sandbox"]
    full_cmd = prefix + command
    try:
        res = subprocess.run(full_cmd, capture_output=True, text=True, timeout=10)
        return {
            "exit_code": res.returncode,
            "stdout": res.stdout,
            "stderr": res.stderr
        }
    except subprocess.TimeoutExpired:
        return {
            "exit_code": -1,
            "stdout": "",
            "stderr": "Execution timed out."
        }
```

### 1.4 Cryptographic Audit Trails & Safety Invariants
To prevent the agent from mutating its own history or disabling security checks, CORTEX logs all state transitions and outputs to an append-only table (`system_audit_ledger`). Each record is cryptographically signed using SHA-256 containing the hash of the previous record, ensuring absolute auditability.

### 1.5 Local Policy Self-Improvement
CORTEX logs failed tasks to a local dataset. Once every 24 hours, a background thread compiles these traces and executes a local **LoRA fine-tuning** process (using PyTorch and Llama Stack APIs) to adjust reasoning weights and correct repeated failure modes without updating external APIs.
