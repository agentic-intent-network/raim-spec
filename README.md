# RAIM Specification

**RAIM** (REST AI Interface Modeling) is a semantic specification framework
for AI Agents calling REST APIs reliably. It defines a structured semantic
intermediate representation (RAIM Document) and a deterministic
validation-execution pipeline that bridges AI-generated operation intent
and concrete API execution.

> **Status**: v1.0 specification draft (2026-04-30). Field-level details
> and examples are subject to revision based on community feedback.

---

## What RAIM Solves

When AI Agents call external REST APIs, two common architectures fall short:

- **Direct HTTP generation by AI**: relies on natural language API docs;
  no deterministic semantic validation; field mapping errors are silent.
- **MCP (Model Context Protocol) tool wrapping**: solves discovery but
  not semantics — preconditions, side effects, field mappings, compensation
  logic are still missing.

RAIM addresses both by providing:

1. A canonical JSON document (**RAIM Document**) that captures field
   semantics, directional/asymmetric field mappings, preconditions,
   business success conditions, and compensation logic;
2. A structured operation intermediate representation (**Operation IR**)
   for AI to express intent without protocol-layer concerns;
3. A deterministic execution pipeline that performs validation, field
   mapping, request rendering, business success determination, and
   compensation triggering — without giving AI direct API execution power.

---

## Documents

- [`docs/raim-core-spec-v1.0.md`](docs/raim-core-spec-v1.0.md) — RAIM Core
  specification: document format, semantic field definitions, convention
  profiles, deviation tolerance, degradation policies.
- [`docs/raim-op-spec-v1.0.md`](docs/raim-op-spec-v1.0.md) — RAIM-OP
  runtime execution specification: Operation IR, execution pipeline,
  asynchronous handling, compensation strategies.

---

## License

This specification is licensed under the **Apache License, Version 2.0**
— see [`LICENSE`](LICENSE) and [`NOTICE`](NOTICE).

The Apache 2.0 license includes an explicit patent grant: any party that
distributes the Work (or a derivative) automatically receives a perpetual,
worldwide, royalty-free patent license from each contributor. The grant
terminates if the recipient initiates patent litigation alleging that the
Work or a contribution constitutes direct or contributory patent
infringement (Apache 2.0 §3).

---

## Patent Non-Assertion Pledge

The author of this specification will not assert any patents (currently
held or hereafter obtained) against any party that:

1. Implements, uses, distributes, or contributes to the RAIM
   specification in good faith; and
2. Has not initiated patent litigation against this project, its
   maintainers, or its users.

This pledge is irrevocable for any party complying with the above
conditions. The intent is to establish RAIM as an open ecosystem
foundation, not a vehicle for licensing revenue.

---

## Citing RAIM

If you reference RAIM in academic work, technical reports, or other
specifications, please cite:

```
Feng, C. RAIM Specification v1.0: REST API Interface Modeling for
AI Agents. 2026. https://github.com/agentic-intent-network/raim-spec
```

---

## Contributing

Issues and pull requests are welcome. By contributing, you agree that
your contributions will be licensed under the Apache License 2.0.

---

## Author

Chong Feng

Related work in the Agentic Intent Network (AIN) ecosystem:

- IETF NMRG: `draft-feng-nmrg-ain-architecture-00`
- IETF NMRG: `draft-feng-nmrg-ain-deployment-00`
- IETF NETMOD: `draft-feng-netmod-naim-00`
