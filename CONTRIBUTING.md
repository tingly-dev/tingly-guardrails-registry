# Contributing

This repository publishes downloadable guardrails policy fragments for Tingly Box.

Its scope is intentionally narrow:
- one registry index
- one YAML fragment per policy
- no runtime application code
- no global guardrails config

## Repository Layout

```text
.
├── README.md
├── CONTRIBUTING.md
├── index.yaml
└── policies
    ├── block-env-read.yaml
    ├── block-private-key-output.yaml
    └── ...
```

## What Belongs Here

This repository should only contain:

- `index.yaml`
- policy fragment files under `policies/`

This repository should not contain:

- top-level `guardrails.yaml`
- group definitions
- app code
- server code
- frontend code
- generated runtime state

## Registry Format

`index.yaml` must stay minimal.

Use this format:

```yaml
policies:
  - id: block-env-read
    path: policies/block-env-read.yaml
  - id: block-private-key-output
    path: policies/block-private-key-output.yaml
```

Rules:

- each entry must have `id`
- each entry must have `path`
- `path` must point to a file inside `policies/`
- `id` must be globally unique across the repository
- keep entries sorted by `id`

Do not duplicate display metadata in `index.yaml`.

Do not add:
- `name`
- `summary`
- `description`
- `topic`
- `tags`
- `version`

That metadata should come from the fragment file itself.

## Policy Fragment Format

Each file under `policies/` must be a standard imported policy fragment.

Allowed top-level fields:

- `policies`

Not allowed top-level fields:

- `groups`
- `strategy`
- `error_strategy`
- `imports`
- `templates`

A valid fragment looks like this:

```yaml
policies:
  - id: block-env-read
    name: Block .env Reads
    groups: [default]
    enabled: false
    kind: resource_access
    verdict: block
    reason: This policy blocks attempts to read environment variable files that may contain secrets.
    match:
      tool_names: [bash]
      actions:
        include: [read]
      resources:
        type: path
        mode: contains
        values: [".env", ".env.local", ".env.production"]
```

## Required Policy Fields

Each policy must define:

- `id`
- `name`
- `groups`
- `enabled`
- `kind`
- `verdict`
- `reason`
- `match`

`scope` is optional.

## Policy Conventions

Use these conventions for all policies in this repository:

- `groups` should usually be `[default]`
- `enabled` should default to `false`
- `verdict` should usually be `block`
- keep `reason` short and user-facing
- keep policy ids stable once published
- prefer one policy per file unless multiple rules are tightly coupled

## Supported Kinds

Common policy kinds:

- `resource_access`
- `command_execution`
- `content`

Choose the narrowest kind that matches the policy behavior.

## Style Guidance

Prefer concise, explicit policy definitions.

Recommended:
- short stable `id`
- clear `name`
- specific `reason`
- narrow matching conditions
- predictable file names

Avoid:
- UI-only metadata
- marketing copy
- broad catch-all rules without clear intent
- unrelated multiple policies in one file unless necessary

## Adding a New Policy

To add a new policy:

1. Create a new fragment file under `policies/`
   Example:
   `policies/block-env-read.yaml`

2. Add a standard fragment:
   ```yaml
   policies:
     - id: block-env-read
       name: Block .env Reads
       groups: [default]
       enabled: false
       kind: resource_access
       verdict: block
       reason: This policy blocks attempts to read environment variable files that may contain secrets.
       match:
         tool_names: [bash]
         actions:
           include: [read]
         resources:
           type: path
           mode: contains
           values: [".env", ".env.local", ".env.production"]
   ```

3. Add an entry to `index.yaml`
   ```yaml
   policies:
     - id: block-env-read
       path: policies/block-env-read.yaml
   ```

4. Keep `index.yaml` sorted by `id`

5. Open a pull request

## Updating an Existing Policy

When updating an existing policy:

- keep the same `id`
- update the existing fragment file in place
- do not create a new file just for a small behavior change
- only rename the file if the old name is clearly wrong

If the semantic intent changes substantially, prefer a new policy id.

## Review Checklist

Before opening a PR, verify:

- the fragment has only top-level `policies`
- every policy has a unique `id`
- `index.yaml` contains the matching `id` and `path`
- `index.yaml` is sorted by `id`
- the file name matches the policy intent
- `enabled` is `false`
- `groups` is set intentionally
- `reason` is clear and concise

## Example Patterns

### Resource Access

```yaml
policies:
  - id: block-private-key-file-read
    name: Block private key file reads
    groups: [default]
    enabled: false
    kind: resource_access
    verdict: block
    reason: This policy blocks attempts to read private key files.
    match:
      tool_names: [bash]
      actions:
        include: [read]
      resources:
        type: path
        mode: contains
        values: [".pem", ".key", "id_rsa", "id_ed25519"]
```

### Command Execution

```yaml
policies:
  - id: block-destructive-rm
    name: Block destructive rm
    groups: [default]
    enabled: false
    kind: command_execution
    verdict: block
    reason: This policy blocks destructive file removal commands.
    match:
      tool_names: [bash]
      actions:
        include: [write]
      terms: ["rm -rf", "rm -fr"]
```

### Content

```yaml
policies:
  - id: block-private-key-output
    name: Block private key output
    groups: [default]
    enabled: false
    kind: content
    verdict: block
    reason: This policy blocks private key material before it is returned.
    match:
      patterns:
        - "BEGIN OPENSSH PRIVATE KEY"
        - "BEGIN RSA PRIVATE KEY"
      pattern_mode: substring
      case_sensitive: false
```

## Pull Requests

PRs should be small and focused.

A good PR usually includes:
- one new policy
- or one clear update to an existing policy
- plus the corresponding `index.yaml` change

Avoid mixing unrelated policy additions in one PR unless they are part of the same set.