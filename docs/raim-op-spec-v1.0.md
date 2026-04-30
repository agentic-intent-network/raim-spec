# RAIM Operations Specification

## REST AI Interface Modeling — Runtime Operation Execution

> **Document Status:** Working Draft — Public Release
> **Version:** 1.0
> **Date:** April 2026
> **Author:** Chong Feng (Independent Researcher & IETF Contributor)
>
> **Intended Audience:** RAIM system implementers, AI intent execution engine developers, REST API automation engineers
> **Upstream Specification:** RAIM Core Specification v1.0
>
> **Design Provenance:** Derived from the layered intent modeling approach of NAIM-OP (Skill E / Skill F / Operation IR / transaction compensation), adapted for the REST API runtime execution scenario using RAIM Documents as the semantic skeleton.

---

## Quick Entry: What This Specification Solves

RAIM-OP is the **runtime operation execution** extension to the RAIM framework.

Given a complete RAIM Document (produced by RAIM Core modeling), RAIM-OP addresses the following problems:

1. When a user describes an operational intent in natural language ("change eth0's MTU to 9000"), how to structure that intent as a verifiable **RAIM Operation IR**;
2. How to validate the Operation IR's parameters against field constraints declared in the RAIM Document;
3. How to deterministically convert the Operation IR into executable REST requests (using the adaptation information in the RAIM Document);
4. How to handle response parsing, session variable capture, multi-step transaction coordination, and compensation rollback;
5. How to translate execution results back into natural language feedback for the user.

> **Key distinction from NAIM-OP:**
> - NAIM-OP targets NETCONF/RESTCONF protocols and requires a separate protocol mapping layer (Skill F generates NETCONF XML);
> - RAIM-OP targets general REST APIs, where the protocol mapping is already embedded in the RAIM Document's `operations.*.request` definitions. Skill F-RAIM directly executes based on the adaptation information without needing an additional protocol translation layer.

> RAIM-OP depends on the RAIM Document (no YANG skeleton), designed for pure REST API scenarios.

> **Scope note:** Specific transport protocols (MCP, gRPC, HTTP, etc.) are outside the scope of this specification. The RAIM-OP execution model is protocol-agnostic — it consumes adaptation information from the RAIM Document and delegates protocol-specific rendering to the execution engine.

---

## Table of Contents

