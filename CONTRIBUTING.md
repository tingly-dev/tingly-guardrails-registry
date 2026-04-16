# Contributing Guardrails Policies

This repository stores Guardrails policy fragments.

Each policy file must use the current Guardrails fragment format so it can be:

- embedded as a builtin example,
- listed in the remote registry,
- imported locally into `guardrails/custom/import.yaml`,
- evaluated without per-file migration logic.

This guide focuses on policy format and contribution requirements.

## 1. Repository Layout

Use this structure:

```text
index.yaml
policies/
  my_policy.yaml
  another_policy.yaml
```

Rules:

- `index.yaml` is the registry manifest.
- `policies/*.yaml` contains policy fragments only.
- A fragment file may contain one policy or multiple policies.
- Fragment files must not declare top-level `groups`, `imports`, `strategy`, or `error_strategy`.

## 2. Fragment File Format

Every policy file must be a Guardrails fragment with this top-level shape:

```yaml
policies:
  - id: example-policy-id
    name: Example policy
    groups: [default]
    enabled: false
    kind: resource_access
    verdict: block
    reason: Short explanation shown in the UI.
    match:
      ...
```

Important:

- Top-level key must be `policies`.
- `policies` must be a YAML list.
- Each policy in the file must have a unique `id`.
- `enabled` should always be written explicitly.
  Omitted `enabled` is treated as `false`.
- `groups` should usually be `[default]`.
- Imported fragments cannot define groups themselves.
  They must reference groups that already exist in the root config.

## 3. Fields Allowed In Imported Policy Fragments

Allowed top-level fragment fields:

- `policies`

Not allowed in fragment files:

- `imports`
- `groups`
- `strategy`
- `error_strategy`

If any of those fields are present, fragment import/loading will fail.

## 4. Policy Fields

Each policy uses this structure:

```yaml
- id: my-policy-id
  name: Human readable name
  groups: [default]
  enabled: false
  kind: resource_access | command_execution | content
  verdict: allow | review | block
  reason: Why this policy exists
  scope:
    scenarios: [...]
    models: [...]
    directions: [request, response]
    tags: [...]
  match:
    ...
```

### Required fields

These fields should always be present:

- `id`
- `groups`
- `enabled`
- `kind`
- `match`

These fields are strongly recommended:

- `name`
- `verdict`
- `reason`

### `id`

Requirements:

- Must be unique across all installed policies.
- Use lowercase kebab-case.
- Make it stable and descriptive.
- Prefix with the behavior being blocked or reviewed.

Good examples:

- `block-ssh-read`
- `block-request-secret-egress`
- `review-external-post`
- `block-malicious-npm-packages`

### `groups`

Requirements:

- Must be a list of existing group IDs.
- Usually use `[default]`.
- Do not leave this empty.
- Do not invent group IDs that are not guaranteed to exist in the consuming config.

Recommended default:

```yaml
groups: [default]
```

### `enabled`

Always write it explicitly:

```yaml
enabled: false
```

Use `false` for builtin and registry policies by default.
Policies should be opt-in unless there is a very strong reason otherwise.

### `kind`

Supported values:

- `resource_access`
- `command_execution`
- `content`

Do not use `operation` in new contributions.
That value exists only as a legacy compatibility kind.

### `verdict`

Supported values in policy config:

- `allow`
- `review`
- `block`

Use `block` by default unless the policy is intentionally advisory.

### `reason`

`reason` should be:

- one sentence,
- user-facing,
- specific about what is being blocked or reviewed.

Good:

```yaml
reason: This policy blocks attempts to read SSH configuration and key directories.
```

Avoid vague reasons like:

```yaml
reason: Dangerous
```

## 5. `scope` Format

`scope` is optional. Use it only when the policy should apply to a narrower subset of traffic.

Example:

```yaml
scope:
  directions: [request]
```

Supported fields:

- `scenarios`
- `models`
- `directions`
- `tags`

### `directions`

Supported values:

- `request`
- `response`

Use `directions` when the policy should only run on request content or only on response content.

Example:

```yaml
scope:
  directions: [request]
```

## 6. `match` Format By Policy Kind

The exact `match` fields depend on `kind`.

### A. `resource_access`

Use this kind for file/path/resource access controls such as read, write, delete, or network access.

Recommended fields:

- `actions.include`
- `resources`
- optional `tool_names`

Example:

