# RAIM Core Specification

## REST AI Interface Modeling — Core Semantic Layer

> **Document Status:** Working Draft — Public Release
> **Version:** 1.0
> **Date:** April 2026
> **Author:** Chong Feng (Independent Researcher & IETF Contributor)

---

## Quick Entry: What This Specification Solves

RAIM (REST AI Interface Modeling) is an intermediate representation framework for **AI-assisted REST API semantic modeling and adaptation**. Its goal is not to force all REST APIs to become "purely RESTful," but rather:

1. To express API design intent and adaptation rules in a **semantically explicit, structurally stable** Canonical JSON document;
2. To support reverse-generation of that document from natural language or existing APIs (packet capture / documentation / OpenAPI);
3. To enable deterministic generation by the Tool layer of:
   - OpenAPI (when generatable)
   - Adaptation execution configuration (request assembly, response parsing, layered cooperation)
   - A human-review-friendly Markdown View
4. To provide **deviation tolerance** and **convention profiles** for real-world "non-standard/inconsistent" APIs, under the principle of "recordable, explainable, and optionally normalizable";
5. To provide the semantic skeleton (field constraints, operation semantics, compensation definitions) for RAIM-OP runtime execution.

> **Important distinctions:**
> - **RAIM Core**: Targets REST API modeling (design-time / reverse engineering); output is a RAIM Document.
> - **RAIM-OP**: Uses the RAIM Document as a semantic skeleton for runtime intent execution (natural language intent → parameter population → execution via RAIM adaptation information).
>
> Specific transport protocols (MCP, gRPC, HTTP, etc.) are outside the scope of this specification.

---

## Table of Contents

