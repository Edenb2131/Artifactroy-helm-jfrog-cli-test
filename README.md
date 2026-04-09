# Case 409097 — JFrog CLI Helm OCI Nested Path Bug

## Bug Summary

JFrog CLI (tested on 2.99.0) generates an incorrect AQL query when publishing a Helm OCI artifact to a **nested path** while build association environment variables (`JFROG_BUILD_NAME` / `JFROG_BUILD_NUMBER`) are set.

### Root Cause

When the OCI target URL contains subdirectories (e.g., `oci://host/REPO/subdir1/subdir2`), the CLI:
1. Incorrectly extracts the **first path segment after the host** as the Artifactory repository name
2. **Truncates the subdirectory** components from the AQL artifact path

This causes a **404 Not Found** when trying to assign build properties to the artifact.

### AQL Comparison

| | Repo in AQL | Path in AQL |
|---|---|---|
| **Incorrect** (nested path) | `subdir1` | `test-chart/0.1.0` |
| **Correct** (expected) | `409097-helm-local` | `subdir1/subdir2/test-chart/0.1.0` |

## Reproduction Scenarios

### Scenario A — Root path + build vars → **PASS** (works correctly)
```bash
export JFROG_BUILD_NAME="my-build"
export JFROG_BUILD_NUMBER="1"
jf helm push test-chart-0.1.0.tgz oci://elinaf.jfrog.io/409097-helm-local
```

### Scenario B — Nested path + build vars → **FAIL** (404 bug)
```bash
export JFROG_BUILD_NAME="my-build"
export JFROG_BUILD_NUMBER="1"
jf helm push test-chart-0.1.0.tgz oci://elinaf.jfrog.io/409097-helm-local/subdir1/subdir2
```
**Incorrect AQL generated:**
```json
items.find({"repo": "subdir1", "path": "test-chart/0.1.0"})
```

### Scenario C — Nested path, no build vars → **PASS** (workaround)
```bash
# Do NOT set JFROG_BUILD_NAME / JFROG_BUILD_NUMBER
jf helm push test-chart-0.1.0.tgz oci://elinaf.jfrog.io/409097-helm-local/subdir1/subdir2
```

## Workarounds

1. **Store charts at repo root** (no subdirectories) — not always feasible
2. **Push without build association** (unset `JFROG_BUILD_NAME` / `JFROG_BUILD_NUMBER`) — loses traceability

## JFrog Artifactory

| Repository | Type | Package |
|---|---|---|
| `409097-helm-local` | Local | Helm |
| `409097-helm-remote` | Remote | Helm |
| `409097-helm-virtual` | Virtual | Helm |

Instance: `elinaf.jfrog.io`

## Pipeline

See [`.github/workflows/helm-oci-bug-test.yml`](.github/workflows/helm-oci-bug-test.yml) for the full test pipeline.
Run it via **Actions → Case 409097 → Run workflow** and optionally specify a different CLI version to test.