```yaml
policies:
  - id: block-ssh-read
    name: Block SSH directory reads
    groups: [default]
    enabled: false
    kind: resource_access
    verdict: block
    reason: This policy blocks attempts to read SSH configuration and key directories.
    match:
      actions:
        include: [read]
      resources:
        type: path
        mode: prefix
        values: ["~/.ssh", "/etc/ssh"]
```

#### `actions.include` for `resource_access`

Recommended values:

- `read`
- `write`
- `delete`
- `network`

Example:

```yaml
actions:
  include: [read]
```

#### `resources`

Format:

```yaml
resources:
  type: path
  mode: prefix | contains | exact
  values:
    - ~/.ssh
    - /etc/ssh
```

Notes:

- Use `resources` for the protected target set.
- `mode` should be one of `prefix`, `contains`, or `exact`.
- `values` should be a non-empty list for most resource access policies.
- `type` is currently a forward-compatible field. Use `path` in contributions today.
- Although the schema exposes `type`, current runtime matching does not yet apply different semantics for different resource types.

#### `tool_names`

`tool_names` is optional for `resource_access`.

If omitted, the runtime uses a built-in compatibility set of shell-oriented tools. Today that default set starts with `bash` and may expand in code over time.

Use `tool_names` only when the policy must explicitly target a narrower tool set.

### B. `command_execution`

Use this kind for command intent rules.

This kind remains intentionally flexible. In current runtime behavior:

- `terms` is the primary matcher for command patterns and install targets
- `actions.include: [install]` can be used to narrow a rule to install-like commands
- `resources` may still be used as an additional filter when needed
- `tool_names` is optional and should usually be omitted for `command_execution`

#### Execute-style command rules

Use `terms` for explicit command patterns.

Example:

```yaml
policies:
  - id: block-destructive-rm
    name: Block destructive rm commands
    groups: [default]
    enabled: false
    kind: command_execution
    verdict: block
    reason: This policy blocks destructive recursive deletion commands.
    match:
      terms: ["rm -rf", "rm -fr"]
      resources:
        type: path
        mode: contains
        values: ["/", "~", "."]
```

Notes:

- If `actions` is omitted, command execution currently defaults to execute-style behavior.
- `terms` should contain concrete command patterns, not broad generic words.
- `resources` is optional and acts as an additional filter, not the primary command matcher.
- Add `tool_names` only when the rule must explicitly target a narrower tool set.

#### Install-style command rules

Use this kind when matching normalized install behavior such as:

- `npm install`
- `pip install`
- `cargo install`
- `uv tool install`
- `code --install-extension`

Use:

```yaml
actions:
  include: [install]
```

Example:

```yaml
policies:
  - id: block-malicious-npm-packages
    name: Block known malicious npm packages
    groups: [default]
    enabled: false
    kind: command_execution
    verdict: block
    reason: This policy blocks package names already reported as malicious.
    match:
      actions:
        include: [install]
      terms:
        - left-pad
        - requests
        - ripgrep
        - ms-python.python
```

Notes:

- Install-style policies should usually express the blocked target in `terms`.
- Do not assume install targets will appear in `resources`.
- Today, many install targets such as package names and extension IDs are normalized into `terms`, not `resources`.
- `resources` may still be added as an optional extra filter when you intentionally want to narrow the rule to path-like or URL-like targets.
- Do not put `install` under `resource_access`.

### C. `content`

Use this kind for request/response text matching, secret egress filtering, or output blocking.

A content policy must include at least one of:

- `patterns`
- `credential_refs`

Example:

```yaml
policies:
  - id: block-request-secret-egress
    name: Block request secret egress
    groups: [default]
    enabled: false
    kind: content
    verdict: block
    reason: This policy blocks request content that appears to contain high-risk secrets.
    scope:
      directions: [request]
    match:
      patterns:
        - "sk-[A-Za-z0-9]{20,}"
        - "AKIA[0-9A-Z]{16}"
      pattern_mode: regex
      case_sensitive: true
```

#### `patterns`

A list of substrings or regexes.

#### `pattern_mode`

Supported values:

- `substring`
- `regex`

#### `case_sensitive`

Boolean.

#### `min_matches`

Optional.
Useful when one pattern alone is too noisy and you want multiple hits before blocking.

Example:

```yaml
match:
  patterns:
    - OPENAI_API_KEY=
    - ANTHROPIC_API_KEY=
    - DATABASE_URL=
  pattern_mode: substring
  case_sensitive: true
  min_matches: 2
```

