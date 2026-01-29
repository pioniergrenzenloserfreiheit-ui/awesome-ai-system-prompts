You’ve already got a solid spine here: clear intent, the two adaptive engines (evolution + personalization), and a practical phase workflow with gates. What’s missing is mostly the “hard governance glue” that makes this spec compile cleanly into an auditable, contract-driven platform: a universal envelope, decision/rollback primitives, telemetry/audit file integrity, and the workflow DAG / policy packs you reference but don’t define yet.

Below are copy-ready “drop-in” additions + a few targeted schema upgrades. (I’m not rewriting your whole doc; I’m giving you the missing load-bearing beams.)

---

## Patch set v1.0.1 (drop-in additions)

### A) Add a normative language block (MUST/SHOULD/MAY)

Insert near the top (after the intro):

**Normative keywords:** The terms **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** are used as normative requirements. If a requirement conflicts with a consent boundary, **consent wins** and the system **fails closed**.

---

### B) Add a universal contract envelope (every payload goes inside this)

Insert as new section “6.0 Contract Envelope (Required)”:

#### `aurora.contract_envelope.v1`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "aurora.contract_envelope.v1",
  "title": "AURORA Contract Envelope v1",
  "type": "object",
  "additionalProperties": false,
  "required": ["type", "version", "timestamp", "trace", "actor", "payload"],
  "properties": {
    "type": { "type": "string", "minLength": 3 },
    "version": { "type": "string", "pattern": "^v\\d+\\.(\\d+)\\.(\\d+)$" },
    "timestamp": { "type": "string", "format": "date-time" },
    "trace": {
      "type": "object",
      "additionalProperties": false,
      "required": ["trace_id"],
      "properties": {
        "trace_id": { "type": "string", "minLength": 8 },
        "correlation_id": { "type": "string" },
        "parent_trace_id": { "type": "string" }
      }
    },
    "actor": {
      "type": "object",
      "additionalProperties": false,
      "required": ["actor_type", "actor_id"],
      "properties": {
        "actor_type": { "type": "string", "enum": ["user", "agent", "service_account"] },
        "actor_id": { "type": "string", "description": "Pseudonymous identifier; MUST NOT be raw PII." },
        "role": { "type": "string" }
      }
    },
    "policy_refs": {
      "type": "array",
      "items": { "type": "string" },
      "description": "Optional: references to policy pack IDs used in this run."
    },
    "payload": { "type": "object" }
  }
}
```

Operational rule (add as text): every API/tool boundary accepts and emits the envelope; the `payload` MUST validate against its referenced schema.

---

### C) Add decision + rollback primitives (this makes governance real)

Insert as new “8.0 Decision, Escalation, Rollback (Required)”:

#### `aurora.decision_record.v1`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "aurora.decision_record.v1",
  "title": "AURORA Decision Record v1",
  "type": "object",
  "additionalProperties": false,
  "required": ["decision_id", "trace_id", "outcome", "gates", "reason", "timestamp"],
  "properties": {
    "decision_id": { "type": "string" },
    "trace_id": { "type": "string" },
    "outcome": { "type": "string", "enum": ["Continue", "Rework", "Halt", "Escalate"] },
    "reason": { "type": "string", "minLength": 3 },
    "blocked_by": { "type": "array", "items": { "type": "string" } },
    "gates": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["gate", "status"],
        "properties": {
          "gate": { "type": "string", "enum": ["structural", "compliance", "security", "output"] },
          "status": { "type": "string", "enum": ["pass", "fail", "skipped"] },
          "evidence_refs": { "type": "array", "items": { "type": "string" } }
        }
      }
    },
    "escalation": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "required": { "type": "boolean" },
        "to": { "type": "string", "enum": ["human_review", "compliance_officer", "security_auditor"] },
        "notes": { "type": "string" }
      }
    },
    "rollback": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "required": { "type": "boolean" },
        "pointer_ref": { "type": "string" }
      }
    },
    "timestamp": { "type": "string", "format": "date-time" }
  }
}
```

