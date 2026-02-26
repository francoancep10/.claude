---
name: add-config-value
description: Add or update a configuration value in a .NET project that uses the AWS SSM → External Secrets Operator → Pod env vars → appsettings.json pattern. Handles all places a config value lives: appsettings.json (tokenized reference), appsettings.Development.json (real dev value), all discovered Helm values files (externalSecrets entries), and the AWS SSM Parameter Store JSON objects for each environment. Use this skill whenever someone mentions adding a new config key, adding a new environment variable, wiring up a new setting, adding a new appsettings entry, or updating a value in SSM. Also trigger when the user says things like "add a new config", "add a setting", "add an env var", "update the SSM value", or "configure a new feature flag".
---

# Add / Update Config Value

This skill handles the full workflow for adding or updating a configuration value across all the places it needs to exist in a .NET project using the standard SSM → External Secrets → env vars → appsettings.json pipeline.

## How config works in this project

Values flow like this:

```
AWS SSM Parameter Store  →  External Secrets Operator  →  Pod env vars  →  appsettings.json tokens  →  .NET app
```

Concretely:
- **`appsettings.json`** holds tokenized references: `"KeyName": "${ENV_VAR_NAME}"`
- **`appsettings.Development.json`** holds real values for local dev
- **`values-*.yaml`** files each have an `externalSecrets` list mapping env var names to SSM properties
- **AWS SSM Parameter Store** stores a single JSON object per environment, one path per Helm file

---

## Discovery Phase

Run these steps before collecting any values from the user.

### D1 — Find appsettings files

Glob `**/appsettings.json` to find all appsettings files. Resolve:
- `{APPSETTINGS_FILE}` — the main appsettings.json
- `{APPSETTINGS_DEV_FILE}` — the corresponding appsettings.Development.json (glob `**/appsettings.Development.json`)

If multiple `appsettings.json` files are found (e.g. in a multi-project repo), ask the user which project to target before continuing.

### D2 — Find Helm files and extract SSM paths

Glob `**/values-*.yaml`. For each discovered file:
1. Read its `externalSecrets` block
2. Extract the `key:` value from the `remoteRef` of the first entry — that is `{SSM_PATH_N}` for that file
3. Build a discovery table:

| Variable | Resolved value |
|----------|---------------|
| `{HELM_FILE_1}` | `automation/helm/config/values-main.yaml` |
| `{SSM_PATH_1}` | `/prd/my-project` |
| `{HELM_FILE_2}` | `automation/helm/config/values-feature-branch.yaml` |
| `{SSM_PATH_2}` | `/feature/my-project` |

If a values file has no `externalSecrets` block, warn the user and skip that file.

### D3 — Confirm discovered paths

Show the user the full discovery table and ask them to confirm before proceeding. Example:

> I found the following files and SSM paths:
>
> | Helm file | SSM path |
> |-----------|----------|
> | `{HELM_FILE_1}` | `{SSM_PATH_1}` |
> | `{HELM_FILE_2}` | `{SSM_PATH_2}` |
>
> Does this look correct?

### D4 — Update-only mode check

After discovery, check whether the key already exists:
- If the key already exists in **all** files → skip Steps 1–3, go straight to Step 4 (auth) + Step 5 (SSM)
- If the key exists in **some** files but not others → skip only the steps for files that already have it
- If the key is new everywhere → proceed with all steps

---

## Gather what you need

Before making changes, collect the following. Do **not** ask for all items upfront — gather them interactively as you reach each step.

| Item | Description |
|------|-------------|
| `{SECTION}` | Section name in appsettings (e.g. `Hangfire`, `MonitorLogs`); empty string if root-level |
| `{KEY}` | The appsettings key name (PascalCase, e.g. `EnabledCL`) |
| `{ENV_VAR_NAME}` | Inferred from `{SECTION}` + `{KEY}` — see inference rule below |
| `{DEV_VALUE}` | Real value for local development |
| `{VALUE_N}` | Value for each discovered environment (one per `{SSM_PATH_N}`) |

### Env var name inference rule

Join `{SECTION}` and `{KEY}` with `_` (omit the prefix if `{SECTION}` is root-level), then apply:
1. Insert `_` before each uppercase letter that follows a lowercase letter: `([a-z])([A-Z])` → `$1_$2`
2. Uppercase everything