## 7. Registry `index.yaml` Format

`index.yaml` is the download manifest.

Current format:

```yaml
policies:
  - id: block-malicious-npm-packages
    name: Block known malicious npm packages
    reason: This policy blocks package names already reported as malicious in the OpenSSF Malicious Packages dataset for npm.
    path: policies/malicious_npm_packages.yaml
```

Fields:

- `id`
- `name`
- `reason`
- `path`

Rules:

- `id` must match a policy ID contained in the target fragment file.
- `path` is relative to the repository root.
- Multiple `index.yaml` entries may point to the same file if that file contains multiple policies.
- Each listed `id` must exist in the file referenced by `path`.

## 8. Examples

### Resource access example

```yaml
policies:
  - id: block-env-read
    name: Block .env file reads
    groups: [default]
    enabled: false
    kind: resource_access
    verdict: block
    reason: This policy blocks attempts to read environment variable files that may contain secrets.
    match:
      actions:
        include: [read]
      resources:
        type: path
        mode: contains
        values: [".env", ".env.local", ".env.production"]
```

### Command execution example

```yaml
policies:
  - id: block-download-and-exec
    name: Block download-and-exec patterns
    groups: [default]
    enabled: false
    kind: command_execution
    verdict: block
    reason: This policy blocks download-and-execute command patterns.
    match:
      terms: ["curl | sh", "curl|sh", "wget | bash", "wget|bash"]
```

### Content example

```yaml
policies:
  - id: block-private-key-output
    name: Block private key output
    groups: [default]
    enabled: false
    kind: content
    verdict: block
    reason: This policy blocks private key material from being returned.
    match:
      patterns:
        - "BEGIN OPENSSH PRIVATE KEY"
        - "BEGIN RSA PRIVATE KEY"
      pattern_mode: substring
      case_sensitive: true
```

## 9. Common Mistakes

Do not do these:

### Adding root config fields to fragments

Invalid:

```yaml
groups:
  - id: default
    enabled: true
policies:
  - ...
```

Fragments must not declare `groups`.

### Omitting `enabled`

Avoid:

```yaml
policies:
  - id: my-policy
    kind: content
    match:
      patterns: ["secret"]
```

Write this instead:

```yaml
policies:
  - id: my-policy
    groups: [default]
    enabled: false
    kind: content
    match:
      patterns: ["secret"]
```

### Using unknown groups

Avoid:

```yaml
groups: [security-team]
```

unless that group is guaranteed to exist in the consuming root config.

### Using empty content policies

Invalid:

```yaml
kind: content
match: {}
```

Content policies must include `patterns` or `credential_refs`.

### Using legacy `operation` for new policies

Do not add new policies with:

```yaml
kind: operation
```

Use:

```yaml
kind: resource_access
```

### Requiring `tool_names: [bash]` in every `resource_access` policy

Avoid treating this as mandatory in contributed policy fragments.

Current runtime behavior already provides a default compatibility set for `resource_access` when `tool_names` is omitted. Add `tool_names` only when the policy must explicitly target a narrower tool set.

### Using `resources` as the primary install target matcher

Avoid this for install blocklists.

Example to avoid:

```yaml
match:
  actions:
    include: [install]
  resources:
    type: path
    mode: contains
    values: ["left-pad"]
```

Prefer:

```yaml
match:
  actions:
    include: [install]
  terms: ["left-pad"]
```

Many install targets are currently normalized into `terms`, not `resources`.

## 10. Contribution Checklist

Before submitting a policy:

- The file is under `policies/`.
- The top-level shape is `policies:`.
- The fragment does not declare `groups`, `imports`, `strategy`, or `error_strategy`.
- Every policy has a unique `id`.
- Every policy sets `groups`.
- Every policy sets `enabled` explicitly.
- Every policy sets `kind`.
- Every policy has a meaningful `reason`.
- `content` policies include `patterns` or `credential_refs`.
- Any `index.yaml` entry points to a file that actually contains the referenced `id`.

## 11. Practical Guidance

Prefer specific policies over broad ones.

Good policy characteristics:

- narrow scope,
- clear intent,
- obvious reason,
- low false-positive risk,
- stable IDs.

When in doubt:

- keep `enabled: false`,
- use `groups: [default]`,
- use `verdict: block`,
- keep fragment files free of root config concerns.