#### `aurora.rollback_pointer.v1`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "aurora.rollback_pointer.v1",
  "title": "AURORA Rollback Pointer v1",
  "type": "object",
  "additionalProperties": false,
  "required": ["pointer_id", "trace_id", "artifacts", "created_at"],
  "properties": {
    "pointer_id": { "type": "string" },
    "trace_id": { "type": "string" },
    "artifacts": {
      "type": "array",
      "minItems": 1,
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["artifact_id", "schema_id", "sha256"],
        "properties": {
          "artifact_id": { "type": "string" },
          "schema_id": { "type": "string" },
          "sha256": { "type": "string", "pattern": "^[a-fA-F0-9]{64}$" },
          "uri": { "type": "string", "format": "uri" }
        }
      }
    },
    "created_at": { "type": "string", "format": "date-time" }
  }
}
```

Text rule (add): any “Halt” outcome MUST provide a rollback pointer (or explicitly declare “no changes were made”).

---

### D) Add workflow DAG (you reference orchestration but only define per-step)

Insert as new schema near your orchestration section:

#### `aurora.workflow_dag.v1`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "aurora.workflow_dag.v1",
  "title": "AURORA Workflow DAG v1",
  "type": "object",
  "additionalProperties": false,
  "required": ["dag_id", "version", "entrypoint", "nodes", "edges"],
  "properties": {
    "dag_id": { "type": "string" },
    "version": { "type": "string", "pattern": "^v\\d+\\.(\\d+)\\.(\\d+)$" },
    "entrypoint": { "type": "string" },
    "nodes": {
      "type": "array",
      "minItems": 1,
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["node_id", "node_type", "contract_ref"],
        "properties": {
          "node_id": { "type": "string" },
          "node_type": {
            "type": "string",
            "enum": ["intake", "ingest", "preprocess", "analyze", "decide", "personalize", "deliver", "learn", "audit"]
          },
          "contract_ref": { "type": "string", "description": "Schema ID expected for node input/output." },
          "requires_scopes": { "type": "array", "items": { "type": "string" } },
          "timeouts_ms": { "type": "integer", "minimum": 0 },
          "retry": {
            "type": "object",
            "additionalProperties": false,
            "properties": {
              "max_attempts": { "type": "integer", "minimum": 0 },
              "backoff_ms": { "type": "integer", "minimum": 0 }
            }
          }
        }
      }
    },
    "edges": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["from", "to"],
        "properties": {
          "from": { "type": "string" },
          "to": { "type": "string" },
          "condition": { "type": "string", "description": "Optional branching predicate; evaluated by Decision Agent." }
        }
      }
    }
  }
}
```

---

### E) Tighten audit + telemetry (so “audit-first delivery” is enforceable)

You have `aurora.audit_event.v1`, but it’s missing file integrity and schema references. Add these two schemas:

#### `aurora.audit_log_entry.v1`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "aurora.audit_log_entry.v1",
  "title": "AURORA Audit Log Entry v1",
  "type": "object",
  "additionalProperties": false,
  "required": ["event_id", "trace_id", "timestamp", "artifact", "status"],
  "properties": {
    "event_id": { "type": "string" },
    "trace_id": { "type": "string" },
    "timestamp": { "type": "string", "format": "date-time" },
    "artifact": {
      "type": "object",
      "additionalProperties": false,
      "required": ["artifact_id", "schema_id", "sha256"],
      "properties": {
        "artifact_id": { "type": "string" },
        "schema_id": { "type": "string" },
        "sha256": { "type": "string", "pattern": "^[a-fA-F0-9]{64}$" },
        "uri": { "type": "string", "format": "uri" }
      }
    },
    "status": { "type": "string", "enum": ["created", "validated", "blocked", "delivered", "rolled_back"] },
    "notes": { "type": "string" }
  }
}
```

#### `aurora.telemetry_event.v1`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "aurora.telemetry_event.v1",
  "title": "AURORA Telemetry Event v1",
  "type": "object",
  "additionalProperties": false,
  "required": ["event_id", "trace_id", "timestamp", "metrics"],
  "properties": {
    "event_id": { "type": "string" },
    "trace_id": { "type": "string" },
    "timestamp": { "type": "string", "format": "date-time" },
    "metrics": {
      "type": "object",
      "additionalProperties": { "type": "number" },
      "description": "Example keys: latency_ms_p95, error_rate, audit_coverage, pii_risk_score, cost_units."
    },
    "labels": {
      "type": "object",
      "additionalProperties": { "type": "string" }
    }
  }
}
```