- [1. Specification Positioning and Principles](#1-specification-positioning-and-principles)
- [2. Problem Statement](#2-problem-statement)
- [3. RAIM Three-Layer Architecture](#3-raim-three-layer-architecture)
- [4. RAIM Document Format (Canonical JSON)](#4-raim-document-format-canonical-json)
  - [4.1 Top-Level Object](#41-top-level-object)
  - [4.2 Convention Profile](#42-convention-profile)
  - [4.3 Resources](#43-resources)
  - [4.4 Fields](#44-fields)
  - [4.5 Operations](#45-operations)
  - [4.6 Field Mapping](#46-field-mapping)
  - [4.7 Inter-Node Cooperation](#47-inter-node-cooperation-body-assembly-and-layered-parsing)
  - [4.8 Deviations](#48-deviations)
  - [4.9 Security](#49-security)
- [5. RAIM Markdown View (Human View)](#5-raim-markdown-view-human-view)
- [6. RAIM Skills](#6-raim-skills)
- [7. Version Management and Compatibility](#7-version-management-and-compatibility)
- [8. Evaluation, Boundaries, and Security](#8-evaluation-boundaries-and-security)
- [9. Appendix](#9-appendix)
  - [9.1 Appendix A: RAIM JSON Schema](#91-appendix-a-raim-json-schema-normative)
  - [9.2 Appendix B: End-to-End Example](#92-appendix-b-end-to-end-example-complete)
  - [9.3 Appendix C: Standard Error Codes](#93-appendix-c-standard-error-codes-normative)
  - [9.4 Appendix D: CDL Mapping](#94-appendix-d-raim-document-to-ain-capability-description-language-cdl-mapping-informative)
  - [9.5 Appendix E: Convention Profile Registry](#95-appendix-e-convention-profile-registry-informative)

---

## 1. Specification Positioning and Principles

RAIM Core defines a **semantic intermediate representation for REST APIs** and the boundaries between Skills and Tools that operate on that representation.

RAIM's core positioning:

1. **It is a framework**: It defines not only document formats but also the responsibility boundaries of Skills (conversation / summarization / reverse engineering) and Tools (validation / generation / transformation).
2. **It is a dual-format intermediate representation**: It provides both JSON (canonical) and Markdown (human view). JSON is the normative primary format; Markdown is a deterministically derivable view.
3. **It is not an alias for OpenAPI**: RAIM is semantically richer than OpenAPI, capable of expressing preconditions, side effects, state transitions, idempotency, deviation tolerance, and adaptation realities (success conditions in body, data paths, parsing strategies) — information that OpenAPI cannot directly express.

### 1.1 Requirements Language

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**, **SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this document are to be interpreted as described in BCP 14 (RFC 2119, RFC 8174).

### 1.2 Key Terminology

- **RAIM**: REST AI Interface Modeling. The framework and intermediate representation system.
- **RAIM Document**: A JSON object conforming to the RAIM Canonical JSON Schema.
- **Canonical Format**: RAIM's JSON representation. The internal canonical object for all Skills and Tools.
- **Markdown View**: A human-review-oriented deterministically derived view.
- **Convention Profile**: A configuration document defining default behaviors and common patterns (success determination, error format, pagination, naming mapping, etc.), reducing repetitive declarations per resource/operation and supporting inheritance.
- **Resource**: A REST semantic resource (collection / item / sub-resource / operation-group), the core modeling object in RAIM.
- **Operation**: An action on a resource (CRUD and non-CRUD), containing request/response adaptation information and operation semantics (preconditions, side effects, state transitions, idempotency).
- **Fields**: Semantic field definitions for a resource (type, constraints, examples, etc.), independent of field-mapping. Provides the basis for runtime parameter validation.
- **Field Mapping**: A declaration of correspondence between model fields and API fields.
- **Deviation**: A record of departure from "ideal REST conventions" (e.g., verb-in-path, non-standard error codes).
- **Precondition**: A condition that must be satisfied before an operation can be executed. Declared as a natural language description.
- **AI Confidence**: Confidence annotations (structural/semantic dimensions) produced by Skill D reverse engineering for fields and operation semantics.

### 1.3 Design Principles

1. **Semantics-first**: RAIM is not an alias for OpenAPI; it is semantically richer, emphasizing operation semantics (preconditions, side effects, state transitions, idempotency) and adaptation realities (success conditions, data paths, parsing strategies).
2. **Node-level definition + cooperative assembly**: Each resource/field/sub-resource defines only its own adaptation information; request dispatch and response parsing are assembled and hierarchically processed by Tools.
3. **Reality-tolerant but governable**: In the face of non-standard REST APIs, deviations must be recordable, and tolerance policies determine whether to record, normalize, or reject.
4. **Deterministic output**: Tool Layer generation must be as deterministic as possible; uncertain cases must produce structured error reports, entering reflection and retry or human supplementation flows.
5. **Explicit field semantics**: Field type constraints, visibility, and examples must be explicitly declared in Fields definitions, not inferred from API documentation descriptions. This is the core manifestation of RAIM being semantically richer than OpenAPI, and the foundation for RAIM-OP runtime parameter validation.
6. **Runtime-consumable**: The RAIM Document MUST, via Skill C, generate an execution configuration consumable by RAIM-OP; Fields semantic definitions and operation preconditions/side-effects MUST be structured and readable.

---

## 2. Problem Statement

### 2.1 Limitations of Generating OpenAPI Directly from Natural Language / Packet Capture

Existing AI tools can generate OpenAPI documentation directly from natural language descriptions or API packet captures, but this workflow has fundamental problems:

#### 2.1.1 Missing Operation Semantics

OpenAPI can describe an endpoint's HTTP method, parameters, and response status codes, but cannot express:

- Operation preconditions (e.g., "interface must be in admin-down state before deletion");
- Operation side effects (e.g., "creating an interface automatically assigns a MAC address");
- Operation idempotency (e.g., "is it safe to trigger multiple times");
- State transition semantics (e.g., "after creation, state transitions from pending to active").

This semantic information is critical for AI-assisted automated operations (RAIM-OP). If not explicitly captured during the modeling phase, parameter reasonability validation and operation safety judgments cannot be performed at runtime.

#### 2.1.2 Missing Success Determination and Data Paths

In reality, many vendor REST API success/failure determinations do not manifest at the HTTP status code level, but are distinguished through business codes in the body (e.g., `$.code == 0`); business data is also not at the response root node (e.g., `$.data`). OpenAPI cannot express this adaptation information, causing generated client code to be unable to correctly determine operation results.

#### 2.1.3 Unknown Field Mapping Symmetry

Without explicit field-mapping declarations, AI cannot determine:

- Whether request field `vlanId` and response field `vlanId` are fully symmetric;
- Whether `vlan-id` (model semantics) and `vlanId` (API format) are the same field;
- Which response fields are "system-generated" (response-only) and cannot be derived from the request.

#### 2.1.4 Lost Deviation Information

When generating OpenAPI from packet capture, verb-paths like `POST /interfaces:batch-delete` are preserved as-is, but the fact that they deviate from ideal REST conventions is not recorded, making unified governance during normalization impossible.

### 2.2 Current State of AI Understanding of REST API Semantics

In the absence of supplementary context, modern large models' understanding of REST API semantics is:

- Can relatively accurately identify the general function of endpoints (60-70%);
- Judgment of success conditions in body, data-path positions, and field mapping symmetry is unstable (<50%);
- Understanding of operation preconditions, side effects, and state transitions almost entirely depends on natural language documentation, and cannot be structurally extracted.

By structurally making relevant semantics explicit through RAIM, the actionable understanding of REST API semantics by AI is estimated to improve to 80-85%, particularly in runtime scenarios such as parameter validation, success determination, and field restoration.

### 2.3 RAIM's Value Proposition

RAIM's core value is not to replace OpenAPI, but to serve as a **semantic enhancement layer**:

1. **Design phase**: Captures "why the operation exists, how it fails, what it affects" earlier than OpenAPI;
2. **Adaptation phase**: Provides field-mapping, success determination, data paths, and other adaptation rules, reducing the vendor-difference cost of client development;
3. **Runtime phase**: Provides RAIM-OP with a complete semantic skeleton (field constraints, preconditions, compensation definitions), giving AI intent execution a safety boundary.

---

## 3. RAIM Three-Layer Architecture

### 3.1 Specification Layer

Defines:

- Canonical JSON structure and constraints;
- Convention Profile syntax and inheritance rules;
- Resource / Operation / Field / Mapping templates;
- Inter-node cooperation rules (body assembly, layered parsing);
- JSON ↔ Markdown deterministic conversion boundaries;
- AI Confidence field specification;
- OpenAPI generation reachability conditions.

### 3.2 Skill Layer

The Skill Layer defines AI-participating modeling workflows, including:

- **Skill A**: Conversational modeling (natural language → RAIM JSON)
- **Skill B**: Summary confirmation (RAIM JSON → natural language summary)
- **Skill C**: Generation (RAIM JSON + Convention Profile → OpenAPI / execution configuration / documentation)
- **Skill D**: Reverse engineering (existing API → RAIM JSON, with confidence annotations)

### 3.3 Tool Layer

The Tool Layer is responsible for deterministic processing of RAIM Documents:

- JSON Schema validation;
- Convention Profile merging;
- Fields constraint extraction and validation;
- `field-mapping` and `fields` association verification;
- Generating request assembler configuration (URL, parameters, templates, mount rules);
- Generating response parser configuration (success determination, data-path, parser);
- Generating OpenAPI (when reachability conditions are met);
- Generating Markdown View (including Resource Tree Diagram);
- Generating execution configuration for RAIM-OP consumption (including field constraints, preconditions, compensation definitions);
- Reverse structure extraction (from OpenAPI / packet capture / example JSON).

### 3.4 Validation-Reflection-Retry

When Tool Layer validation fails or deterministic generation is impossible (e.g., missing required fields, conflicting mappings, unresolvable data paths):

1. MUST output a Structured Error Report;
2. SHOULD inject Skill A context for automatic retry (RECOMMENDED maximum 3 times);
3. After exceeding the limit, MUST degrade to user-comprehensible follow-up questions and supplementary guidance.

A Structured Error Report MUST contain:

| Field | Description |
|---|---|
| `error_code` | Error type identifier (e.g., `SchemaValidationError`, `MissingRequiredField`, `MappingConflict`) |
| `path` | JSON Pointer to the erroneous field |
| `message` | Specific error description, semantically clear, directly understandable by AI |
| `expected` | Expected format or valid value reference |
| `severity` | `error` / `warn` |
| `hint` | Optional remediation suggestion |

### 3.5 End-to-End Workflow

RAIM's defined complete workflow:

1. User provides REST API natural language description / documentation link / packet capture / OpenAPI input;
2. Skill A (new) or Skill D (reverse) generates an initial RAIM Document draft and identifies missing semantics;
3. Skill A fills gaps through targeted questioning: resource hierarchy → operation list → field definitions → field-mapping → operation semantics (preconditions / side effects);
4. Skill B generates a natural language summary from the complete RAIM Document for user confirmation;
5. After user confirmation, Skill C invokes the Tool Layer to generate: OpenAPI / execution configuration / Markdown View;
6. Optionally: for RAIM-OP consumption, Skill C additionally outputs execution configuration (including field constraints, operation semantics, compensation definitions);
7. For existing OpenAPI / packet captures, optionally use Skill D to reverse-generate a RAIM Document and supplement AI-understandable semantics with confidence annotations.

> **Key principle: The RAIM Document is the canonical internal artifact. Markdown is only for display, review, or human editing, but the internal canonical object for all Tools and Skills is always JSON.**

---

## 4. RAIM Document Format (Canonical JSON)

### 4.1 Top-Level Object

The top-level object MUST be a JSON object and MUST contain at least:

- `version`
- `api`
- `resources`

And MAY contain:

- `convention-profile` (inline or referenced)
- `deviations`
- `common-fields` (reusable field definitions)
- `security`
- `tags`

#### 4.1.1 Top-Level Fields Overview

| Field | Type | Required | Description |
|---|---|---:|---|
| `version` | string | Yes | Specification version (SemVer), e.g., `"1.0"` |
| `api` | object | Yes | API metadata |
| `convention-profile` | object | No | Convention configuration (default behaviors), inline or via `profile-ref` |
| `resources` | array | Yes | Resource definition array |
| `deviations` | array | No | Global deviation records |
| `common-fields` | array | No | Reusable field definitions (referenceable by multiple resources) |
| `security` | object | No | Global authentication and authorization conventions (overridable per resource) |
| `tags` | array | No | Resource grouping tags |

#### 4.1.2 `api` Metadata

`api` MUST contain:

- `name`
- `description`

And MAY contain:

- `environments`: Multi-environment configuration (including labels and base-url)
- `versioning`
- `owners`

Example:

```json
{
  "api": {
    "name": "device-mgmt",
    "description": "Device management API for interface configuration",
    "environments": [
      { "name": "production", "base-url": "https://192.168.1.1" },
      { "name": "staging",    "base-url": "https://staging.192.168.1.1" }
    ],
    "versioning": { "strategy": "path", "value": "v1" },
    "owners": ["team-netops"]
  }
}
```

---

### 4.2 Convention Profile

The Convention Profile's goal is to **reduce the noise of repetitive declarations per resource/operation** and provide inheritable/overridable default policies.

#### 4.2.1 Structure

```json
{
  "convention-profile": {
    "profile": {
      "name": "pragmatic-rest-v1",
      "base": "strict-rest-v1",
      "strictness": "advisory"
    },
    "conventions": {
      "success": {
        "default-success-codes": "200-299",
        "default-success-condition": "$.code == 0",
        "default-error-message-path": "$.message"
      },
      "data": {
        "default-data-path": "$.data"
      },
      "naming": {
        "model-to-api": "camelCase"
      },
      "pagination": {
        "strategy": "cursor",
        "request": { "cursor-param": "after", "limit-param": "limit", "default-page-size": 100 },
        "response": { "next-cursor-path": "$.data.nextCursor", "total-path": "$.data.total" }
      },
      "auth": {
        "type": "bearer",
        "token-header": "Authorization",
        "token-prefix": "Bearer",
        "token-source": "env:API_TOKEN"
      }
    },
    "deviation-tolerance": {
      "verb-in-path": "record",
      "post-for-read": "record",
      "non-standard-error-schema": "warn",
      "missing-content-type": "warn"
    },
    "degradation-policy": {
      "auto-restore-fail": "field-mapping",
      "field-mapping-fail": "require-template"
    },
    "tool-hints": {
      "openapi-version": "3.1.0",
      "generate-examples": true,
      "example-strategy": "use-field-examples"
    }
  }
}
```

#### 4.2.2 Inheritance

`profile.base` allows a Convention Profile to inherit another Profile (referenced by `name`). Sub-profiles replace identically-named fields wholesale (replace semantics). The Tool resolves the inheritance chain at compile time.

#### 4.2.3 `deviation-tolerance` Behavior Definitions

The tolerance policy value for each deviation type:

| Value | Behavior |
|---|---|
| `record` | Record the deviation, continue generation; deviation information appended to `deviations[]` |
| `warn` | Output a warning in the Execution Report, continue generation |
| `normalize` | Generate a normalized version (per `deviation.canonical` field) alongside the original version (dual output), with deviation points annotated |
| `reject` | Reject generation/execution, MUST explain the reason in a Structured Error Report |

#### 4.2.4 `degradation-policy`

Declares the Tool's automatic degradation strategy when parser conditions are not satisfied. For example:

- `auto-restore-fail: "field-mapping"`: Degrade to field-mapping parsing
- `auto-restore-fail: "require-template"`: Require the user to provide a template
- `field-mapping-fail: "require-template"`: Require a template when field-mapping fails

Degradation results MUST be recorded in the Execution Report.

#### 4.2.5 Override and Scope

Convention Profile conventions serve as global defaults and can be overridden by the following fields (priority from low to high):

1. `convention-profile` (global default)
2. `resource.convention-overrides` (resource-level override)
3. `operation.convention-overrides` (operation-level override)

Override semantics RECOMMENDED as "whole-block replace" to avoid deep-merge ambiguity.

#### 4.2.6 Registry References

To promote interoperability, implementations SHOULD support referencing Convention Profiles from a community-maintained registry rather than requiring inline definition. A registry reference uses a `profile-ref` URI:

```json
{
  "convention-profile": {
    "profile": {
      "profile-ref": "registry:notion-api-v1"
    }
  }
}
```

The `registry:` scheme indicates that the profile is resolved from a well-known registry. See Appendix E for the recommended registry structure and a list of predefined profiles. Implementations MAY maintain their own private registries and MAY cache registry profiles locally.

The Tool MUST resolve the registry reference at RAIM Document load time. If the referenced profile cannot be resolved, the Tool MUST report a `ProfileResolutionError` and refuse generation.

---

### 4.3 Resources

Each resource object in `resources[]` MUST contain:

- `name`
- `resource-type`
- `path-template`
- `description`
- `operations`

And MAY contain:

- `parent` / `parent-binding` (hierarchical relationship)
- `identity` (primary key field)
- `fields` (field semantic definitions)
- `field-mapping` (field mapping)
- `convention-overrides`
- `body-cooperation` (body mount rules)
- `response-delegation` (layered parsing delegation)
- `ai-confidence` (confidence annotations from Skill D reverse engineering)
- `tags`

#### 4.3.1 `resource-type`

RECOMMENDED values:

| Type | Description | Example Path |
|---|---|---|
| `collection` | Collection resource | `/interfaces` |
| `item` | Entity resource | `/interfaces/{name}` |
| `sub-resource` | Sub-resource | `/interfaces/{name}/stats` |
| `operation-group` | Non-CRUD operation grouping; used for verb-path classification | `/interfaces/{name}:actions` |

`operation-group` is RAIM's tolerant handling of real-world REST API "verb-path operations": instead of forcing them back to CRUD, it classifies them as a resource object and defines each custom operation within the corresponding `operations`.

#### 4.3.2 Path Template Variable Syntax

`path-template` uses single-brace syntax (OpenAPI path parameter style) for path parameters:

```
/api/v1/namespaces/{ns}/interfaces/{name}
```

> **Note**: `path-template` uses `{varname}` single-brace syntax, while `body.template` uses `{{.varname}}` double-brace syntax. These are different rendering stages with different syntax and MUST NOT be mixed.
> - `path-template` / `url-template` (request path): `{varname}` (OpenAPI style, processed by path parameter renderer)
> - `body.template` (request body template): `{{.varname}}` (template engine style, processed by body template renderer)

#### 4.3.3 Hierarchical Relationships

In RAIM, resource hierarchy MUST be explicit (RAIM has no YANG semantic skeleton and cannot automatically derive parent-child relationships).

- `parent`: Parent resource `name`
- `parent-binding`: Parent-child binding method

```json
{
  "name": "interface",
  "resource-type": "item",
  "path-template": "/api/v1/namespaces/{ns}/interfaces/{name}",
  "parent": "interfaces",
  "parent-binding": {
    "path-contains": true,
    "inherit-params": ["ns"],
    "identity-param": "name"
  }
}
```

#### 4.3.4 `ai-confidence` (Confidence Annotations from Skill D)

Resources, operations, fields, and other objects MAY attach `ai-confidence` in the format:

```json
{
  "ai-confidence": {
    "structural": "high",
    "semantic": "review-required",
    "note": "Resource type inferred from path pattern; operation semantics need human review"
  }
}
```

| Field | Values | Meaning |
|---|---|---|
| `structural` | `high` / `medium` / `review-required` | Confidence in structural information (endpoint/method/path) |
| `semantic` | `high` / `medium` / `review-required` | Confidence in semantic information (preconditions/side-effects/field meanings) |
| `note` | string | Explanation of inference basis or items needing attention |

---

### 4.4 Fields

`fields[]` defines the field semantics of a resource, independent of `field-mapping`, providing the basis for runtime parameter validation (RAIM-OP) and documentation generation (Skill C).

Each field object MUST contain:

- `name`: Field name (model side, corresponds to `field-mapping.model-field`)
- `description`: Field semantic explanation

And MAY contain:

- `type`: Semantic type
- `required`: Whether required (default `false`)
- `constraints`: Value constraints (natural language or structured)
- `default`: Default value
- `examples`: Array of example values
- `visibility`: Visibility conditions
- `read-only`: Whether it is a response-only field
- `ai-confidence`: Field-level confidence annotation

RECOMMENDED `type` values: `string` / `uint8` / `uint16` / `uint32` / `uint64` / `int32` / `int64` / `boolean` / `number` / `array` / `object` / `enum` / `datetime` / `ipv4` / `ipv6` / `cidr` / `uuid`

#### 4.4.1 `common-fields` Reference Mechanism

Top-level `common-fields` allows the definition of reusable field groups. Within a resource's `fields[]` array, each entry can be:

- A regular field object (`fieldDef`);
- A reference object in the format `{ "$ref": "#/common-fields/{group-name}" }`, which the Tool MUST expand at compile time, inlining the corresponding field group's `fields[]` list.

```json
{
  "common-fields": [
    {
      "name": "common-metadata",
      "fields": [
        { "name": "created-at", "type": "datetime", "description": "Creation timestamp", "read-only": true },
        { "name": "updated-at", "type": "datetime", "description": "Last update timestamp", "read-only": true }
      ]
    }
  ],
  "resources": [
    {
      "name": "interface",
      "fields": [
        { "name": "name", "type": "string", "description": "Interface name" },
        { "$ref": "#/common-fields/common-metadata" }
      ]
    }
  ]
}
```

The `$ref` referenced `group-name` MUST exist in `common-fields[]`, otherwise the Tool MUST report a `MissingCommonFieldGroup` error.

#### 4.4.2 Relationship Between `field-mapping` and `fields`

`field-mapping[].model-field` values SHOULD have corresponding entries defined in the resource's `fields[]` or in `$ref`-expanded field lists. Fields in `fields[]` not referenced by any `field-mapping` entry are treated as documentation-only field definitions: they participate in RAIM-OP Hard Constraint validation but do not participate in request assembly or response parsing mapping.

---

### 4.5 Operations

Each resource MUST define `operations`, named semantically rather than forced into a one-to-one correspondence with HTTP methods.

RECOMMENDED operation keys: `create` / `get` / `list` / `update` / `patch` / `delete` / `custom` (array)

Each operation MUST contain at least: `request` / `response`

And SHOULD contain (semantic fields):

- `preconditions`
- `side-effects`
- `idempotency`

And MAY contain:

- `undo`
- `state-transition`
- `error-conditions`
- `convention-overrides`
- `ai-confidence`

#### 4.5.1 Operation Semantic Fields

| Field | Type | Description |
|---|---|---|
| `preconditions` | string or object | Preconditions for executing this operation; natural language description (string) or structured check definition (object, see §4.5.2) |
| `side-effects` | string | Impact on system state after successful operation |
| `idempotency` | boolean or `"conditional"` | Whether the operation is idempotent; `conditional` means it depends |
| `undo` | object | The reverse request definition for this operation, for RAIM-OP auto-discovery of compensation operations |
| `state-transition` | object | Resource state transition triggered by the operation (from → to) |
| `error-conditions` | array of string | List of known failure conditions |
| `async` | object | Asynchronous operation configuration (see §4.5.8) |

#### 4.5.2 `preconditions`

Preconditions MAY be expressed in one of two forms:

**Form A — Natural language (REQUIRED baseline)**: A string describing conditions that must be satisfied before the operation can execute:

```json
{
  "preconditions": "Interface must be in admin-down state before deletion"
}
```

When the precondition is a natural language string, RAIM-OP MUST display the content to the user and request manual confirmation before proceeding.

**Form B — Structured precondition (OPTIONAL)**: An object that enables automated precondition checking by the execution engine:

```json
{
  "preconditions": {
    "description": "Interface must be in admin-down state before deletion",
    "check": {
      "resource": "interface",
      "operation": "get",
      "params": {
        "ns": "$params.ns",
        "name": "$params.name"
      },
      "assert": "$.data.admin-status == 'down'"
    }
  }
}
```

Structured precondition fields:

| Field | Required | Description |
|---|---|---|
| `description` | MUST | Human-readable precondition description (used for display and as fallback if auto-check is unsupported) |
| `check.resource` | MUST | The RAIM Document resource name to query for state verification |
| `check.operation` | MUST | The operation name to use for the check query (typically `get` or `list`) |
| `check.params` | MUST | Parameters for the check query; `$params.{field}` references the current Operation IR's params |
| `check.assert` | MUST | A boolean expression evaluated against the check response; MUST evaluate to `true` for the precondition to be satisfied |

When a structured precondition is present and the execution engine supports automated checking:

1. The engine executes the `check` query against the specified resource and operation;
2. Evaluates the `assert` expression against the check response;
3. If `assert` evaluates to `true`, the precondition is satisfied — proceed;
4. If `assert` evaluates to `false`, the precondition is NOT satisfied — abort and report;
5. If the engine does not support automated checking or the check query fails, it MUST fall back to displaying `description` for manual confirmation.

RAIM-OP defines the execution engine behavior for precondition checking (see RAIM-OP §3.4).

#### 4.5.3 `undo` Field

`create` and `update` operations SHOULD define `undo`, describing the operation's reverse request:

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

`undo` object fields:

| Field | Required | Description |
|---|---|---|
| `description` | SHOULD | Human-readable rollback operation description |
| `operation` | MUST | Operation name executed during rollback (`delete` / `update` / other defined operation name) |
| `params-from-session` | SHOULD | Session Variable binding for rollback parameters; key is parameter name, value is `$session.{varname}` reference |
| `params-static` | MAY | Static parameter values (fixed parameters not dependent on Session Variables) |

If `undo` is not defined, RAIM-OP MUST record "no undo defined for operation" in the Execution Report and MUST NOT silently skip it.

#### 4.5.4 `on-fail` Semantics

| Value | Behavior |
|---|---|
| `"stop"` (default) | On operation failure, stop all subsequent operations, trigger undo for successfully completed operations in reverse order |
| `"continue"` | Record the current operation failure, do not trigger any undo, continue executing subsequent operations; summarize all failures in the Execution Report after the entire execution chain completes |
| `"compensate-and-continue"` | Trigger undo for the current operation (if defined); after compensation completes (whether successful or not), continue executing subsequent operations |

> **Note**: `"continue"` and `"compensate-and-continue"` have different semantics:
> - `"continue"` means "ignore the failure of this operation, preserve possible partial-completion state, continue forward";
> - `"compensate-and-continue"` means "first roll back the side effects of the current operation, then continue forward".

#### 4.5.5 Request Body Definition (request)

`request` SHOULD support the following fields:

| Field | Required | Description |
|---|---|---|
| `method` | MUST | HTTP method |
| `url-template` | MUST | URL template, path variables use `{varname}` single-brace syntax (OpenAPI style); MUST NOT use `{{.varname}}` double-brace syntax in url-template |
| `headers` | SHOULD | Request headers |
| `query-params` | MAY | Query parameter definitions |
| `path-params` | MAY | Path parameter descriptions (documentation) |
| `body` | MAY | Body definition (can be omitted for GET/DELETE) |
| `timeout` | MAY | Timeout override (seconds) |

`body` definition, using `{{.varname}}` double-brace template syntax:

```json
{
  "body": {
    "type": "template",
    "template-language": "naim-template-v1",
    "template": "{ \"name\": \"{{.name}}\", \"mtu\": {{.mtu}}, \"vlanId\": {{.vlan-id}} }"
  }
}
```

**Variable syntax instructions**:

- `url-template` uses `{varname}` (single brace, OpenAPI path parameter style): processed by the path parameter renderer;
- `body.template` uses `{{.varname}}` (double brace, template engine style): processed by the body template renderer, supports all field types and Session Variable (`{{.$session.varname}}`) references.

These are independent rendering stages and MUST NOT be mixed. If `{{.varname}}` appears in `url-template`, the Tool MUST report `TemplateSyntaxError`.

#### 4.5.6 Response Body Definition (response)

`response` MUST contain `parser`, and SHOULD contain:

| Field | Description |
|---|---|
| `success-codes` | HTTP status code range (e.g., `"200-299"`) |
| `success-condition` | Business code determination expression (e.g., `"$.code == 0"`) |
| `error-message-path` | Error message extraction path |
| `data-path` | Path where business data resides |
| `parser` | Parsing strategy: `auto` / `field-mapping` / `template` / `script` |
| `capture` | Response field capture |

#### 4.5.7 `capture` and Session Variable Naming

`capture` specification:

```json
{
  "capture": [
    { "name": "resource-uuid", "data-path": "$.data.uuid", "required": true }
  ]
}
```

**Session Variable naming source**: `capture[].name` is the Session Variable's key name. It is referenced in subsequent `url-template` via `{$session.resource-uuid}` and in `body.template` via `{{.$session.resource-uuid}}`.

> **Rule**: The Session Variable key name source is unique — it can only be the value of `capture[].name`. `$session.*` variables not declared through `capture` MUST NOT be used in `url-template` or `body.template`.

#### 4.5.8 `async` — Asynchronous Operation Support

Some API operations are asynchronous: the initial request returns an acknowledgement with a task/polling identifier, and the final result must be retrieved through subsequent polling.

When an operation supports asynchronous execution, the `async` object SHOULD be defined within the operation:

```json
{
  "create": {
    "request": { "method": "POST", "url-template": "/api/v1/namespaces/{ns}/interfaces" },
    "response": {
      "success-codes": "200,201,203-205",
      "parser": "auto",
      "capture": [{ "name": "task-id", "data-path": "$.data.taskId", "required": true }]
    },
    "async": {
      "enabled": true,
      "poll-url": "/api/v1/tasks/{task-id}",
      "poll-interval": 5,
      "max-wait": 300,
      "completion-condition": "$.data.status == 'completed'",
      "failure-condition": "$.data.status == 'failed'"
    }
  }
}
```

`async` object fields:

| Field | Required | Description |
|---|---|---|
| `enabled` | MUST | `true` if the operation is asynchronous |
| `poll-url` | MUST | URL template for polling; path variables reference `capture` names from the initial response (e.g., `{task-id}` resolves to the captured `task-id` Session Variable) |
| `poll-interval` | SHOULD | Polling interval in seconds (default: `5`) |
| `max-wait` | SHOULD | Maximum total wait time in seconds before timing out (default: `300`) |
| `completion-condition` | MUST | Expression evaluated against polling response; when `true`, the operation is considered complete |
| `failure-condition` | SHOULD | Expression evaluated against polling response; when `true`, the operation is considered failed |

The RAIM-OP execution engine handles polling behavior based on these declarations (see RAIM-OP §5.3).

---

### 4.6 Field Mapping

#### 4.6.1 `field-mapping` Structure

Each field-mapping item MUST contain `api-field`, and SHOULD contain `direction`:

| Field | Required | Description |
|---|---|---|
| `model-field` | SHOULD | Business/model field name (SHOULD exist in `fields[]`); `null` indicates no corresponding model field (response-only field) |
| `api-field` | MUST | API field name or JSONPath |
| `direction` | SHOULD | `both` / `request-only` / `response-only`; defaults to `both` |
| `symmetric` | SHOULD | Symmetric declaration (default `false`); when `true`, indicates the field has symmetric request/response forms |
| `transform` | MAY | Optional transformation; natural language description or structured expression |
| `capture-as` | MAY | Session Variable name for response-only fields |

#### 4.6.2 Response Parsing Strategies

RAIM supports multiple response parsing strategies, declared via `response.parser`:

| Parser | Description |
|---|---|
| `auto` | Automatic restoration based on field-mapping symmetry declarations |
| `field-mapping` | Extract field values per `field-mapping[].{api-field → model-field}` mappings |
| `template` | Extract via reverse template |
| `script` | Execute a script in a sandbox, returning field key-value pairs |

The Tool selects the appropriate parser based on the declared parser and Convention Profile settings.

---

### 4.7 Inter-Node Cooperation: Body Assembly and Layered Parsing

#### 4.7.1 Request Body Assembly (Sub-Resource Mount)

Sub-resources MAY define `body-cooperation.body-mount`:

```json
{
  "body-cooperation": {
    "body-mount": {
      "mount-to-resource": "interfaces",
      "mount-to-operation": "create",
      "mount-path": "$.interfaces.interface[]",
      "mode": "append-array"
    }
  }
}
```

`body-mount` fields:

| Field | Required | Description |
|---|---|---|
| `mount-to-resource` | MUST | Parent resource `name` |
| `mount-to-operation` | MUST | Which parent resource operation needs mounting |
| `mount-path` | MUST | JSONPath expression for mounting location in parent body |
| `mode` | SHOULD | `merge-object` / `append-array` / `set-field`; defaults to `merge-object` |

If a virtual request needs to include a resource's body:

```json
{
  "body-cooperation": {
    "virtual-request": {
      "name": "batch-create",
      "method": "POST",
      "url-template": "/api/v1/namespaces/{ns}/interfaces/batch",
      "body-template": { "type": "json-object", "template": "{}" },
      "response": { "success-codes": "200-299", "parser": "auto" },
      "mounts": ["interface"]
    }
  }
}
```

#### 4.7.2 Response Layered Parsing (Delegation)

`response-delegation` field:

```json
{
  "response-delegation": {
    "child-resource": "interface",
    "data-path": "$.data.interfaces[*]",
    "trigger": ["list", "get"]
  }
}
```

---

### 4.8 Deviations

#### 4.8.1 Deviation Records

`deviations[]` records differences between real-world APIs and ideal conventions.

Each deviation item RECOMMENDED fields:

| Field | Required | Description |
|---|---|---|
| `type` | MUST | Deviation type (`verb-in-path` / `post-for-read` / `non-standard-success` / `custom-error-envelope` / `missing-content-type` / `non-json-body` / `custom`) |
| `resource` | SHOULD | Resource name where deviation occurs |
| `operation` | MAY | Operation name where deviation occurs |
| `observed` | SHOULD | Observed facts |
| `canonical` | SHOULD | Normalization suggestion |
| `normalize` | SHOULD | Whether to normalize during generation |
| `ai-confidence` | MAY | Deviation identification confidence |

---

### 4.9 Security

```json
{
  "security": {
    "type": "bearer | basic | apikey | cookie | mtls | none",
    "token-header": "Authorization",
    "token-prefix": "Bearer",
    "token-source": "env:API_TOKEN | session:token",
    "token-refresh": {
      "endpoint": "/api/v1/auth/refresh",
      "method": "POST",
      "token-path": "$.data.accessToken",
      "refresh-token-source": "env:REFRESH_TOKEN"
    }
  }
}
```

Security conventions MUST NOT hardcode credentials; `token-source` must reference environment variables or a key management service.

#### 4.9.1 Authentication Types

| Type | Description | Additional Fields |
|---|---|---|
| `bearer` | Bearer token in Authorization header | `token-header`, `token-prefix`, `token-source` |
| `basic` | HTTP Basic Authentication | `username-source`, `password-source` (both env references) |
| `apikey` | API key in header or query parameter | `api-key-header` (for header) or `api-key-query-param` (for query parameter), `api-key-source` |
| `cookie` | Cookie-based session authentication | `cookie-name`, `cookie-source` |
| `mtls` | Mutual TLS with client certificate | `cert-source`, `key-source` (both env references to file paths) |
| `none` | No authentication | — |

#### 4.9.2 API Key in Query Parameters

Many real-world REST APIs pass API keys as query parameters rather than headers. The `apikey` type supports both modes:

```json
{
  "security": {
    "type": "apikey",
    "api-key-query-param": "api_key",
    "api-key-source": "env:API_KEY"
  }
}
```

If both `api-key-header` and `api-key-query-param` are present, the key is sent in both locations.

#### 4.9.3 Token Refresh

When `token-refresh` is defined, the execution engine SHOULD:

1. Detect token expiry (HTTP 401 or specific error condition from `success-condition`);
2. Execute the token refresh request using the refresh token from `refresh-token-source`;
3. Extract the new token from the refresh response using `token-path`;
4. Retry the original request with the new token;
5. If the refresh also fails, abort and report.

#### 4.9.4 Security Override Scope

Security conventions serve as global defaults and can be overridden:

1. `convention-profile.conventions.auth` (global default)
2. `resources[].security-override` (resource-level override)
3. `resources[].operations.*.convention-overrides.auth` (operation-level override)

Override semantics use "whole-block replace" to avoid deep-merge ambiguity.

---

## 5. RAIM Markdown View (Human View)

### 5.1 Round-trip Safety Requirements

JSON → Markdown → JSON conversion MUST satisfy:

- No semantic loss (all RAIM Document JSON fields have corresponding Markdown expressions);
- No structural ambiguity (each Markdown block can deterministically correspond to a RAIM JSON object);
- Non-semantic field order variation is permitted;
- Resource hierarchy can be stably reconstructed (via `parent` field rather than heading nesting level).

The Tool Layer MUST ensure round-trip through:

1. Each Resource must have a unique `name`, serving as the Markdown section anchor;
2. `field-mapping` and `fields` association must be determined via `model-field` name rather than positional indexing;
3. `ai-confidence` fields are expressed in Markdown as comments or tags, not affecting structural parsing.

#### 5.1.1 Deterministic Round-trip Algorithm (RECOMMENDED)

The following algorithm ensures lossless JSON ↔ Markdown ↔ JSON conversion. Implementations MAY use alternative approaches provided they satisfy the round-trip safety requirements above.

**JSON → Markdown (serialization):**

1. Emit frontmatter block: `# api: {api.name}`, `## description`, `## version`, `## convention-profile`
2. Emit environments table from `api.environments[]`
3. Build resource tree from `resources[]` using `parent` relationships:
   - Root: resources with no `parent` field
   - Children: resources with `parent` matching a root resource `name`
4. Emit resource tree diagram using indent-based hierarchy
5. For each resource (in tree order, depth-first):
   a. Emit `### {resource.name} ({resource-type})` heading
   b. Emit fields in declared order: `path-template`, `description`, `parent`, `identity`, `fields`, `operations`, `field-mapping`, `body-cooperation`, `response-delegation`, `convention-overrides`, `deviations`, `ai-confidence`
   c. For `fields[]`: emit table with columns name/type/required/description
   d. For each operation: emit `#### {operation-name}` subheading, tabular request/response, list semantic fields (preconditions/side-effects/idempotency/undo/on-fail)
   e. For `field-mapping[]`: emit table with columns model-field/api-field/direction/symmetric/transform
6. Emit global `deviations[]` as a separate flat section
7. Emit `security` as a separate section

**Markdown → JSON (deserialization):**

1. Parse frontmatter block → `version`, `api.name`, `api.description`
2. Parse environments table → `api.environments[]`
3. Scan all `### {name} ({type})` headings → build flat resource list with `name` and `resource-type`
4. For each resource section, sequentially parse subsections into corresponding JSON fields
5. Parse `### {name}` heading → `resource.name`
6. Parse `path-template` line → `resource.path-template`
7. Parse parent reference → `resource.parent`
8. Parse fields table → `resource.fields[]`
9. For each `#### {op-name}` subsection → parse request/response tables, semantic fields → `resource.operations.{op-name}`
10. Parse field-mapping table → `resource.field-mapping[]`
11. Reconstruct `convention-profile` from its dedicated section

**Conflict resolution**: If a field appears in both JSON and Markdown forms, JSON is authoritative. If a field in Markdown has no JSON equivalent, it is treated as a documentation-only annotation.

### 5.2 Header Rules

```markdown
# api: {api.name}
## description: {api.description}
## version: {version}
## convention-profile: {convention-profile.profile.name}
## environments
| name | base-url |
|------|----------|
| production | https://... |
```

### 5.3 Resource Tree Diagram

```text
api: device-mgmt
  + interfaces (collection)  GET /api/v1/namespaces/{ns}/interfaces
    + interface (item)       CRUD /api/v1/namespaces/{ns}/interfaces/{name}
      + interface-stats (sub-resource)  GET .../{name}/stats
      + interface-actions (operation-group)
          - restart  POST .../{name}/actions/restart
```

### 5.4 Flat Resource Definitions

Resource definitions adopt a flat list, with each resource starting with `### {resource.name} ({resource-type})`.

RECOMMENDED field order:

1. `path-template`
2. `description`
3. `parent`
4. `identity`
5. `fields`
6. `operations` (each operation as a subsection)
7. `field-mapping`
8. `body-cooperation` / `response-delegation`
9. `convention-overrides`
10. `deviations`
11. `ai-confidence`

---

## 6. RAIM Skills

### 6.1 Skill A: Conversational API Modeling

**Goal**: Natural language → RAIM Document JSON

**Workflow steps** (RECOMMENDED):

**Phase 1: Entry confirmation**

1. First confirm whether this is a new API specification or reverse completion based on an existing API;
2. If new, first ask the user to provide API name and description;
3. If reverse completion, transition to Skill D workflow, then return to Skill A to supplement semantics.

**Phase 2: Resource hierarchy establishment**

4. First ask about the resource list;
5. Confirm each resource's `resource-type` one by one;
6. Clarify parent-child relationships between resources;
7. Detect "verb-path operations" and automatically classify them as `operation-group` (annotated as deviations).

**Phase 3: Operations and request/response**

8. Ask about only one resource's operations at a time;
9. Ask: which operations are supported;
10. Ask: HTTP method and URL for each operation (reminder: URL uses `{varname}` path parameter format);
11. Ask: success determination;
12. Ask: response data path (`data-path`);
13. Ask: whether pagination is present.

**Phase 4: Fields and mapping**

14. Ask: what fields the resource has;
15. Ask: whether field model names and API names are consistent (automatically generate transform when naming differences are discovered);
16. Ask: which fields are response-only;
17. Attempt to infer symmetry, ask user for confirmation;
18. If inference is not possible, explicitly ask whether template/script parsing is needed.

**Phase 5: Operation semantics**

19. Ask: preconditions for each write operation;
20. Ask: side effects of each write operation;
21. Ask: whether the operation is idempotent;
22. Ask: whether `create`/`update` needs `undo` defined;
23. Ask: whether there are known failure conditions.

**Phase 6: Confirmation and output**

24. Skill B summary confirmation;
25. After confirmation, first ask whether to synchronously generate Markdown View;
26. Finally output RAIM Canonical JSON.

### 6.2 Skill B: Natural Language Summarization

Output summary SHOULD emphasize:

- What resources exist and their hierarchical relationships;
- What operations each resource supports and what HTTP methods are used;
- Which operations are asynchronous / have side effects / are non-idempotent;
- Success determination and error semantics;
- Which fields can use auto parsing and which must use explicit parsing;
- Detected deviation types and handling strategies;
- Which operations have `undo` defined and which do not (risk prompt);
- Closing with a user confirmation request.

### 6.3 Skill C: Spec Generation (OpenAPI / Adaptation Configuration)

**Input**: RAIM Document JSON + Convention Profile

**Output (optional multi-artifact)**:

- OpenAPI 3.x (when reachability conditions are met)
- Execution configuration (request assembly + response parsing + layered cooperation)
- Execution package for RAIM-OP consumption (including field constraints, operation semantics, compensation definitions)
- Markdown View
- Generation report (including degradation reasons, deviation records, confidence warnings)

#### 6.3.1 OpenAPI Reachability Conditions

The Tool MUST be able to generate standard OpenAPI 3.x under the following conditions:

- All `resource-type` values are not `operation-group` (or have been normalized);
- All operation success determinations depend only on HTTP codes;
- All field-mappings do not require script parsing;
- All deviations with `normalize: true` have corresponding `canonical` fields.

If conditions are not satisfied, the Tool MUST output partial OpenAPI + a list of unreachable reasons.

#### 6.3.2 RAIM-OP Execution Package

When Skill C generates for RAIM-OP consumption, the output SHOULD additionally include:

- Field constraints for each resource (from `fields[]` + `field-mapping`);
- `preconditions` for each write operation;
- `capture` rules for each operation;
- `undo` definitions for each `create`/`update` operation.

### 6.4 Skill D: Reverse Engineering

Skill D has two steps:

**Step 1: Structure extraction (Tool, high confidence)**: endpoints, methods, URL parameters, request body field structure, response field structure, data-path guesses, HTTP status codes, deviation detection.

**Step 2: Semantic completion (AI, low confidence)**: resource-type inference, field meanings, operation preconditions, side effects, symmetry inference, deviation explanation.

Output MUST annotate `ai-confidence`, and highlight `semantic: "review-required"` portions in the Markdown View.

---

## 7. Version Management and Compatibility

### 7.1 RAIM Document Version Management

The top-level `version` field uses SemVer.

### 7.2 Convention Profile Version Management

The Profile's `profile.name` SHOULD include version information (e.g., `pragmatic-rest-v1`).

### 7.3 API Version and RAIM Document Compatibility

When a vendor API is upgraded, SHOULD automatically check via CI whether the RAIM Document's `url-template`, `field-mapping`, and `success-condition` are still consistent with the new API specification. If incompatibility is detected, SHOULD add a `type: "api-version-upgrade"` entry in `deviations[]`.

### 7.4 `ai-confidence` Lifecycle

`ai-confidence: "review-required"` fields generated by Skill D SHOULD be updated to `"high"` after human review and confirmation. In version management systems, changes to the `ai-confidence` field serve as traceable evidence of "human confirmation".

---

## 8. Evaluation, Boundaries, and Security

### 8.1 Evaluation Dimensions

| Evaluation Item | Method |
|---|---|
| Operation semantic completeness | Whether write operations have preconditions / side-effects / idempotency |
| `undo` coverage | Whether create/update operations all have undo defined |
| Field semantic completeness | Whether `fields[]` covers all model-field entries in `field-mapping` |
| Auto parsing applicability | Proportion of symmetric fields, whether degradation is needed |
| Structured precondition coverage | Proportion of preconditions in structured Form B (auto-checkable) vs Form A natural language |
| Async operation support | Whether asynchronous operations have complete `async` configuration (poll-url, completion-condition, failure-condition) |
| Confidence coverage | Among Skill D generated entries, the proportion of review-required items |
| Deviation governance rate | Among normalize=true deviations, the completeness rate of canonical fields |

### 8.2 Boundaries

RAIM does not guarantee lossless mapping of all real-world REST APIs to OpenAPI; the following situations may only produce execution configurations: non-standard success determination, complex parsing heavily dependent on scripts, asynchronous flows spanning multiple endpoint collaborations, multi-phase commit/rollback semantics.

### 8.3 Security

- RAIM Documents may contain sensitive URLs, token rules, device capabilities, and side-effect information;
- AI-generated parsing scripts and templates must be audited;
- `security.token-source` MUST NOT hardcode credentials;
- Reverse-engineered packet capture input must undergo data sanitization and access control.

---

## 9. Appendix

### 9.1 Appendix A: RAIM JSON Schema (Normative)

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "RAIM Core Document Schema v1.0",
  "type": "object",
  "required": ["version", "api", "resources"],
  "additionalProperties": false,
  "properties": {
    "version":  { "type": "string" },
    "api":      { "$ref": "#/definitions/apiMeta" },
    "convention-profile": { "$ref": "#/definitions/conventionProfile" },
    "resources": {
      "type": "array",
      "minItems": 1,
      "items": { "$ref": "#/definitions/resource" }
    },
    "deviations": { "type": "array", "items": { "$ref": "#/definitions/deviation" } },
    "common-fields": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["name", "fields"],
        "additionalProperties": false,
        "properties": {
          "name":   { "type": "string" },
          "fields": { "type": "array", "items": { "$ref": "#/definitions/fieldDef" } }
        }
      }
    },
    "security": { "$ref": "#/definitions/securitySpec" },
    "tags":     { "type": "array", "items": { "type": "string" } }
  },
  "definitions": {
    "apiMeta": {
      "type": "object",
      "required": ["name", "description"],
      "additionalProperties": false,
      "properties": {
        "name":        { "type": "string" },
        "description": { "type": "string" },
        "environments": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["name", "base-url"],
            "additionalProperties": false,
            "properties": {
              "name":     { "type": "string" },
              "base-url": { "type": "string" }
            }
          }
        },
        "versioning": { "type": "object" },
        "owners":     { "type": "array", "items": { "type": "string" } }
      }
    },
    "conventionProfile": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "profile": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "name":       { "type": "string" },
            "base":       { "type": "string" },
            "strictness": { "type": "string", "enum": ["strict", "advisory", "permissive"] }
          }
        },
        "conventions": { "type": "object" },
        "deviation-tolerance": {
          "type": "object",
          "additionalProperties": { "type": "string", "enum": ["record", "warn", "normalize", "reject"] }
        },
        "degradation-policy": { "type": "object" },
        "tool-hints":          { "type": "object" }
      }
    },
    "resource": {
      "type": "object",
      "required": ["name", "resource-type", "path-template", "description", "operations"],
      "additionalProperties": false,
      "properties": {
        "name":          { "type": "string" },
        "resource-type": { "type": "string", "enum": ["collection", "item", "sub-resource", "operation-group"] },
        "path-template": { "type": "string" },
        "description":   { "type": "string" },
        "parent":        { "type": "string" },
        "parent-binding":{ "type": "object" },
        "identity":      { "type": "string" },
        "fields": {
          "type": "array",
          "items": {
            "oneOf": [
              { "$ref": "#/definitions/fieldDef" },
              {
                "type": "object",
                "required": ["$ref"],
                "additionalProperties": false,
                "properties": {
                  "$ref": { "type": "string", "pattern": "^#/common-fields/.+" }
                }
              }
            ]
          }
        },
        "operations":    { "$ref": "#/definitions/operationsMap" },
        "field-mapping": { "type": "array", "items": { "$ref": "#/definitions/fieldMappingItem" } },
        "body-cooperation":    { "$ref": "#/definitions/bodyCooperation" },
        "response-delegation": { "$ref": "#/definitions/responseDelegation" },
        "convention-overrides":{ "type": "object" },
        "security-override":   { "$ref": "#/definitions/securitySpec" },
        "ai-confidence":       { "$ref": "#/definitions/aiConfidence" },
        "tags":                { "type": "array", "items": { "type": "string" } }
      }
    },
    "fieldDef": {
      "type": "object",
      "required": ["name", "description"],
      "additionalProperties": false,
      "properties": {
        "name":        { "type": "string" },
        "description": { "type": "string" },
        "type": {
          "type": "string",
          "enum": ["string","uint8","uint16","uint32","uint64","int32","int64","boolean","number","array","object","enum","datetime","ipv4","ipv6","cidr","uuid"]
        },
        "required":    { "type": "boolean", "default": false },
        "read-only":   { "type": "boolean", "default": false },
        "constraints": true,
        "default":     true,
        "examples":    { "type": "array" },
        "visibility":  { "type": "string" },
        "ai-confidence": { "$ref": "#/definitions/aiConfidence" }
      }
    },
    "operationsMap": {
      "type": "object",
      "properties": {
        "create": { "$ref": "#/definitions/operation" },
        "get":    { "$ref": "#/definitions/operation" },
        "list":   { "$ref": "#/definitions/operation" },
        "update": { "$ref": "#/definitions/operation" },
        "patch":  { "$ref": "#/definitions/operation" },
        "delete": { "$ref": "#/definitions/operation" },
        "custom": {
          "type": "array",
          "items": {
            "allOf": [
              { "$ref": "#/definitions/operation" },
              { "required": ["name"], "properties": { "name": { "type": "string" } } }
            ]
          }
        }
      }
    },
    "operation": {
      "type": "object",
      "required": ["request", "response"],
      "additionalProperties": false,
      "properties": {
        "request":  { "$ref": "#/definitions/opRequest" },
        "response": { "$ref": "#/definitions/opResponse" },
        "undo":     { "$ref": "#/definitions/undoDef" },
        "preconditions": {
          "oneOf": [
            { "type": "string" },
            {
              "type": "object",
              "required": ["description", "check"],
              "additionalProperties": false,
              "properties": {
                "description": { "type": "string" },
                "check": {
                  "type": "object",
                  "required": ["resource", "operation", "params", "assert"],
                  "additionalProperties": false,
                  "properties": {
                    "resource":  { "type": "string" },
                    "operation": { "type": "string" },
                    "params":    { "type": "object" },
                    "assert":    { "type": "string" }
                  }
                }
              }
            }
          ]
        },
        "side-effects":   { "type": "string" },
        "idempotency": {
          "oneOf": [{ "type": "boolean" }, { "type": "string", "enum": ["conditional"] }]
        },
        "state-transition": { "type": "object" },
        "error-conditions": { "type": "array", "items": { "type": "string" } },
        "on-fail": {
          "type": "string",
          "enum": ["stop", "continue", "compensate-and-continue"],
          "default": "stop"
        },
        "async": {
          "type": "object",
          "required": ["enabled", "poll-url", "completion-condition"],
          "additionalProperties": false,
          "properties": {
            "enabled":              { "type": "boolean" },
            "poll-url":             { "type": "string" },
            "poll-interval":        { "type": "number", "minimum": 1, "default": 5 },
            "max-wait":             { "type": "number", "minimum": 1, "default": 300 },
            "completion-condition": { "type": "string" },
            "failure-condition":    { "type": "string" }
          }
        },
        "convention-overrides": { "type": "object" },
        "ai-confidence": { "$ref": "#/definitions/aiConfidence" }
      }
    },
    "undoDef": {
      "type": "object",
      "required": ["operation"],
      "additionalProperties": false,
      "properties": {
        "description":         { "type": "string" },
        "operation":           { "type": "string" },
        "params-from-session": { "type": "object", "additionalProperties": { "type": "string" } },
        "params-static":       { "type": "object" }
      }
    },
    "opRequest": {
      "type": "object",
      "required": ["method"],
      "additionalProperties": false,
      "properties": {
        "method":       { "type": "string", "enum": ["GET","POST","PUT","PATCH","DELETE"] },
        "url-template": { "type": "string" },
        "headers":      { "type": "object", "additionalProperties": { "type": "string" } },
        "query-params": { "type": "object" },
        "path-params":  { "type": "object" },
        "body":         { "$ref": "#/definitions/bodySpec" },
        "timeout":      { "type": "number", "minimum": 1 }
      }
    },
    "opResponse": {
      "type": "object",
      "required": ["parser"],
      "additionalProperties": false,
      "properties": {
        "success-codes":      { "type": "string" },
        "success-condition":  { "type": "string" },
        "error-message-path": { "type": "string" },
        "data-path":          { "type": "string" },
        "parser":             { "type": "string", "enum": ["auto", "field-mapping", "template", "script"] },
        "template":           { "type": "object" },
        "script":             { "$ref": "#/definitions/scriptSpec" },
        "capture": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["name", "data-path"],
            "additionalProperties": false,
            "properties": {
              "name":      { "type": "string" },
              "data-path": { "type": "string" },
              "required":  { "type": "boolean", "default": true }
            }
          }
        }
      }
    },
    "bodySpec": {
      "type": "object",
      "required": ["type"],
      "additionalProperties": false,
      "properties": {
        "type":              { "type": "string", "enum": ["template","json-object","none"] },
        "template-language": { "type": "string" },
        "template":          { "type": "string" }
      }
    },
    "scriptSpec": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "language": { "type": "string", "enum": ["python","javascript"] },
        "source":   { "type": "string" }
      }
    },
    "fieldMappingItem": {
      "type": "object",
      "required": ["api-field"],
      "additionalProperties": false,
      "properties": {
        "model-field": true,
        "api-field":   { "type": "string" },
        "direction":   { "type": "string", "enum": ["both","request-only","response-only"], "default": "both" },
        "symmetric":   { "type": "boolean", "default": false },
        "transform":   true,
        "capture-as":  { "type": "string" }
      }
    },
    "bodyCooperation": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "body-mount":    { "$ref": "#/definitions/bodyMount" },
        "virtual-request": { "$ref": "#/definitions/virtualRequest" }
      }
    },
    "bodyMount": {
      "type": "object",
      "required": ["mount-to-resource", "mount-to-operation", "mount-path"],
      "additionalProperties": false,
      "properties": {
        "mount-to-resource":  { "type": "string" },
        "mount-to-operation": { "type": "string" },
        "mount-path":         { "type": "string" },
        "mode":               { "type": "string", "enum": ["merge-object","append-array","set-field"], "default": "merge-object" }
      }
    },
    "virtualRequest": {
      "type": "object",
      "required": ["name","method","url-template","response"],
      "additionalProperties": false,
      "properties": {
        "name":          { "type": "string" },
        "method":        { "type": "string", "enum": ["GET","POST","PUT","PATCH","DELETE"] },
        "url-template":  { "type": "string" },
        "body-template": { "$ref": "#/definitions/bodySpec" },
        "response":      { "$ref": "#/definitions/opResponse" },
        "mounts":        { "type": "array", "items": { "type": "string" } }
      }
    },
    "responseDelegation": {
      "type": "object",
      "required": ["child-resource", "data-path"],
      "additionalProperties": false,
      "properties": {
        "child-resource": { "type": "string" },
        "data-path":      { "type": "string" },
        "trigger":        { "type": "array", "items": { "type": "string" } }
      }
    },
    "deviation": {
      "type": "object",
      "required": ["type"],
      "additionalProperties": false,
      "properties": {
        "type":      { "type": "string" },
        "resource":  { "type": "string" },
        "operation": { "type": "string" },
        "observed":  true,
        "canonical": true,
        "normalize": { "type": "boolean" },
        "ai-confidence": { "$ref": "#/definitions/aiConfidence" }
      }
    },
    "securitySpec": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "type":              { "type": "string", "enum": ["bearer","basic","apikey","cookie","mtls","none"] },
        "token-header":      { "type": "string" },
        "token-prefix":      { "type": "string" },
        "token-source":      { "type": "string" },
        "username-source":   { "type": "string" },
        "password-source":   { "type": "string" },
        "api-key-header":    { "type": "string" },
        "api-key-query-param": { "type": "string" },
        "api-key-source":    { "type": "string" },
        "cookie-name":       { "type": "string" },
        "cookie-source":     { "type": "string" },
        "cert-source":       { "type": "string" },
        "key-source":        { "type": "string" },
        "token-refresh": {
          "type": "object",
          "required": ["endpoint", "method", "token-path"],
          "additionalProperties": false,
          "properties": {
            "endpoint":             { "type": "string" },
            "method":               { "type": "string", "enum": ["GET","POST"] },
            "token-path":           { "type": "string" },
            "refresh-token-source": { "type": "string" }
          }
        }
      }
    },
    "aiConfidence": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "structural": { "type": "string", "enum": ["high","medium","review-required"] },
        "semantic":   { "type": "string", "enum": ["high","medium","review-required"] },
        "note":       { "type": "string" }
      }
    }
  }
}
```

### 9.2 Appendix B: End-to-End Example (Complete)

```json
{
  "version": "1.0",
  "api": {
    "name": "example-device-api",
    "description": "Example REST API for interface management",
    "environments": [
      { "name": "production", "base-url": "https://192.168.1.1" }
    ],
    "versioning": { "strategy": "path", "value": "v1" }
  },
  "convention-profile": {
    "profile": { "name": "vendor-pragmatic-v1", "strictness": "advisory" },
    "conventions": {
      "success": {
        "default-success-codes": "200-299",
        "default-success-condition": "$.code == 0",
        "default-error-message-path": "$.message"
      },
      "data":   { "default-data-path": "$.data" },
      "naming": { "model-to-api": "camelCase" },
      "auth":   { "type": "bearer", "token-source": "env:API_TOKEN" }
    },
    "deviation-tolerance": { "verb-in-path": "record" },
    "degradation-policy":  { "auto-restore-fail": "field-mapping" }
  },
  "resources": [
    {
      "name": "interfaces",
      "resource-type": "collection",
      "path-template": "/api/v1/namespaces/{ns}/interfaces",
      "description": "Interface collection",
      "operations": {
        "list": {
          "request": {
            "method": "GET",
            "url-template": "/api/v1/namespaces/{ns}/interfaces"
          },
          "response": {
            "success-codes": "200-299",
            "success-condition": "$.code == 0",
            "error-message-path": "$.message",
            "data-path": "$.data.items",
            "parser": "field-mapping"
          },
          "idempotency": true
        }
      }
    },
    {
      "name": "interface",
      "resource-type": "item",
      "path-template": "/api/v1/namespaces/{ns}/interfaces/{name}",
      "description": "Single interface resource",
      "parent": "interfaces",
      "parent-binding": {
        "path-contains": true,
        "inherit-params": ["ns"],
        "identity-param": "name"
      },
      "fields": [
        {
          "name": "name",
          "type": "string",
          "description": "Interface name, unique within namespace",
          "required": true,
          "constraints": "1..64 characters",
          "examples": ["eth0", "loopback0"]
        },
        {
          "name": "mtu",
          "type": "uint32",
          "description": "Maximum transmission unit in bytes",
          "required": false,
          "constraints": "68..65535",
          "default": 1500,
          "examples": [1500, 9000]
        },
        {
          "name": "vlan-id",
          "type": "uint16",
          "description": "VLAN identifier",
          "required": false,
          "constraints": "1..4094",
          "examples": [100, 200]
        },
        {
          "name": "uuid",
          "type": "uuid",
          "description": "System-assigned unique identifier",
          "read-only": true
        }
      ],
      "operations": {
        "create": {
          "request": {
            "method": "POST",
            "url-template": "/api/v1/namespaces/{ns}/interfaces",
            "headers": { "content-type": "application/json" },
            "body": {
              "type": "template",
              "template-language": "naim-template-v1",
              "template": "{ \"name\": \"{{.name}}\", \"mtu\": {{.mtu}}, \"vlanId\": {{.vlan-id}} }"
            }
          },
          "response": {
            "success-codes": "200-299",
            "success-condition": "$.code == 0",
            "error-message-path": "$.message",
            "data-path": "$.data",
            "parser": "auto",
            "capture": [
              { "name": "created-uuid", "data-path": "$.data.uuid", "required": true }
            ]
          },
          "undo": {
            "description": "Delete the created interface using captured uuid",
            "operation": "delete",
            "params-from-session": { "name": "$session.created-uuid" }
          },
          "preconditions": "Interface name must be unique within the namespace",
          "side-effects": "Allocates network resources; assigns MAC address",
          "idempotency": false,
          "state-transition": { "from": null, "to": "pending" },
          "error-conditions": [
            "Interface name already exists",
            "VLAN ID already in use",
            "Resource quota exceeded"
          ],
          "on-fail": "stop"
        },
        "update": {
          "request": {
            "method": "PUT",
            "url-template": "/api/v1/namespaces/{ns}/interfaces/{name}",
            "headers": { "content-type": "application/json" },
            "body": {
              "type": "template",
              "template-language": "naim-template-v1",
              "template": "{ \"name\": \"{{.name}}\", \"mtu\": {{.mtu}}, \"vlanId\": {{.vlan-id}} }"
            }
          },
          "response": {
            "success-codes": "200-299",
            "success-condition": "$.code == 0",
            "error-message-path": "$.message",
            "data-path": "$.data",
            "parser": "auto"
          },
          "preconditions": "Interface must exist",
          "side-effects": "MTU change may cause brief packet loss",
          "idempotency": true,
          "on-fail": "stop"
        },
        "delete": {
          "request": {
            "method": "DELETE",
            "url-template": "/api/v1/namespaces/{ns}/interfaces/{name}"
          },
          "response": {
            "success-codes": "200-299",
            "success-condition": "$.code == 0",
            "parser": "auto"
          },
          "preconditions": "Interface must be admin-down with no active sessions",
          "side-effects": "Releases all associated network resources",
          "idempotency": "conditional",
          "error-conditions": ["Interface not found", "Interface has active sessions"],
          "on-fail": "stop"
        },
        "get": {
          "request": {
            "method": "GET",
            "url-template": "/api/v1/namespaces/{ns}/interfaces/{name}"
          },
          "response": {
            "success-codes": "200-299",
            "success-condition": "$.code == 0",
            "data-path": "$.data",
            "parser": "field-mapping"
          },
          "idempotency": true
        }
      },
      "field-mapping": [
        { "model-field": "name",    "api-field": "$.name",   "direction": "both", "symmetric": true },
        { "model-field": "mtu",     "api-field": "$.mtu",    "direction": "both", "symmetric": true },
        { "model-field": "vlan-id", "api-field": "$.vlanId", "direction": "both", "symmetric": true,
          "transform": "model uses kebab-case integer; api uses camelCase integer; value is identical" },
        { "model-field": null, "api-field": "$.uuid", "direction": "response-only", "capture-as": "created-uuid" }
      ]
    }
  ]
}
```

### 9.3 Appendix C: Standard Error Codes (Normative)

All Structured Error Reports (see §3.4) MUST use error codes from the following catalog. Implementations MAY define additional codes for domain-specific errors, but MUST prefix them with the implementation namespace (e.g., `acme:RateLimitExceeded`).

| Error Code | Severity | Description |
|---|---|---|
| `SchemaValidationError` | `error` | RAIM Document or Operation IR failed JSON Schema validation |
| `MissingRequiredField` | `error` | A `required: true` field in `fields[]` has no value in Operation IR `params` |
| `TypeMismatchError` | `error` | A `params` value type does not match `fields[].type` |
| `ConstraintViolationError` | `error` | A `params` value violates `fields[].constraints` |
| `ReadOnlyFieldError` | `error` | A `read-only: true` field appears in `params` of a write operation |
| `MappingConflict` | `error` | Multiple `field-mapping` entries reference the same `api-field` with conflicting definitions |
| `MissingCommonFieldGroup` | `error` | A `$ref` target in `common-fields` does not exist |
| `RequiredCaptureFailure` | `error` | A `capture.required: true` field was not found in the response |
| `SessionVariableNotFound` | `error` | A `$session.*` reference has no corresponding `capture[].name` declaration |
| `TemplateSyntaxError` | `error` | Double-brace syntax `{{.varname}}` found in `url-template`, or single-brace in `body.template` |
| `ProfileResolutionError` | `error` | Referenced Convention Profile (`profile-ref: registry:...`) could not be resolved |
| `UnsupportedParserError` | `error` | The execution engine does not support the declared `response.parser` |
| `CompensationNotDefined` | `warn` | An operation has no `undo` definition and no Operation IR `compensation` |
| `CompensationFailed` | `error` | A compensation operation execution failed; see `residual-states` for manual intervention needed |
| `PreconditionFailed` | `error` | Structured precondition `check.assert` evaluated to `false` |
| `PreconditionCheckUnavailable` | `warn` | Structured precondition defined but engine cannot auto-check; fell back to manual confirmation |
| `AsyncTimeoutError` | `error` | Asynchronous operation exceeded `async.max-wait` without completing |
| `AuthRefreshFailed` | `error` | Token refresh request failed or returned an invalid response |
| `DryRunWarning` | `warn` | Operation is in dry-run mode; no actual request was sent |

### 9.4 Appendix D: RAIM Document to AIN Capability Description Language (CDL) Mapping (Informative)

RAIM Documents are candidate inputs to AIN (Agentic Intent Network) capability descriptions. This appendix describes the RECOMMENDED mapping from RAIM Document elements to CDL entries, enabling RAIM-compliant Handlers to announce capabilities in an AIN routing domain.

#### D.1 Mapping Principles

1. **One RAIM Resource = One CDL Capability Entry**: Each resource in `resources[]` maps to a distinct CDL capability.
2. **IC-OID Derivation**: The IC-OID (Intent Class OID) is derived from the API name and resource path.
3. **Operation List**: Each operation on a resource maps to an AIN action descriptor.
4. **Preconditions as Capability Constraints**: Preconditions become input constraints on the CDL entry.

#### D.2 Mapping Table

| RAIM Document Element | CDL Element | Mapping Rule |
|---|---|---|
| `api.name` | `cdl.provider` | The capability provider identifier |
| `api.description` | `cdl.description` | Human-readable capability description |
| `resource.name` | `cdl.capability-name` | The capability name within the provider namespace |
| `resource.resource-type` | `cdl.capability-type` | `collection` → `listable`, `item` → `targetable`, `sub-resource` → `scoped`, `operation-group` → `action-group` |
| `resource.path-template` | `cdl.ic-oid` suffix | Derived: `{api.name}.{resource.path-template sanitized}` |
| `resource.operations.*` | `cdl.actions[]` | Each operation key becomes an action entry |
| `resource.fields[]` | `cdl.input-schema` | Field definitions become the input parameter schema |
| `operations.*.preconditions` | `cdl.constraints[]` | Each precondition becomes a capability constraint |
| `operations.*.side-effects` | `cdl.side-effects` | Side-effect declarations |
| `operations.*.idempotency` | `cdl.idempotency` | Boolean or `conditional` |
| `api.environments[]` | `cdl.endpoints[]` | Each environment maps to an endpoint |
| `convention-profile.conventions.auth` | `cdl.auth-requirements` | Authentication requirements |

#### D.3 Example

Given a RAIM Document with `api.name: "device-mgmt"` and a resource `interface`, a corresponding CDL entry fragment:

```json
{
  "cdl": {
    "provider": "device-mgmt",
    "capability-name": "interface",
    "capability-type": "targetable",
    "ic-oid": "api.device-mgmt.resource.interface",
    "description": "Single interface resource",
    "actions": [
      {
        "name": "get",
        "input-schema": {
          "ns": "string", "name": "string"
        },
        "idempotency": true
      },
      {
        "name": "create",
        "input-schema": {
          "ns": "string", "name": "string", "mtu": "uint32", "vlan-id": "uint16"
        },
        "constraints": ["Interface name must be unique within the namespace"],
        "side-effects": ["Allocates network resources; assigns MAC address"],
        "idempotency": false
      }
    ],
    "endpoints": [
      { "environment": "production", "base-url": "https://192.168.1.1" }
    ]
  }
}
```

> **Note**: This appendix is informative. The CDL format and IC-OID namespace structure are formally defined in the AIN architecture specification. This mapping describes the RECOMMENDED derivation path and is not normative for RAIM Core compliance.

### 9.5 Appendix E: Convention Profile Registry (Informative)

To promote interoperability, common REST API platforms SHOULD have standardized Convention Profiles. This appendix defines a recommended registry structure. Profile authors MAY submit profiles following this format.

#### E.1 Registry Entry Format

```json
{
  "profile-name": "notion-api-v1",
  "base": "pragmatic-rest-v1",
  "platform": "Notion",
  "platform-url": "https://developers.notion.com/reference",
  "version": "1.0.0",
  "conventions": {
    "success": {
      "default-success-codes": "200-299",
      "default-success-condition": "$.object != 'error'",
      "default-error-message-path": "$.message"
    },
    "data": {
      "default-data-path": "$"
    },
    "naming": {
      "model-to-api": "snake_case"
    },
    "pagination": {
      "strategy": "cursor",
      "request": { "cursor-param": "start_cursor", "limit-param": "page_size" },
      "response": { "next-cursor-path": "$.next_cursor", "total-path": null }
    },
    "auth": {
      "type": "bearer",
      "token-header": "Authorization",
      "token-prefix": "Bearer",
      "token-source": "env:NOTION_API_TOKEN"
    }
  },
  "deviation-tolerance": {
    "post-for-read": "record"
  }
}
```

#### E.2 Recommended Registry Profiles

| Profile Name | Platform | Key Convention |
|---|---|---|
| `notion-api-v1` | Notion | `snake_case` naming, `$.object != 'error'` success, cursor pagination |
| `github-api-v1` | GitHub REST API v3 | `camelCase` naming, `$.message` errors, Link-header pagination |
| `slack-web-api-v1` | Slack Web API | `$.ok == true` success, `$.error` error path |
| `jira-cloud-v1` | Jira Cloud REST API | `$.errorMessages[]` errors, offset pagination |
| `stripe-api-v1` | Stripe | `snake_case` naming, no data envelope, idempotency-key header |
| `salesforce-rest-v1` | Salesforce REST API | `$.errors[]` error format, `$.done` for batch results |
| `servicenow-rest-v1` | ServiceNow | `$.result` data envelope, `$.error.message` for errors |
| `pragmatic-rest-v1` | Generic (default) | `$.code == 0` success, `$.data` data path, `$.message` errors |

> **Note**: This registry is informative. Profiles listed here are maintained by the community and do not constitute endorsement. Implementations SHOULD treat profile names as case-sensitive and SHOULD verify profile compatibility before use.

---

## References

- **RAIM-OP Specification** (companion document) — Runtime intent execution specification
- **AIN Architecture Specification** (draft-feng-nmrg-ain-architecture) — AIN capability description and IC-OID namespace
- OpenAPI Specification: https://spec.openapis.org/oas/latest.html
- JSON Schema: https://json-schema.org/
