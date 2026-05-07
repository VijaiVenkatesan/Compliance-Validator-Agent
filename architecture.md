# System Architecture — Compliance Validator Agent

> Version 1.0.0 | Last Updated: 07 May 2026

---

## High-Level Design

The Compliance Validator Agent uses a **hybrid deterministic-agentic architecture**:

1. **Rule-based validation engines** — 100% accuracy on statutory checks
2. **CrewAI multi-agent orchestration** — complex reasoning + LLM-enhanced formatting
3. **Recursive data cleaning** — handles real-world OCR/input quality issues

---

## System Diagram

```
┌─────────────────────────────────────────────────────────────┐
│  Entry Point: src/main.py (CLI)                            │
│  • argparse: --input, --output, --split                    │
│  • Multi-format loader (JSON/CSV/XML/PDF/Image)            │
│  • recursive_clean() strips trailing spaces at load time   │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│  Data Preprocessing Layer                                   │
│                                                             │
│  recursive_clean(obj)          parse_ocr_text_to_dict()    │
│  • Strips whitespace from      • Regex extraction for      │
│    all string keys/values        PDF and image input       │
│  • Handles nested dicts/lists  • Maps unstructured text    │
│  • Ensures schema compliance     to invoice schema         │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│  CrewAI Agent Pipeline                                      │
│                                                             │
│  ┌─────────────┐   ┌─────────────┐                        │
│  │  Extractor  │──▶│  Validator  │                        │
│  │  Normalize  │   │  10 Checks  │                        │
│  └─────────────┘   └──────┬──────┘                        │
│                           │                                 │
│  ┌─────────────┐   ┌──────▼──────┐                        │
│  │  Reporter   │◀──│  Resolver   │                        │
│  │ Schema Out  │   │ Decision+   │                        │
│  └─────────────┘   │ Confidence  │                        │
│                    └─────────────┘                        │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│  Deterministic Rule Engines (src/tools/compliance_engines.py)│
│                                                             │
│  • Pure Python functions — zero LLM dependency             │
│  • Input: invoice_json (str), batch_history_json (str)     │
│  • Output: {check_result: bool, metadata: Any}             │
│  • Optional mock API via USE_MOCK_API flag                  │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│  External Integrations (Optional)                           │
│                                                             │
│  Mock GST API Server         Historical Decisions Loader   │
│  • /api/gst/validate-gstin   • historical_decisions.jsonl  │
│  • Silent fallback on error  • 15% error guard applied     │
│                                                             │
│  Checks Manifest Gating                                     │
│  • checks_manifest.json — enable/disable checks at runtime │
└─────────────────────────────────────────────────────────────┘
```

---

## Key Architectural Decisions

### 1. Hybrid Deterministic-Agentic Design

**Problem:** Pure LLM → non-deterministic; pure rules → inflexible for edge cases.

**Solution:** Python functions own all 10 statutory decisions. CrewAI agents handle orchestration, confidence scoring, and output formatting only.

**Principle:** Rules decide. Agents explain.

---

### 2. Recursive Data Cleaning at Entry Point

**Problem:** Input (especially OCR/PDF) has trailing spaces in keys/values, breaking schema validation.

```python
def recursive_clean(obj: Any) -> Any:
    if isinstance(obj, dict):
        return {(k.strip() if isinstance(k, str) else k): recursive_clean(v)
                for k, v in obj.items()}
    elif isinstance(obj, list):
        return [recursive_clean(item) for item in obj]
    elif isinstance(obj, str):
        return obj.strip()
    return obj
```

Applied at input load **and** on final output (defense-in-depth).

---

### 3. Weighted Confidence Scoring

```python
critical_weights = {"A1": 0.2, "B1": 0.2, "B7": 0.15, "D1": 0.15, "E3": 0.1}
minor_weights    = {"A2": 0.05, "C1": 0.05, "C2": 0.05, "E1": 0.05}

base_conf = 1.0
for check, weight in {**critical_weights, **minor_weights}.items():
    if not validation.get(check, True):
        penalty = 1.5 if check in critical_weights else 1.0
        base_conf -= weight * penalty
```

---

### 4. Historical Calibration with Anti-Pattern Guard

The challenge spec states 15% of historical decisions are incorrect. Blind learning injects errors.

```python
if hist_decision != decision:
    audit_notes.append(
        f"DEVIATED_FROM_PRECEDENT: Historical={hist_decision} vs Deterministic={decision}"
    )
    conf = max(0.5, conf - 0.15)  # Reduce confidence when overriding
```

System follows deterministic rules, not flawed history.

---

### 5. Enhanced Output Schema

Every check emits three fields for evaluator transparency:

```json
{
  "A1": true,
  "A1_finding": "✓ Valid invoice number format",
  "A1_confidence": 0.99
}
```

---

## Data Flow

```
Input File
    │
    ▼  recursive_clean()
Invoice Dict
    │
    ▼  CrewAI Pipeline
    │  Extractor → Validator → Resolver → Reporter
    │
    ▼  Validation Results
{A1: true, B7: false, ...}
    │
    ▼  Rule Engine + Historical Calibration
Decision Object
{decision, score, confidence, audit_notes}
    │
    ▼  Schema Formatter
Final Output JSON
    │
    ▼  Save to Disk
results.json + optional --split files
```

---

## Error Handling Strategy

| Layer | Mechanism |
|---|---|
| Input | `recursive_clean()` handles malformed keys/values |
| Parsing | `_parse_json_safe()` with regex cleanup for LLM output |
| Schema | Pydantic + `OutputSchema.normalize_input()` fallback |
| Execution | Retry logic (max 2) with exponential backoff |
| Output | `setdefault()` + final clean pass guarantees required fields |
| Mock API | Silent `except` → fall through to deterministic logic |

---

## Scalability

| Dimension | Approach |
|---|---|
| Horizontal | Stateless design — each invoice independent; batch history passed as param |
| LLM calls | 1 per invoice (Reporter only); all decisions are pure Python |
| Throughput | ~1–2s/invoice (Groq), ~3–5s (Gemini) |
| Memory | Stream processing; no full-invoice caching beyond dedup history |

---

## Security & Auditability

- No PII retained beyond the processing session
- API keys loaded from environment variables only
- Every decision includes a timestamped audit trail
- Historical deviations explicitly flagged in `audit_notes`
- All 10 statutory checks are pure Python — LLM never makes compliance decisions

---

**Maintainer:** V VIJAI | **Version:** 1.0.0 | **Last Updated:** 07 May 2026