Text rule (add): delivery MUST be blocked if required `audit_log_entry` emission fails.

---

### F) Make error handling machine-real: add an error taxonomy

You already have `aurora.error_response.v1`; it needs a canonical registry for codes + severities.

#### `aurora.error_taxonomy.v1`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "aurora.error_taxonomy.v1",
  "title": "AURORA Error Taxonomy v1",
  "type": "object",
  "additionalProperties": false,
  "required": ["version", "codes"],
  "properties": {
    "version": { "type": "string", "pattern": "^v\\d+\\.(\\d+)\\.(\\d+)$" },
    "codes": {
      "type": "array",
      "minItems": 1,
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["code", "severity", "recoverable"],
        "properties": {
          "code": { "type": "string", "pattern": "^[A-Z0-9_]+$" },
          "severity": { "type": "string", "enum": ["info", "warning", "critical"] },
          "recoverable": { "type": "boolean" },
          "default_next_action": { "type": "string" }
        }
      }
    }
  }
}
```

Recommended minimum code set (add as text): `CONSENT_REQUIRED`, `SCHEMA_VERSION_MISMATCH`, `IMAGE_INTEGRITY_FAILURE`, `NO_COMPLIANT_PIPELINE`, `SCOPE_MISSING`, `PII_RISK_BLOCKED`, `MODEL_POLICY_BLOCKED`, `AUDIT_PERSIST_FAILED`, `HUMAN_REVIEW_REQUIRED`.

---

### G) Two small schema upgrades you should apply to existing ones

These are “silent foot-guns” that will matter the moment you integrate.

1. Add `"$schema": "https://json-schema.org/draft/2020-12/schema"` and `"additionalProperties": false` to every schema you’ve defined. This makes validation deterministic.

2. Upgrade `aurora.image_input.v1` checksum from a bare string to a structured checksum (algorithm + value), and restrict URI to actual URI format:

```json
{
  "$id": "aurora.image_input.v1",
  "type": "object",
  "additionalProperties": false,
  "required": ["image_id", "uri", "format", "checksum"],
  "properties": {
    "image_id": { "type": "string" },
    "uri": { "type": "string", "format": "uri" },
    "format": { "type": "string" },
    "checksum": {
      "type": "object",
      "additionalProperties": false,
      "required": ["alg", "value"],
      "properties": {
        "alg": { "type": "string", "enum": ["sha256"] },
        "value": { "type": "string", "pattern": "^[a-fA-F0-9]{64}$" }
      }
    },
    "metadata": { "type": "object" }
  }
}
```

---

## Optional but high-value next patch (keeps evolution/personalization safe)

If you want the adaptive parts to stay tame under pressure, add two constraints as text:

* **Evolution constraint:** evolutionary mutation is only allowed on “safe knobs” (thresholds, ordering, model choice within allowlist). Anything affecting consent/retention/redaction is policy-owned and not evolvable.
* **Personalization constraint:** “preference memory with decay” MUST be implementable as (timestamp + TTL + revocation), and the personalization engine MUST be able to run in stateless mode when consent is absent.

(If you want, I can provide `aurora.preference_memory.v1` and `aurora.policy_pack.v1` schemas in the same style.)

Roter Faden: You’re turning a strong narrative spec into a contract-strict, auditable, evolvable platform blueprint.
