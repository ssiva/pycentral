# New Central On-Prem Token Endpoint Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan. The orchestrator must assign the implementation to one `gpt-5.6-terra` subagent and the completed diff review to one separate `gpt-5.6-luna` subagent. Do not use additional implementation or review agents.

**Goal:** Restrict the custom `token_endpoint` override to New Central On-Prem credentials while preserving all existing Cloud, GLP, and unified token-generation behavior.

**Architecture:** Keep endpoint selection in `new_parse_input_args()`, where token configuration is already validated and the internal `_token_url` is derived. `new_central` may override the default issuer with `token_endpoint`; `glp` and `unified` reject that field before any Cloud fallback can occur. No `on_prem` flag, URL derivation, helper abstraction, or new test framework is needed.

**Tech Stack:** Python 3.10+, `urllib.parse` through the existing `valid_url()`, Markdown, Git.

## Global Constraints

- The orchestrator assigns all implementation steps to one `gpt-5.6-terra` subagent.
- After implementation and validation, the orchestrator assigns one full-diff review to a separate `gpt-5.6-luna` subagent.
- `token_endpoint` is supported only inside `token_info["new_central"]`.
- `token_endpoint` under `glp` or `unified` raises `ValueError`; it must never be ignored or silently routed to a Cloud issuer.
- Omitting `token_endpoint` preserves current Cloud behavior exactly.
- `new_central` and standalone `glp` continue to default to `AUTHENTICATION["OAUTH"]`.
- `unified` continues to use `AUTHENTICATION["OAUTH_GLOBAL"]/{workspace_id}/token`.
- The token issuer is independent of `base_url`; never derive one from the other.
- Preserve the existing `valid_url()` behavior, including support for HTTP endpoints on private On-Prem networks.
- Do not add `on_prem`, a URL value type, a token URL helper, dependencies, or a test framework.
- Do not change MSP token exchange behavior.
- Do not claim broad Central On-Prem SDK support; document only the New Central token issuer override.
- The user accepted no new automated tests for this experimental change.

---

### Task 1: Restrict the token endpoint override to New Central

**Assigned agent:** One `gpt-5.6-terra` implementation subagent, coordinated by the orchestrator.

**Files:**
- Modify: `pycentral/utils/base_utils.py:20-35`
- Modify: `pycentral/utils/base_utils.py:61-71`
- Modify: `pycentral/utils/base_utils.py:94-129`

**Interfaces:**
- Consumes: `new_parse_input_args(token_info: dict | str) -> dict`, `valid_url(url: str) -> str`, and `AUTHENTICATION`.
- Produces: `token_info["new_central"]["_token_url"]` set to the validated override or the existing Cloud default.
- Preserves: standalone `glp` and `unified` `_token_url` derivation.

- [ ] **Step 1: Remove the public field from unified defaults**

Delete `"token_endpoint": None` from `UNIFIED_DEFAULT_ARGS`. Keep it in `NEW_CENTRAL_C_DEFAULT_ARGS`, because that defaults dictionary is shared by standalone app entries and the parser will explicitly reject the field for `glp`.

```python
UNIFIED_DEFAULT_ARGS = {
    "client_id": None,
    "client_secret": None,
    "workspace_id": None,
    "glp_base_url": None,
    "base_url": None,
    "access_token": None,
}
```

- [ ] **Step 2: Reject the field in unified mode**

Immediately after copying the unified input, reject `token_endpoint` before resolving any Cloud URLs:

```python
unified = dict(token_info["unified"])
if "token_endpoint" in unified:
    raise ValueError(
        "'token_endpoint' is supported only for 'new_central' "
        "Central On-Prem credentials."
    )
```

Use key presence rather than truthiness so an empty or `None` value cannot be silently ignored and routed to the Cloud issuer.

- [ ] **Step 3: Restore unchanged unified token URL selection**

Remove the unified custom-endpoint branch and retain only the workspace-scoped Cloud URL:

```python
unified["_token_url"] = (
    f"{AUTHENTICATION['OAUTH_GLOBAL']}/{unified['workspace_id']}/token"
)
```

`_validate_token_creation_keys("unified", unified)` already requires `workspace_id`, so no conditional or fallback is needed.

- [ ] **Step 4: Reject the field for standalone GLP**

In the non-unified app loop, after validating the app name and before resolving URLs, reject key presence for every app except `new_central`:

```python
if app != "new_central" and "token_endpoint" in app_token_info:
    raise ValueError(
        "'token_endpoint' is supported only for 'new_central' "
        "Central On-Prem credentials."
    )
```

- [ ] **Step 5: Limit override selection to New Central**

Replace the generic custom endpoint branch with explicit `new_central` handling while preserving the existing default for both standalone modes:

```python
if app == "new_central" and "token_endpoint" in app_token_info:
    app_token_info["_token_url"] = valid_url(app_token_info["token_endpoint"])
else:
    app_token_info["_token_url"] = AUTHENTICATION["OAUTH"]
```

Do not derive the endpoint from `base_url`. Do not add scheme restrictions beyond `valid_url()`.

- [ ] **Step 6: Narrow the parser documentation**

Replace the generic docstring claim with:

```python
An optional `token_endpoint` key can be provided under `new_central`
to override the default OAuth token endpoint used for token creation
and refresh in Central On-Prem deployments.
```

- [ ] **Step 7: Run focused validation**

Run:

```bash
python3 -m py_compile pycentral/utils/base_utils.py
git diff --check
```

Expected: both commands exit with status 0.

If project dependencies are already installed, also run this parser-level behavior check:

