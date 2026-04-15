# Tingly Guardrails Registry

This repository publishes downloadable guardrails policy fragments for Tingly Box.

It is intentionally narrow:
- one registry index
- one YAML fragment per policy
- no runtime application code

## Example Layout

```text
.
├── README.md
├── index.yaml
└── policies
    ├── p1.yaml
    └── p2.yaml