- [1. Specification Positioning and Terminology](#1-specification-positioning-and-terminology)
  - [1.1 Requirements Language](#11-requirements-language)
  - [1.2 Key Terminology](#12-key-terminology)
  - [1.3 Design Principles](#13-design-principles)
- [2. Three-Layer Architecture and Responsibility Boundaries](#2-three-layer-architecture-and-responsibility-boundaries)
  - [2.1 Specification Layer](#21-specification-layer)
  - [2.2 Skill Layer](#22-skill-layer)
  - [2.3 Tool Layer (Execution Engine)](#23-tool-layer-execution-engine)
  - [2.4 End-to-End Workflow](#24-end-to-end-workflow)
- [3. RAIM Operation IR Specification](#3-raim-operation-ir-specification)
  - [3.1 Basic Structure](#31-basic-structure)
  - [3.2 Core Field Reference](#32-core-field-reference)
  - [3.3 Parameter Validation Rules](#33-parameter-validation-rules)
  - [3.4 Precondition Checking](#34-precondition-checking)
  - [3.5 Transaction and Compensation](#35-transaction-and-compensation)
  - [3.6 Dynamic Values and Session References](#36-dynamic-values-and-session-references)
- [4. Skill E-RAIM: Runtime Intent Modeling](#4-skill-e-raim-runtime-intent-modeling)
  - [4.1 Workflow Steps](#41-workflow-steps)
  - [4.2 Intent Disambiguation](#42-intent-disambiguation)
  - [4.3 Parameter Completion Questioning](#43-parameter-completion-questioning)
- [5. Skill F-RAIM: REST Execution](#5-skill-f-raim-rest-execution)
  - [5.1 Execution Flow](#51-execution-flow)
  - [5.2 Session Variable Management](#52-session-variable-management)
  - [5.3 Asynchronous Operation Handling](#53-asynchronous-operation-handling)
  - [5.4 Response Parsing and Result Dispatch](#54-response-parsing-and-result-dispatch)
  - [5.5 Failure Handling and Compensation Triggering](#55-failure-handling-and-compensation-triggering)
  - [5.6 Execution Report](#56-execution-report)
- [6. Transaction and Compensation Extensions](#6-transaction-and-compensation-extensions)
  - [6.1 Transaction Scope](#61-transaction-scope)
  - [6.2 Compensation Operation Definition](#62-compensation-operation-definition)
  - [6.3 Compensation Execution Rules](#63-compensation-execution-rules)
- [7. Relationship with RAIM Core](#7-relationship-with-raim-core)
- [8. Security and Audit](#8-security-and-audit)
- [9. Evaluation and Boundaries](#9-evaluation-and-boundaries)
- [10. Appendix](#10-appendix)
  - [10.1 Appendix A: RAIM Operation IR JSON Schema](#101-appendix-a-raim-operation-ir-json-schema)
  - [10.2 Appendix B: Typical Examples](#102-appendix-b-typical-examples)

---

## 1. Specification Positioning and Terminology

RAIM-OP is the **runtime operation execution extension** to RAIM Core. Its goal is to transform a user's natural language operational request into a verifiable, executable, and auditable structured intent object — the **RAIM Operation IR** — and then have a deterministic execution engine issue REST requests and parse results based on the adaptation information in the RAIM Document.

### 1.1 Requirements Language

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in BCP 14 (RFC 2119, RFC 8174).

### 1.2 Key Terminology

- **RAIM Document**: The REST API semantic document (Canonical JSON) defined by RAIM Core — the semantic skeleton source for RAIM-OP.
- **RAIM Operation IR**: A runtime intent object, generated by Skill E-RAIM and consumed by Skill F-RAIM.
- **Skill E-RAIM**: The conversational modeling process that converts natural language operational intent into a RAIM Operation IR.
- **Skill F-RAIM**: The deterministic execution process: RAIM Operation IR + RAIM Document → REST request execution + result parsing.
- **Session Variable**: Temporary key-value pairs maintained by the executor within a single session. Written by `response.capture[].name` and consumed by subsequent requests via `$session.{name}` references.
- **Transaction**: A set of Operation IRs sharing the same `transaction-id`, treated as a single execution unit.
- **Compensation Operation**: A reverse operation used to roll back successfully executed operations, defined in the RAIM Document's `undo` field or the Operation IR's `compensation` field.
- **Hard Constraint**: Strong constraints from the RAIM Document `fields[]` (type / value range). Violation results in execution refusal.
- **Soft Constraint**: Optional, user-supplied condition-type constraints (e.g., "MTU must be greater than 1500") used for intent reasonability checking. Violation prompts a confirmation question rather than immediate refusal.
- **Precondition**: Conditions from the RAIM Document `operations.*.preconditions` that MUST be checked before execution. Declared as natural language strings.

### 1.3 Design Principles

1. **Protocol transparency**: The Operation IR is fully defined before any protocol-specific request is constructed. Skill F-RAIM is responsible for converting the IR into actual requests based on RAIM Document adaptation information; the AI does not directly participate in request assembly.
2. **Semantics bound to the RAIM Document**: Parameter validation must be based not only on user input but also on the RAIM Document's `fields[]` constraints and `operations.*.preconditions`.
3. **User transparency**: Users always interact through natural language and do not directly edit Operation IRs or REST request details.
4. **Explicit rejection of unsupported extensions**: If the execution engine does not support a particular capability (e.g., async polling, transaction compensation), it MUST explicitly refuse and explain to the user, rather than silently degrading.
5. **Auditable execution**: All requests, responses, captured values, and compensation operations MUST be recorded in the Execution Report, supporting complete traceability.
6. **Lightweight cooperation**: RAIM-OP does not define its own REST adaptation rules. All adaptation information (URL templates, body templates, parser, capture) is read from the RAIM Document.

---

## 2. Three-Layer Architecture and Responsibility Boundaries

### 2.1 Specification Layer

Defines:

- RAIM Operation IR structure and constraints;
- Parameter validation rules (based on RAIM Document fields);
- Precondition checking mechanism;
- Transaction and compensation model;
- Execution Report format requirements;
- Soft constraint expression syntax.

### 2.2 Skill Layer

- **Skill E-RAIM**: Natural language intent → identify target resource/operation → parameter population and validation → output RAIM Operation IR
- **Skill F-RAIM**: RAIM Operation IR → retrieve adaptation information from RAIM Document → assemble request → execute → parse result → natural language feedback

### 2.3 Tool Layer (Execution Engine)

The execution engine MUST support:

- Loading RAIM Documents (including execution configuration after Convention Profile merging);
- Locating adaptation information by resource name + operation name;
- Rendering URL templates and body templates (injecting Operation IR `params`);
- Injecting authentication headers from Convention Profile (if a resource/operation has `security-override`, use the override value with priority);
- Executing HTTP requests (including timeout control);
- Success/failure determination (HTTP code + body business code);
- Session Variable management (capture writes + subsequent request reads);
- Asynchronous operation polling (if declared in the RAIM Document);
- Response parsing (restoring field values per the parser strategy);
- Compensation execution (triggering undo for successfully completed operations on failure);
- Execution Report generation.

### 2.4 End-to-End Workflow

```
User natural language intent
       │
       ▼
  Skill E-RAIM
  ┌─────────────────────────────────┐
  │  1. Load RAIM Document          │
  │  2. Identify target Resource + Operation │
  │  3. Parameter population and questioning │
  │  4. Hard Constraint validation   │
  │  5. Soft Constraint confirmation │
  │  6. Precondition check (optional)│
  │  7. Output Operation IR         │
  └─────────────────────────────────┘
       │
       ▼
  [User confirmation (optional)]
       │
       ▼
  Skill F-RAIM (Execution Engine)
  ┌─────────────────────────────────┐
  │  1. Retrieve adaptation info    │
  │     from RAIM Document          │
  │  2. Render URL / body templates │
  │  3. Inject auth headers         │
  │  4. Execute HTTP request        │
  │  5. Success/failure determination│
  │  6. Async polling (if needed)   │
  │  7. capture → Session Variable  │
  │  8. Response parsing → business fields │
  │  9. Failure compensation (if triggered) │
  │ 10. Generate Execution Report   │
  └─────────────────────────────────┘
       │
       ▼
  Natural language result feedback to user
```

---

## 3. RAIM Operation IR Specification

### 3.1 Basic Structure

A RAIM Operation IR is a JSON object describing a single runtime operation intent:

```json
{
  "operation-id":   "uuid",
  "transaction-id": "uuid",
  "resource":       "interface",
  "operation":      "update",
  "params": {
    "ns":   "default",
    "name": "eth0",
    "mtu":  9000
  },
  "soft-constraints": {},
  "precondition-check": true,
  "compensation": [],
  "context": {
    "raim-document-ref": "file://models/device-api.raim.json",
    "environment":       "production",
    "timeout":           30,
    "dry-run":           false
  },
  "description": "Update interface eth0 MTU to 9000",
  "order": 1
}
```

#### 3.1.1 Support Levels

Fields fall into two categories:

- **core**: The execution engine MUST support;
- **optional extension**: Optional support; if unsupported, the engine MUST explicitly reject the entire Operation IR and explain the reason to the user.

### 3.2 Core Field Reference

#### 3.2.1 `resource` (core, MUST)

The target resource name. MUST match an object in the `resources[]` array of the RAIM Document pointed to by `raim-document-ref`.

#### 3.2.2 `operation` (core, MUST)

The target operation name: `create` / `get` / `list` / `update` / `patch` / `delete`, or the `name` of a `custom` operation. The execution engine MUST find the corresponding operation definition in the RAIM Document, otherwise it MUST refuse execution.

#### 3.2.3 `params` (core, MUST)

Key-value pairs of operation parameters. Sources include:

- path parameters (e.g., `ns`, `name`);
- body fields (e.g., `mtu`, `vlan-id`);
- query parameters (e.g., `page`, `limit`).

The execution engine MUST match keys in `params` against `fields[].name` in the RAIM Document and perform Hard Constraint validation (see 3.3).

#### 3.2.4 `context` (core, MUST)

Execution context:

| Field | Required | Description |
|---|---|---|
| `raim-document-ref` | MUST | URI or file path pointing to the RAIM Document |
| `environment` | SHOULD | Which environment from `api.environments` to use |
| `timeout` | MAY | Override the global timeout setting (seconds) |
| `dry-run` | MAY | When `true`, validate only — do not send requests. Output describes what requests would be sent. |

#### 3.2.5 `transaction-id` (optional extension)

When present, indicates this operation belongs to a transaction. Operation IRs sharing the same `transaction-id` are treated as a single execution unit and compensated as a whole on failure.

#### 3.2.6 `order` (optional extension)

Execution order within a transaction (starting from 1). The execution engine SHOULD execute in ascending `order`.

#### 3.2.7 `soft-constraints` (optional extension)

Optional runtime condition constraints (not from model hard constraints). Format: field name → condition expression.

Soft constraints represent user preferences or business rules beyond what the RAIM Document declares. When a soft constraint is violated, the system SHOULD ask the user for confirmation rather than directly refusing execution.

#### 3.2.8 `precondition-check` (optional extension)

When `true`, the execution engine SHOULD initiate a query before the main operation (corresponding to a get/list operation in the RAIM Document) to verify that preconditions are satisfied. The check behavior depends on the type of `preconditions` field in the RAIM Document (see 3.4).

#### 3.2.9 `compensation` (optional extension)

Explicitly declares the compensation definition for this operation on failure, in the format `operation-ir[]`. If not defined, the execution engine automatically triggers compensation based on the operation's `undo` definition in the RAIM Document (if it exists).

### 3.3 Parameter Validation Rules

Skill E-RAIM MUST perform the following validations on `params` before generating an Operation IR:

1. **Required field check**: Fields declared as `required: true` in the RAIM Document `fields[]` that are absent from `params` MUST trigger a question to the user to supplement;
2. **Type validation**: The value type of each field in `params` must match `fields[].type`; type mismatches MUST be reported as errors;
3. **Constraint validation**: Value ranges in `fields[].constraints` must be satisfied; violations MUST be pointed out to the user with the constraint that was violated, and execution refused;
4. **Read-only field check**: Fields declared as `read-only: true` MUST NOT appear in the `params` of write operations;
5. **Soft Constraint check**: If `soft-constraints` are present, they are checked after Hard Constraints pass; on violation, the user is asked for confirmation rather than execution being directly refused.

Parameter validation failures MUST produce a Structured Error Report:

```json
{
  "error_code": "ParameterValidationError",
  "path": "/params/mtu",
  "message": "mtu value 70000 exceeds maximum allowed value 65535",
  "expected": "uint32 in range 68..65535",
  "severity": "error",
  "hint": "Please provide a value between 68 and 65535"
}
```

### 3.4 Precondition Checking

When the Operation IR has `precondition-check: true`, the execution engine determines behavior based on the form of the target operation's `preconditions` field in the RAIM Document (see RAIM Core v1.0, §4.5.2):

#### 3.4.1 Structured Preconditions (Automated Check)

If `preconditions` is a structured object with a `check` field:

1. Render the `check.params` using the current Operation IR's `params` (resolving `$params.{field}` references);
2. Execute the `check` query: locate the target `resource` + `operation` in the RAIM Document and execute the corresponding request;
3. Evaluate `check.assert` against the check response body;
4. If `assert` evaluates to `true` — precondition satisfied, proceed with execution;
5. If `assert` evaluates to `false` — precondition NOT satisfied, MUST abort and report `PreconditionFailed` error;
6. If the execution engine does not support automated precondition checking, MUST fall back to manual confirmation using `preconditions.description` (see §3.4.2).

#### 3.4.2 Natural Language Preconditions (Manual Confirmation)

If `preconditions` is a plain string:

1. The execution engine MUST display the precondition content to the user and request manual confirmation:

> "The following precondition must be satisfied before executing this operation: {preconditions}. Please confirm this condition is met (yes/no)?"

2. After user confirmation, continue execution; on denial or timeout, MUST abort execution.

### 3.5 Transaction and Compensation

#### 3.5.1 Transaction Scope

Operation IRs sharing a `transaction-id` form a transaction. The execution engine SHOULD:

- Execute operations in `order` sequence;
- Record each step's execution state (success / failure);
- When any step fails (and `on-fail` logic triggers compensation), trigger compensation for the successfully completed operations;
- Output a unified Execution Report at transaction completion.

#### 3.5.2 Compensation Definition Priority

Compensation operation definition sources (priority from high to low):

1. `operation-ir.compensation` (explicitly specified);
2. The target operation's corresponding `undo` definition in the RAIM Document (see 6.2);
3. No compensation defined: record "no compensation defined," do not execute compensation, but MUST warn in the Execution Report.

#### 3.5.3 Transaction and External Orchestration

If RAIM-OP runs within an external orchestration framework (such as a workflow engine), the external framework is responsible for cross-system transaction management. RAIM-OP internal compensation MUST only cover operations already executed through RAIM-OP and MUST NOT duplicate compensation across systems.

### 3.6 Dynamic Values and Session References

#### 3.6.1 Session Variable References

Values in `params` may reference Session Variables (captured from previous steps):

```json
{
  "params": {
    "ns":   "default",
    "name": "$session.created-interface-name"
  }
}
```

The `$session.` prefix identifies a Session Variable reference. The execution engine MUST retrieve the value from the session before rendering. If the variable does not exist, the engine MUST report an error and abort execution.

#### 3.6.2 Previous Step Result References

Within a transaction, subsequent operations may reference the parsed result of a previous step via `$prev.{operation-id}.{field}`:

```json
{
  "params": {
    "uuid": "$prev.op-001.uuid"
  }
}
```

**Resolution priority**: When the same parameter name exists in both a `$prev.*` reference and a Session Variable (`$session.*`), the resolution order is:

1. Explicit literal values in `params` (highest priority)
2. `$prev.{operation-id}.{field}` references
3. `$session.{name}` references (lowest priority)

If a value is only available through one mechanism, that mechanism is used regardless of priority. The execution engine MUST report an error and abort if a reference cannot be resolved through any mechanism.

---

## 4. Skill E-RAIM: Runtime Intent Modeling

### 4.1 Workflow Steps

Skill E-RAIM's goal is to convert a user's natural language intent into a complete and executable RAIM Operation IR.

**Phase 1: RAIM Document Loading**

1. Confirm which RAIM Document to use (if multiple exist in the system, let the user choose or determine from context);
2. Load the RAIM Document and build a resource index (resource name → operations map).

**Phase 2: Initial Intent Identification**

3. Extract the action verb and target object from the user's input and map to a resource + operation in the RAIM Document;
4. If ambiguity exists (e.g., multiple resources match), enter the intent disambiguation flow (see 4.2);
5. Confirm the identification result with the user: "I understand you want to perform `{operation}` on resource `{resource}`. Is that correct?" (OPTIONAL, configurable as silent confirmation).

**Phase 3: Parameter Population**

6. Extract existing parameters from the user's input (e.g., interface name, MTU value);
7. Compare against the RAIM Document `fields[]` to identify missing required parameters, and ask for them one at a time (see 4.3);
8. Perform Hard Constraint validation on all parameters.

**Phase 4: Constraint and Condition Confirmation**

9. Check Soft Constraints (if present); if violated, prompt the user and request confirmation;
10. If `preconditions` exist, display them to the user and request confirmation;
11. Display the operation's `side-effects` so the user is aware of the impact (especially for non-idempotent operations);
12. For non-idempotent (`idempotency: false`) operations with side effects, SHOULD request secondary confirmation from the user.

**Phase 5: Output Operation IR**

13. Generate the RAIM Operation IR;
14. Proceed to Skill F-RAIM execution.

### 4.2 Intent Disambiguation

When the following situations occur, Skill E-RAIM MUST enter the disambiguation flow rather than silently guessing:

- Multiple resources match the intent description (e.g., both `interface` and `interface-actions` could match "restart interface");
- The operation is unclear (e.g., "update interface" could be `update` or `patch`);
- Parameters are ambiguous (e.g., "interface name" could refer to `name` or `uuid`).

Disambiguation method: ask the user a single-choice question; do not list all possibilities at once.

Example:

> "Regarding the 'restart interface' operation you mentioned, does it refer to:
> A. Updating interface configuration (update), or
> B. Executing a restart action (interface-actions/restart)?"

### 4.3 Parameter Completion Questioning

Questioning rules:

- Ask about only one missing parameter at a time; do not list all missing items at once;
- When questioning, SHOULD provide the field's type and constraint hints (from `fields[].description` and `fields[].constraints`);
- If the field has `examples`, SHOULD provide example values when questioning;
- If the user provides a value exceeding the constraint range, MUST point out the violated constraint and request re-entry.

Example question:

> "Please provide the MTU value (valid range: 68~65535, common values: 1500 or 9000):"

---

## 5. Skill F-RAIM: REST Execution

Skill F-RAIM is the deterministic execution phase. It does not participate in intent understanding and is only responsible for executing requests and returning results based on the RAIM Operation IR + RAIM Document.

### 5.1 Execution Flow

The execution engine processes each Operation IR in the following order:

1. **Load RAIM Document adaptation information**: Find the corresponding `operations.*.request` and `operations.*.response` by `resource` + `operation`;
2. **Merge Convention Profile**: Apply default headers / auth / timeout settings from the Convention Profile;
3. **Render endpoint template**: Substitute path parameters from `params` into `request.url-template`, applying protocol-appropriate encoding (e.g., URL encoding for HTTP, path segment encoding for gRPC);
4. **Render body template**: Substitute body fields from `params` into `request.body.template`, applying protocol-appropriate escaping for each value (e.g., JSON escaping for HTTP JSON bodies, protobuf serialization for gRPC);
5. **Inject authentication credentials**: Inject auth credentials per the Convention Profile's `auth` configuration; if the target resource/operation has `security-override`, use the `security-override` instead of the Convention Profile's `auth` configuration (see 7.2);
6. **Execute protocol request**: Send the request via the transport protocol determined by the RAIM Document's adaptation information, applying `timeout` control;
7. **Success/failure determination**:
   a. Transport-level response code matches `response.success-codes` (e.g., HTTP 200-299, gRPC OK);
   b. `response.success-condition` expression evaluates to true on the response body;
8. **Async handling** (if applicable, see 5.3);
9. **capture execution**: Extract fields per `response.capture[]` and store in Session Variables;
10. **Response parsing**: Restore business fields per the `response.parser` strategy;
11. **Failure handling** (see 5.5).

### 5.2 Session Variable Management

Session Variables are temporary key-value pairs maintained by the executor within a single transaction:

- **Write**: The sole write source is the `response.capture[]` operation. The Session Variable key name is the value of `capture[].name`.
- **Read**:
  - `$session.{name}` references in Operation IR `params`;
  - `{$session.{name}}` references in subsequent url-templates;
  - `{{.$session.{name}}}` references in subsequent body templates;
- **Lifecycle**: From transaction start to Execution Report output completion; MUST be cleared afterward;
- **Cross-step access**: All subsequent steps within the same `transaction-id` share Session Variables;
- **Naming source**: The Session Variable key name source is unique — it can only be the value of `capture[].name`. Any `$session.*` reference must correspond to a declared `capture[].name`.

If a `capture.required: true` capture fails (target path does not exist), the execution engine MUST stop all subsequent steps that depend on that variable and report a `RequiredCaptureFailure` error.

### 5.3 Asynchronous Operation Handling

When the target operation in the RAIM Document declares `async.enabled: true` (see RAIM Core v1.0, §4.5.8), the execution engine enters an asynchronous handling flow:

1. Execute the initial request as normal (steps 1–6 of §5.1);
2. Extract captured variables from the initial response (e.g., `task-id`);
3. Render the `async.poll-url` using captured variables (e.g., `{task-id}` resolves to the captured `task-id` Session Variable value);
4. Periodically send polling requests to the rendered poll URL at the interval specified by `async.poll-interval`;
5. After each polling response, evaluate:
   - `async.completion-condition` — if `true`, the operation is complete; proceed to step 7;
   - `async.failure-condition` — if `true`, the operation is failed; proceed to step 8;
6. If neither condition is met, continue polling. If the elapsed time exceeds `async.max-wait`, MUST stop polling, report an `AsyncTimeoutError`, and trigger compensation per §5.5;
7. **Completion**: Capture response fields per the final polling response, parse business fields per `response.parser`, and proceed to subsequent steps or report completion;
8. **Failure**: Treat as operation execution failure; trigger compensation per §5.5.

The RAIM Document's `async` object provides all declarations (`poll-url`, `poll-interval`, `max-wait`, `completion-condition`, `failure-condition`) that define the polling behavior — the execution engine follows these declarations deterministically.

### 5.4 Response Parsing and Result Dispatch

Per the RAIM Document's `response.parser` strategy:

- `auto`: Automatically restore based on symmetric field-mapping declarations;
- `field-mapping`: Extract field values per `field-mapping[].{api-field → model-field}` mappings;
- `template`: Extract via reverse template;
- `script`: Execute a script in a sandbox and return field key-value pairs.

Parsing results SHOULD be presented in a user-readable format (Skill F-RAIM formats field values into a natural language summary).

On parse failure:

- This is not equivalent to request failure (the request may have succeeded);
- MUST record the parse failure reason in the Execution Report;
- The returned field set may be partial or empty, but MUST NOT report "request failed."

### 5.5 Failure Handling and Compensation Triggering

When an Operation IR execution fails (HTTP failure or business failure):

1. Record the failure reason (including the business error message extracted via `error-message-path`);
2. Determine subsequent behavior based on the `on-fail` strategy (if declared in the Operation IR):

| `on-fail` Value | Behavior |
|---|---|
| `"stop"` (default) | Stop all subsequent steps. Trigger each successfully completed step's `undo` (if defined). Output Execution Report after the entire transaction completes. |
| `"continue"` | Record the current step failure, do **not** trigger any compensation, continue executing subsequent steps. Summarize all failed steps in the Execution Report after the execution chain completes. |
| `"compensate-and-continue"` | Trigger the **current step's** `undo` (if defined) for compensation. After compensation completes (whether successful or not), continue executing subsequent steps. Record the current step failure and compensation result in the Execution Report. |

> **Note**: `"continue"` and `"compensate-and-continue"` have different semantics:
> - `"continue"` means "ignore the failure of this operation, preserve possible partial-completion state, continue forward";
> - `"compensate-and-continue"` means "first roll back the side effects of the current operation, then continue forward."
>
> If you need to both roll back the current step and continue subsequent steps after a failure, you MUST use `"compensate-and-continue"` — do not misuse `"continue"`.

3. Trigger compensation operations (see Section 6);
4. Output the Execution Report;
5. Feed the result back to the user in natural language:
   > "Interface eth0 MTU update failed: {error-message}. Rollback to original state has been attempted."

#### 5.5.1 Dry-run Mode

When `context.dry-run: true`:

- The execution engine MUST NOT send any actual HTTP requests;
- The engine MUST output a preview describing what requests would be sent, including method, URL, headers (with credentials redacted), and body;
- Any precondition checks declared in the Operation IR are also described in the preview without sending requests.

```json
{
  "dry-run": true,
  "would-execute": [
    {
      "step": 1,
      "method": "DELETE",
      "url":    "https://192.168.1.1/api/v1/namespaces/default/interfaces/eth0",
      "headers": { "Authorization": "Bearer [REDACTED]" },
      "body":   null
    }
  ],
  "warnings": [
    "Interface eth0 deletion will release all associated resources (side-effect from RAIM Document)"
  ]
}
```

### 5.6 Execution Report

After each execution (including transactions), an Execution Report MUST be generated containing:

| Field | Description |
|---|---|
| `transaction-id` | Transaction ID (if present) |
| `operations` | Execution result for each Operation IR (success / failed / skipped) |
| `session-variables` | List of captured Session Variable names (values redacted) |
| `compensations` | Compensation operation execution results (success / failed / not defined) |
| `raim-document-version` | RAIM Document version used |
| `convention-profile` | Convention Profile name used |
| `degradations` | Parser degradation records (if any) |
| `warnings` | Warning list (Session Variable overwrites, precondition check failures, etc.) |
| `timestamp` | Execution timestamp |
| `residual-states` | Description of residual states requiring manual intervention (MUST be populated when compensation fails, see 6.3) |

---

## 6. Transaction and Compensation Extensions

### 6.1 Transaction Scope

RAIM-OP transactions are **logical transactions within a single execution session**, not cross-system distributed transactions:

- All Operation IRs sharing a `transaction-id` execute sequentially within the same execution engine instance;
- Transactions do not guarantee atomicity (each REST request is an independent HTTP call), but do guarantee **compensation reachability** (when each step has a corresponding compensation definition, compensation will necessarily be attempted on failure).

### 6.2 Compensation Operation Definition

Compensation operations can come from two sources:

**Source 1: `operation.undo` in the RAIM Document** (formally specified in RAIM Core v1.0)

The RAIM Document's operation definition supports an `undo` field describing the reverse request for that operation (see RAIM Core v1.0, Section 4.5.3):

```json
{
  "create": {
    "request": { "method": "POST", "url-template": "/api/v1/namespaces/{ns}/interfaces" },
    "response": {
      "success-codes": "200-299",
      "parser": "auto",
      "capture": [{ "name": "created-name", "data-path": "$.data.name", "required": true }]
    },
    "undo": {
      "description": "Delete the created interface if subsequent steps fail",
      "operation": "delete",
      "params-from-session": { "name": "$session.created-name" }
    },
    "idempotency": false
  }
}
```

When triggering compensation, the execution engine MUST read the `undo` definition from the RAIM Document and combine `undo.operation` + `undo.params-from-session` + `undo.params-static` to compose the compensation operation's parameters.

**Source 2: Explicit `compensation` declared in the Operation IR**

```json
{
  "compensation": [
    {
      "resource":  "interface",
      "operation": "delete",
      "params":    { "ns": "default", "name": "$session.created-interface-name" },
      "description": "Delete the interface if subsequent update fails"
    }
  ]
}
```

### 6.3 Compensation Execution Rules

1. When a transaction fails, compensation is triggered for operations that have already completed successfully;
2. Each compensation operation's `params` SHOULD preferentially use Session Variables, ensuring compensation targets the actually executed resource rather than an assumed value;
3. If a compensation operation itself fails, MUST record the compensation failure and continue to the next compensation — a single compensation failure MUST NOT interrupt the entire compensation sequence;
4. After compensation completes, summarize in the Execution Report which compensations succeeded and which failed;
5. If any compensation operation fails, the Execution Report MUST include a description of residual states requiring manual intervention, identifying the affected resources and their last known state.

> **Nested compensation is not supported**: When a compensation operation itself fails, RAIM-OP MUST NOT trigger compensation for the compensation (compensation of compensation is not allowed). The only handling at this point is to output `residual-states` information for manual intervention.

---

## 7. Relationship with RAIM Core

### 7.1 Responsibility Division

| Specification | Responsibility | Positioning |
|---|---|---|
| **RAIM Core** | REST API semantic modeling (design-time / reverse engineering); outputs RAIM Document | Modeling framework |
| **RAIM-OP** | Runtime intent understanding and execution; consumes RAIM Document | Execution framework |

### 7.2 RAIM-OP's Dependencies on the RAIM Document

RAIM-OP reads from the RAIM Document:

| RAIM Document Field | RAIM-OP Usage |
|---|---|
| `resources[].fields[]` | Hard Constraint validation, parameter questioning hints |
| `resources[].operations.*.request` | URL template, body template |
| `resources[].operations.*.response` | Success determination, data-path, parser, capture |
| `resources[].operations.*.preconditions` | Precondition checking; Form A (string) → manual confirmation, Form B (structured) → automated check if engine-supported (see §3.4) |
| `resources[].operations.*.undo` | Auto-discovery of compensation operation definitions |
| `resources[].operations.*.side-effects` | User notification before execution |
| `resources[].operations.*.idempotency` | Whether secondary confirmation is needed |
| `resources[].operations.*.error-conditions` | Failure message mapping reference |
| `resources[].operations.*.async` | Asynchronous operation polling configuration (poll-url, poll-interval, max-wait, completion/failure conditions; see §5.3) |
| `resources[].security-override` | Resource-level auth override; when present, RAIM-OP execution engine MUST preferentially use `security-override` instead of Convention Profile's `auth` configuration for injecting auth headers |
| `convention-profile` | Auth headers (default, used when resource/operation has no `security-override`), default success-codes, parser degradation strategy, token-refresh configuration |

> **Auth header injection priority (high to low)**:
> 1. `resources[].operations.*.convention-overrides.auth` (operation-level override)
> 2. `resources[].convention-overrides.auth` or `resources[].security-override` (resource-level override)
> 3. `convention-profile.conventions.auth` (global default)

### 7.3 The `undo` Field in RAIM Document

RAIM Core v1.0 has formally specified the `operations.*.undo` field structure (see RAIM Core v1.0, Section 4.5.3). The execution engine SHOULD preferentially auto-discover compensation operations from the RAIM Document's `undo` definitions rather than relying on explicit `compensation` declarations in the Operation IR.

---

## 8. Security and Audit

### 8.1 Operation Trustworthiness

- Externally or AI-generated Operation IRs MUST undergo parameter validation (Hard Constraints) before execution;
- Operation IRs from untrusted sources should enter a manual approval process;
- Production environments MUST NOT allow high-risk operations (delete / non-idempotent create) to execute automatically without human confirmation.

### 8.2 Parameter Injection Prevention

- All variables substituted into URLs MUST be URL-encoded;
- All string variables substituted into bodies MUST be JSON-escaped;
- Session Variable values MUST also undergo type validation and escaping before substitution.

### 8.3 Credential Security

- Convention Profile or RAIM Document `token-source` MUST NOT hardcode credentials;
- Execution Reports and logs MUST redact authentication headers (Authorization / Cookie);
- If Session Variables capture credential-type fields (e.g., tokens), they MUST be marked as sensitive and redacted in logs.

### 8.4 Script Sandboxing

Response parsing scripts for `response.parser.type = "script"` MUST execute in a sandbox:

- No filesystem read/write (outside the working directory);
- No network access;
- Execution time limit (recommended 30 seconds);
- Memory quota (recommended 256MB).

### 8.5 Authorization and Audit

- Operation IR execution permission is distinct from RAIM Document access permission;
- High-risk operations (delete / batch create) SHOULD require secondary confirmation;
- All execution, compensation, and Session Variable captures MUST be recorded in the Execution Report and persisted, supporting complete audit traceability;
- `residual-states` produced by compensation failures MUST be persisted until manual confirmation of resolution.

---

## 9. Evaluation and Boundaries

### 9.1 Boundaries

RAIM-OP does not guarantee:

- **Fully automatic precondition checking**: Structured preconditions (RAIM Core §4.5.2 Form B) enable automated checking, but engine support is RECOMMENDED not REQUIRED. Natural language preconditions (Form A) can only be displayed to the user for manual confirmation;
- **Cross-system transaction consistency**: RAIM-OP's transaction compensation only covers operations executed through RAIM-OP and does not coordinate external systems;
- **Complex workflows**: Scenarios requiring conditional branching, loops, or other control flow are beyond RAIM-OP's scope and should be handled by external workflow engines;
- **All RAIM Documents are directly executable**: If a RAIM Document contains `parser: "script"` and the sandbox does not support it, the execution engine MUST refuse and explain;
- **Nested compensation**: When a compensation operation itself fails, RAIM-OP does not trigger secondary compensation and only outputs residual state information for manual handling.

### 9.2 Evaluation Dimensions

| Evaluation Item | Method |
|---|---|
| Parameter coverage | Whether Operation IR `params` fields cover all `fields[].required=true` required fields in the RAIM Document |
| Compensation coverage | Whether write operations all have corresponding compensation definitions (`undo` or `compensation`) |
| Precondition checkability | Proportion of preconditions in structured Form B (auto-checkable) vs Form A natural language (manual confirmation required) |
| Execution determinism | Whether all parsers are independent of scripts (deterministically executable) |
| Session Variable completeness | Whether all `$session.*` references have corresponding `capture[].name` declarations |

---

## 10. Appendix

### 10.1 Appendix A: RAIM Operation IR JSON Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "RAIM Operation IR",
  "type": "object",
  "required": ["resource", "operation", "params", "context"],
  "additionalProperties": false,
  "properties": {
    "operation-id":   { "type": "string" },
    "transaction-id": { "type": "string" },
    "resource":   { "type": "string" },
    "operation":  { "type": "string" },
    "params":     { "type": "object" },
    "soft-constraints": { "type": "object" },
    "precondition-check": { "type": "boolean", "default": false },
    "compensation": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["resource", "operation", "params"],
        "properties": {
          "resource":  { "type": "string" },
          "operation": { "type": "string" },
          "params":    { "type": "object" },
          "description": { "type": "string" }
        }
      }
    },
    "context": {
      "type": "object",
      "required": ["raim-document-ref"],
      "additionalProperties": false,
      "properties": {
        "raim-document-ref": { "type": "string" },
        "environment":       { "type": "string" },
        "timeout":           { "type": "number", "minimum": 1 },
        "dry-run":           { "type": "boolean", "default": false }
      }
    },
    "on-fail": {
      "type": "string",
      "enum": ["stop", "continue", "compensate-and-continue"],
      "default": "stop"
    },
    "description": { "type": "string" },
    "order":       { "type": "integer", "minimum": 1 }
  }
}
```

---

### 10.2 Appendix B: Typical Examples

#### B.1 Single-Step Operation: Update Interface MTU

```json
{
  "operation-id":   "op-001",
  "resource":       "interface",
  "operation":      "update",
  "params": {
    "ns":     "default",
    "name":   "eth0",
    "mtu":    9000,
    "vlan-id": 100
  },
  "precondition-check": true,
  "context": {
    "raim-document-ref": "file://models/device-api.raim.json",
    "environment":       "production",
    "timeout":           30
  },
  "on-fail": "stop",
  "description": "Update interface eth0 MTU to 9000"
}
```

#### B.2 Transaction: Create Then Update, with Rollback on Failure

> **Note**: In this example, `op-002`'s `params.name` references `$session.created-interface-name`. This Session Variable's write source is the capture rule defined in the RAIM Document's `interface.create.response.capture`:
> ```json
> "capture": [{ "name": "created-interface-name", "data-path": "$.data.name", "required": true }]
> ```
> After `op-001` succeeds, the execution engine writes the value at `$.data.name` from the response into the Session Variable `created-interface-name` for `op-002` to reference.

```json
[
  {
    "operation-id":   "op-001",
    "transaction-id": "txn-abc123",
    "resource":       "interface",
    "operation":      "create",
    "params": {
      "ns":   "default",
      "name": "eth1",
      "mtu":  1500
    },
    "context": {
      "raim-document-ref": "file://models/device-api.raim.json",
      "environment": "production"
    },
    "compensation": [
      {
        "resource":  "interface",
        "operation": "delete",
        "params":    { "ns": "default", "name": "$session.created-interface-name" },
        "description": "Delete the interface if subsequent update fails"
      }
    ],
    "on-fail": "stop",
    "order": 1
  },
  {
    "operation-id":   "op-002",
    "transaction-id": "txn-abc123",
    "resource":       "interface",
    "operation":      "update",
    "params": {
      "ns":   "default",
      "name": "$session.created-interface-name",
      "mtu":  9000
    },
    "context": {
      "raim-document-ref": "file://models/device-api.raim.json",
      "environment": "production"
    },
    "on-fail": "stop",
    "order": 2
  }
]
```

#### B.3 Dry-run: Preview Request Without Actual Execution

```json
{
  "operation-id": "op-dry-001",
  "resource":     "interface",
  "operation":    "delete",
  "params": {
    "ns":   "default",
    "name": "eth0"
  },
  "context": {
    "raim-document-ref": "file://models/device-api.raim.json",
    "environment":       "production",
    "dry-run":           true
  },
  "description": "Preview: What requests would be sent to delete eth0?"
}
```

In dry-run mode, the execution engine MUST output a preview of the request without sending any actual request:

```json
{
  "dry-run": true,
  "would-execute": [
    {
      "step": 1,
      "method": "DELETE",
      "url":    "https://192.168.1.1/api/v1/namespaces/default/interfaces/eth0",
      "headers": { "Authorization": "Bearer [REDACTED]" },
      "body":   null
    }
  ],
  "warnings": [
    "Interface eth0 deletion will release all associated resources (side-effect from RAIM Document)"
  ]
}
```

#### B.4 Soft Constraint Questioning Scenario

User input: "Change eth0's MTU to a very large value"

Skill E-RAIM processing:

1. Identify intent: `interface` / `update` / `params.name=eth0`;
2. MTU value not provided → question: "Please provide the MTU value (valid range: 68~65535, common values: 1500 or 9000):";
3. User answers: "100000" → Hard Constraint validation fails (exceeds 65535) → report error and re-question;
4. User answers: "9000" → passes validation, continue.

#### B.5 `on-fail: "compensate-and-continue"` Scenario

Used for scenarios where "partial failure is tolerable but the current step's side effects must be rolled back." For example, in batch configuration where one interface fails but others should continue:

```json
[
  {
    "operation-id":   "op-001",
    "transaction-id": "txn-batch-001",
    "resource":       "interface",
    "operation":      "update",
    "params": { "ns": "default", "name": "eth0", "mtu": 9000 },
    "context": { "raim-document-ref": "file://models/device-api.raim.json", "environment": "production" },
    "on-fail": "compensate-and-continue",
    "order": 1
  },
  {
    "operation-id":   "op-002",
    "transaction-id": "txn-batch-001",
    "resource":       "interface",
    "operation":      "update",
    "params": { "ns": "default", "name": "eth1", "mtu": 9000 },
    "context": { "raim-document-ref": "file://models/device-api.raim.json", "environment": "production" },
    "on-fail": "compensate-and-continue",
    "order": 2
  }
]
```

Execution engine behavior: If `op-001` fails, trigger `op-001`'s `undo`, then continue executing `op-002` after completion. Record `op-001` failure and its compensation result in the Execution Report.

---

## References

- **RAIM Core Specification** (companion document) — Architecture, RAIM Document format, Convention Profile, Fields, Operations, Field Mapping, and Compensation definitions
- OpenAPI Specification: https://spec.openapis.org/oas/latest.html
- JSON Schema: https://json-schema.org/