```bash
python3 - <<'PY'
from pycentral.utils.base_utils import new_parse_input_args
from pycentral.utils.constants import AUTHENTICATION

cloud = new_parse_input_args({
    "new_central": {
        "client_id": "id",
        "client_secret": "secret",
        "base_url": "https://central.example",
    }
})
assert cloud["new_central"]["_token_url"] == AUTHENTICATION["OAUTH"]

on_prem = new_parse_input_args({
    "new_central": {
        "client_id": "id",
        "client_secret": "secret",
        "base_url": "https://central.internal",
        "token_endpoint": "http://issuer.internal/oauth2/token",
    }
})
assert on_prem["new_central"]["_token_url"] == (
    "http://issuer.internal/oauth2/token"
)

for app, credentials in (
    ("glp", {
        "client_id": "id",
        "client_secret": "secret",
        "token_endpoint": "http://issuer.internal/oauth2/token",
    }),
    ("unified", {
        "client_id": "id",
        "client_secret": "secret",
        "workspace_id": "workspace",
        "token_endpoint": "http://issuer.internal/oauth2/token",
    }),
):
    try:
        new_parse_input_args({app: credentials})
    except ValueError as error:
        assert "supported only for 'new_central'" in str(error)
    else:
        raise AssertionError(f"{app} accepted token_endpoint")
PY
```

Expected: exit status 0. If imports fail solely because declared dependencies are not installed, record that limitation; do not install dependencies or replace this check with a new test framework.

- [ ] **Step 8: Commit the parser change**

```bash
git add pycentral/utils/base_utils.py
git commit -m "fix: scope token endpoint to new central"
```

### Task 2: Narrow authentication documentation to New Central On-Prem

**Assigned agent:** The same `gpt-5.6-terra` implementation subagent.

**Files:**
- Modify: `docs/getting-started/authentication.md:30-117`

**Interfaces:**
- Consumes: the public `new_central.token_endpoint` behavior from Task 1.
- Produces: documentation that does not advertise the field for unified or GLP credentials.

- [ ] **Step 1: Remove unified support claims**

Delete the `token_endpoint` row from the Unified Credentials required-fields table.

Delete the complete **Unified — Central On-Prem with custom token endpoint** section and its example. Do not replace it with a unified or GLP example.

- [ ] **Step 2: Keep one New Central On-Prem section**

Retain the New Central section with concise wording:

```markdown
**Custom Token Endpoint** _(Central On-Prem only)_:

For Central On-Prem deployments whose OAuth issuer differs from the
standard Cloud issuer, set `token_endpoint` to the complete token URL.
The SDK uses it for initial token creation and automatic renewal.
`base_url` remains the Central API URL and may use a different host.
This option is supported only under `new_central`.
```

Keep the existing `new_central` example with separate `base_url` and `token_endpoint` values. Use different host placeholders to make their independence explicit:

```python
token_info = {
    "new_central": {
        "client_id": "<client-id>",
        "client_secret": "<client-secret>",
        "base_url": "https://<on-prem-central-api-host>",
        "token_endpoint": "http://<on-prem-token-issuer>/oauth2/token"
    }
}
```

- [ ] **Step 3: Check documentation scope and formatting**

Run:

```bash
rg -n "token_endpoint|On-Prem" docs/getting-started/authentication.md
git diff --check
```

Expected:
- `token_endpoint` appears only in the New Central On-Prem explanation and example.
- No Unified or GLP text advertises `token_endpoint`.
- `git diff --check` exits with status 0.

- [ ] **Step 4: Commit the documentation change**

```bash
git add docs/getting-started/authentication.md
git commit -m "docs: narrow token endpoint scope"
```

### Task 3: Review the complete implementation

**Assigned agent:** One separate `gpt-5.6-luna` review subagent, coordinated by the orchestrator. This agent must not modify files.

**Files:**
- Review: `pycentral/utils/base_utils.py`
- Review: `docs/getting-started/authentication.md`

**Interfaces:**
- Consumes: the complete diff produced by Tasks 1 and 2.
- Produces: a read-only list of high-confidence correctness, compatibility, and scope findings.

- [ ] **Step 1: Give Luna the review contract**

The orchestrator must provide the fixed implementation range and ask Luna to verify:

```text
Review the implementation diff against this plan. Report only high-confidence
bugs or requirement mismatches. Confirm:
1. Existing Cloud new_central and glp defaults remain AUTHENTICATION["OAUTH"].
2. Unified remains AUTHENTICATION["OAUTH_GLOBAL"]/{workspace_id}/token.
3. token_endpoint works only for new_central.
4. glp/unified key presence raises ValueError before fallback.
5. HTTP and HTTPS remain accepted through valid_url().
6. base_url and token_endpoint stay independent.
7. Documentation advertises only New Central On-Prem support.
8. No on_prem flag, endpoint derivation, helper abstraction, dependencies,
   tests, MSP changes, or unrelated refactors were added.
Do not edit files.
```

- [ ] **Step 2: Resolve review findings**

The orchestrator sends any confirmed finding back to the same Terra implementation subagent for correction. Reuse Terra rather than creating another implementation agent. Reject stylistic suggestions, speculative abstractions, and requests for a new test framework.

- [ ] **Step 3: Re-run final validation**

Run:

```bash
python3 -m py_compile pycentral/utils/base_utils.py
git diff --check
git status --short
```

Expected:
- Syntax and diff checks exit with status 0.
- Only intended plan files are changed relative to the implementation starting point.

- [ ] **Step 4: Commit review corrections if needed**

If Terra made corrections:

```bash
git add pycentral/utils/base_utils.py docs/getting-started/authentication.md
git commit -m "fix: address token endpoint review"
```

If Luna reports no confirmed findings, do not create an empty commit.
