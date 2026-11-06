# Topic 13: Error Handling, Retries & Recovery Strategies

> **Institution:** Tayana Academy | LMS Assessment & Hands-On Guide  
> **Course:** n8n Workflow Automation — Level 2 (Intermediate)  
> **Topic / Assessment:** Topic 13: Error Handling, Retries & Recovery Strategies  
> **Workflow Name:** `Topic 13 - Error Handling, Retries & Recovery Strategies`  
> **Target Audience:** DevOps Engineers, Reliability Specialists & Enterprise Architects  
> **Difficulty Level:** Intermediate  
> **Platform Version:** n8n version 1.0.0 or higher  
> **Student:** Abhishek Nikam (`abhishek.n@tayanasolutions.com`)  
> **Repository:** [topic-13-error-handling-retries-recovery](https://github.com/abhisheknikam3/topic-13-error-handling-retries-recovery)  

---

## 1. Executive Summary & Objective

In enterprise integration workflows, transient network timeouts (HTTP 504), rate limit throttling (HTTP 429), and remote service outages (HTTP 502/503) are unavoidable. Workflows lacking automated error interception and recovery either crash completely—leaving customer orders half-processed—or swallow errors silently, resulting in data loss and blind operational firefighting.

### Architectural Solution
This assessment implements a production-grade **Global Error Interception, Dead Letter Queue (DLQ) & Self-Healing Replay Pipeline** in n8n. The workflow:
1. Simulates an order payment process encountering upstream 504 Gateway Timeouts (`Code_SimulateFailingOrder` ➔ `HTTP_FlakyPaymentGateway` configured with 2 retry attempts).
2. Intercepts the exhausted unhandled crash globally via n8n's **Error Trigger** node (`Error_Trigger`), decoupling business processing from failure recovery.
3. Compiles structured diagnostic telemetry (`Code_ExtractDiagnostics`), extracting the failing node name, HTTP error status code, diagnostic error message, and original unmutated order payload.
4. Categorizes errors into a recovery taxonomy:
   - **Transient Faults (HTTP 429, 502, 503, 504):** Classified as `AUTOMATED_REPLAY`.
   - **Permanent Faults (HTTP 400, 401, 404, Schema Invalidation):** Classified as `MANUAL_INVESTIGATION`.
5. Persists the failed transaction and payload into a stateful **Dead Letter Queue (DLQ)** store (`Code_PersistToDLQ`), stamping each record with a unique `dlqId`.
6. Evaluates recoverability through a multi-path **Switch** node (`Switch_ClassifyRecovery`):
   - **Transient Branch:** Enqueues the transaction for delayed replay against a healthy backup endpoint (`Code_ScheduleReplay`).
   - **Permanent Branch:** Generates a high-priority incident escalation card for on-call SREs (`Code_EscalateIncident`), preventing poison-pill replay loops.
7. Emits a consolidated operational recovery audit receipt (`Code_RecoveryAudit`).

---

## 2. Canvas Architecture & Workflow Diagram

```
[ Primary Execution Flow ]
[ Manual_Trigger ] ➔ [ Code_SimulateFailingOrder ] ➔ [ HTTP_FlakyPaymentGateway ]
                                                              │ (Fails after 2 retries)
                                                              ▼
                                                        [ Unhandled Crash ]

═════════════════════════════════════════════════════════════════════════════════

[ Global Failure Interception & Self-Healing Scope ]
                 [ Error_Trigger ]
                 (Global Crash Listener)
                         │
                         ▼
             [ Code_ExtractDiagnostics ]
             (Parse Telemetry, Status & Payload)
                         │
                         ▼
                [ Code_PersistToDLQ ]
                (Store Payload in DLQ Table)
                         │
                         ▼
             [ Switch_ClassifyRecovery ]
                         │
        ┌────────────────┴────────────────┐
 [ AUTOMATED_REPLAY ]            [ MANUAL_INVESTIGATION ]
 (Transient: 504, 503, 429)      (Permanent: 400, Schema)
        │                                 │
        ▼                                 ▼
[ Code_ScheduleReplay ]          [ Code_EscalateIncident ]
(5s Backoff Replay Dispatcher)   (On-Call SRE Incident Card)
        │                                 │
        └────────────────┬────────────────┘
                         │
                         ▼
               [ Code_RecoveryAudit ]
               (Master Incident Telemetry Card)
```

---

## 3. Error Classification Taxonomy & Recovery Matrix

| HTTP Status Code | Root Cause Category | Fault Classification | Recovery Strategy | Action Executed |
| :---: | :--- | :---: | :---: | :--- |
| **`504`** | Gateway Timeout | **Transient** | `AUTOMATED_REPLAY` | Enqueue in Replay Engine with 5s delay |
| **`503`** | Service Unavailable | **Transient** | `AUTOMATED_REPLAY` | Enqueue in Replay Engine with exponential backoff |
| **`429`** | Rate Limit Throttled | **Transient** | `AUTOMATED_REPLAY` | Dynamic pause and replay |
| **`400`** | Bad Request / Schema Error | **Permanent** | `MANUAL_INVESTIGATION` | Quarantine in DLQ; escalate to `#sre-escalations` |
| **`401 / 403`** | Authentication Failure | **Permanent** | `MANUAL_INVESTIGATION` | Alert SecOps; halt retries immediately |

---

## 4. Node-by-Node Settings Blueprint

| Node Name | Node Type | Critical Parameters & Settings | Architectural Role |
| :--- | :---: | :--- | :--- |
| **`Manual_Trigger`** | `manualTrigger` (v1) | On-demand trigger | Initiates execution of test transaction. |
| **`Code_SimulateFailingOrder`** | `code` (v2) | Emits order `TX-CRASH-801` with amount \$1,450.00 | Simulates business order triggering upstream failure. |
| **`HTTP_FlakyPaymentGateway`** | `httpRequest` (v4.2) | `POST https://httpstat.us/504`, `Retry on Fail: ON` (Max: 2, Wait: 1000ms) | Primary payment gateway simulating gateway timeout. |
| **`Error_Trigger`** | `errorTrigger` (v1) | Global listener | Intercepts unhandled pipeline crashes across the workflow. |
| **`Code_ExtractDiagnostics`** | `code` (v2) | Parses `execution.error`, extracts failed node & original unmutated payload | Evaluates transient vs permanent recovery classification. |
| **`Code_PersistToDLQ`** | `code` (v2) | Generates `dlqId`, appends entry into `staticData.dlqRecords` | Persists unmutated payload for audit and replay. |
| **`Switch_ClassifyRecovery`** | `switch` (v3) | Rules: `AUTOMATED_REPLAY` (Out 0), `MANUAL_INVESTIGATION` (Out 1) | Directs transient faults to replay and permanent faults to SRE. |
| **`Code_ScheduleReplay`** | `code` (v2) | Enqueues replay task with 5s backoff delay | Self-healing engine re-executing transient transactions. |
| **`Code_EscalateIncident`** | `code` (v2) | Formats P1 incident card for `#sre-escalations` | Prevents poison-pill infinite retry loops. |
| **`Code_RecoveryAudit`** | `code` (v2) | Formats final recovery receipt (`HEALTHY_FAILSAFE_ENGAGED`) | Master operational audit log. |

---

## 5. Verification Checkpoints & Testing Guide

### Test 1: Transient 504 Timeout Self-Healing Replay
1. **Primary Execution:**
   - Order `TX-CRASH-801` dispatched to `HTTP_FlakyPaymentGateway`.
   - Node attempts 2 retries and terminates with HTTP 504 Gateway Timeout.
2. **Error Interception & Diagnostics:**
   - `Error_Trigger` intercepts the crash.
   - `Code_ExtractDiagnostics` identifies status 504, classifying `recoveryCategory: "AUTOMATED_REPLAY"` and `isTransientFault: true`.
3. **DLQ Persistence & Routing:**
   - Record stored in DLQ with ID `DLQ-...`.
   - `Switch_ClassifyRecovery` routes to `Code_ScheduleReplay`.
4. **Final Telemetry:**
   - `Code_RecoveryAudit` logs `auditStatus: "INCIDENT_INTERCEPTED_AND_QUARANTINED"` and `recoveryAction: "REPLAY_DISPATCHED"`.

### Test 2: Permanent Schema Error Quarantine
1. **Simulation:**
   - Inject permanent error (HTTP 400 Bad Request or malformed schema).
2. **Execution:**
   - `Code_ExtractDiagnostics` classifies `recoveryCategory: "MANUAL_INVESTIGATION"`.
   - `Switch_ClassifyRecovery` routes to `Code_EscalateIncident`.
   - Incident quarantined in DLQ; zero auto-replay loops triggered.

---

## 6. Production Risks & Mitigation Strategies

| Production Risk | Operational Impact | Architectural Mitigation |
| :--- | :--- | :--- |
| **Poison-Pill Infinite Replay Loops** | Unrecoverable payloads retried indefinitely, exhausting API quotas and compute resources. | Enforce hard cap (`replayCount >= 3` terminates retries and escalates to permanent DLQ). |
| **DLQ Storage Disk Exhaustion** | Unpruned error archives accumulate millions of rows, crashing database storage. | Implement partition pruning policies (delete or archive DLQ records older than 30 days). |
| **Sensitive Data Exposure in DLQ** | Plaintext credentials or payment cards captured in error payloads. | Sanitize payload objects in `Code_ExtractDiagnostics` (mask credit card numbers and passwords) before persisting to DLQ. |

---

## 7. LMS Submission Notes & Assessor Text

```text
Title: Topic 13: Error Handling, Retries & Recovery Strategies
Repository: https://github.com/abhisheknikam3/topic-13-error-handling-retries-recovery
Student: Abhishek Nikam (abhishek.n@tayanasolutions.com)

Assessment Deliverables Summary:
1. Workflow JSON: Topic_13_Error_Handling_Retries_Recovery.json
   - 10 nodes fully configured: Failing Order Simulator, HTTP Gateway with 2x Retry on Fail, Global Error Trigger Listener, Diagnostic Telemetry Parser, Stateful DLQ Persistence Engine, 2-Way Recovery Classifier Switch (Automated Replay vs Manual Investigation), Self-Healing Replay Dispatcher, SRE Incident Escalation Logger, and Master Recovery Telemetry Auditor.
   - Pre-pinned execution test data proving seamless error interception, DLQ persistence, and transient fault self-healing.
2. Comprehensive Documentation: README.md & README.pdf
   - Business problem statement, ASCII & Mermaid architecture diagrams, error classification taxonomy, test verification traces, and production risk mitigations (poison-pill prevention, DLQ pruning, sensitive data masking).
```
