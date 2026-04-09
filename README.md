# Helm OCI Nested Path Bug Test

Test pipeline for a JFrog CLI bug where pushing a Helm OCI artifact to a **nested path** while build association environment variables are set causes a 404 error.

## Bug Summary

When the OCI target URL contains subdirectories (e.g., `oci://host/REPO/subdir1/subdir2`), the CLI:
1. Incorrectly extracts the **first path segment after the host** as the Artifactory repository name
2. **Truncates the subdirectory** components from the AQL artifact path

This causes a **404 Not Found** when trying to assign build properties to the artifact.

### AQL Comparison

| | Repo in AQL | Path in AQL |
|---|---|---|
| **Incorrect** (nested path) | `subdir1` | `test-chart/0.1.0` |
| **Correct** (expected) | `helm-local` | `subdir1/subdir2/test-chart/0.1.0` |

## Test Scenarios

### Scenario A — Root path + build vars → PASS
```bash
export JFROG_BUILD_NAME="my-build"
export JFROG_BUILD_NUMBER="1"
jf helm push test-chart-0.1.0.tgz oci://<host>/helm-local
```

### Scenario B — Nested path + build vars → FAIL (404)
```bash
export JFROG_BUILD_NAME="my-build"
export JFROG_BUILD_NUMBER="1"
jf helm push test-chart-0.1.0.tgz oci://<host>/helm-local/subdir1/subdir2
```

### Scenario C — Nested path, no build vars → PASS
```bash
# JFROG_BUILD_NAME / JFROG_BUILD_NUMBER not set
jf helm push test-chart-0.1.0.tgz oci://<host>/helm-local/subdir1/subdir2
```

## Running the Pipeline

Go to **Actions → Helm OCI Nested Path Bug Test → Run workflow**.

You can optionally specify a different JFrog CLI version to test via the `jfrog_cli_version` input.

## Required Secrets

| Secret | Description |
|--------|-------------|
| `JF_URL` | Artifactory base URL |
| `JF_USER` | Artifactory username |
| `JF_PASSWORD` | Artifactory password or API key |