Examples:
- `SincroAndrea` + `EnabledCL` → `SINCRO_ANDREA_ENABLED_CL`
- `Hangfire` + `RetencionResultadosEnDias` → `HANGFIRE_RETENCION_RESULTADOS_EN_DIAS`
- Root-level `FeatureFlagX` → `FEATURE_FLAG_X`

**Always show the inferred name to the user and allow them to override it before continuing.**

### Per-environment values

After resolving `{ENV_VAR_NAME}`, explicitly ask the user for the value in each discovered environment. Present the list derived from the Helm discovery table:

> What value should `{ENV_VAR_NAME}` have in each environment?
>
> - Production (`{SSM_PATH_1}`):
> - Staging (`{SSM_PATH_2}`):

Do **not** assume the values are the same across environments. Prompt for each individually.

---

## Step 1: appsettings.json

File: `{APPSETTINGS_FILE}`

Add the new key with a tokenized reference. Always use the exact format `"${ENV_VAR_NAME}"`. Place it in the correct `{SECTION}` block, grouped with related keys.

```json
"{SECTION}": {
  "ExistingKey": "${EXISTING_ENV_VAR}",
  "{KEY}": "${ENV_VAR_NAME}"   // ← new
}
```

For a root-level key (no section), add it at the top level of the JSON object.

## Step 2: appsettings.Development.json

File: `{APPSETTINGS_DEV_FILE}`

Add the same key in the same `{SECTION}` with the **real dev value** (`{DEV_VALUE}`), not a token. The structure must mirror appsettings.json.

```json
"{SECTION}": {
  "ExistingKey": "existing-dev-value",
  "{KEY}": "{DEV_VALUE}"   // ← new
}
```

## Step 3: Helm values files

For each discovered `{HELM_FILE_N}`, append a new entry to its `externalSecrets` list using the corresponding `{SSM_PATH_N}`:

```yaml
- secretKey: {ENV_VAR_NAME}
  remoteRef:
    key: {SSM_PATH_N}
    property: {ENV_VAR_NAME}
```

Place the new entry near related entries (e.g. group with the same section's other keys). Repeat for every Helm file in the discovery table.

---

## Step 4: AWS auth check

Before any SSM operation, verify authentication:

```bash
aws sts get-caller-identity
```

If this fails:
1. Automatically run `aws sso login` — do not just tell the user to do it
2. After `aws sso login` completes, re-run `aws sts get-caller-identity` to confirm auth succeeded
3. Only proceed to Step 5 once the identity check passes

---

## Step 5: AWS SSM Parameter Store

### Confirmation gate

Before executing any write, show the user a confirmation table:

```
Environment   SSM path        Key               Value
{ENV_NAME_1}  {SSM_PATH_1}    {ENV_VAR_NAME}    {VALUE_1}
{ENV_NAME_2}  {SSM_PATH_2}    {ENV_VAR_NAME}    {VALUE_2}
```

Ask: **"Ready to write these values to SSM? (yes/no)"**

Only proceed after the user confirms.

### Write commands

Run once per `{SSM_PATH_N}`, substituting the correct `{VALUE_N}` for each environment:

```bash
PARAM_NAME="{SSM_PATH_N}"
CURRENT=$(MSYS_NO_PATHCONV=1 aws ssm get-parameter \
  --name "$PARAM_NAME" \
  --with-decryption \
  --query Parameter.Value \
  --output text)

UPDATED=$(echo "$CURRENT" | python3 -c "
import sys, json
d = json.load(sys.stdin)
d['{ENV_VAR_NAME}'] = '{VALUE_N}'
print(json.dumps(d))
")

MSYS_NO_PATHCONV=1 aws ssm put-parameter \
  --name "$PARAM_NAME" \
  --value "$UPDATED" \
  --type SecureString \
  --overwrite
```

---

## Step 6: Summary

After finishing, report clearly:

- Which files were changed and what was added/updated in each:
  - `{APPSETTINGS_FILE}` — added `{SECTION}.{KEY}` = `"${ENV_VAR_NAME}"`
  - `{APPSETTINGS_DEV_FILE}` — added `{SECTION}.{KEY}` = `{DEV_VALUE}`
  - Each `{HELM_FILE_N}` — added `externalSecrets` entry for `{ENV_VAR_NAME}` → `{SSM_PATH_N}`
- What was written to AWS SSM (key, value per environment)
- Any files that were **skipped** and why (e.g. key already existed, no `externalSecrets` block found)
- Any next steps the user should take (e.g. redeploy for the changes to take effect in the running cluster)
