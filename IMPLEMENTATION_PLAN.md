# Centralized ACR Image Pipeline

## Goal

Replace the empty root workflow with one secure, centralized pipeline for the four Docker image projects: `axion-data-simulator`, `axion-ingestion-service`, `axion-telemetry-query-service`, and `axion-ui`. The matrix executes each image concurrently and keeps Dockerfile linting, image building, vulnerability scanning, and ACR publishing as separate stages.

`axion-database-schema` is excluded because it does not contain a Dockerfile.

## Clear Diff Summary

| Area | Current state | Planned state |
| --- | --- | --- |
| Root workflow | Contains only a name | Central matrix pipeline with lint, build, scan, and push jobs |
| Registry | Four local workflows push to Docker Hub | One root workflow publishes to Azure Container Registry (ACR) |
| Authentication | Long-lived Docker Hub token secrets | GitHub Actions OIDC federation; no Azure client secret or registry password |
| Parallelism | One workflow per service | Each stage uses a four-image matrix; builds and pushes run concurrently |
| Scan gate | None | HIGH/CRITICAL findings block publishing by default; manual runs can opt out |
| Build contexts | No `.dockerignore` files | Minimal per-service `.dockerignore` files |

## Workflow Design

### Triggers and permissions

Update `.github/workflows/buildAndPush.yml` to run for pull requests and pushes to `main`, plus `workflow_dispatch`. Apply path filters for this workflow and the four image directories.

Manual dispatch exposes this input:

```yaml
inputs:
  skip_scan_failure:
    description: Allow publishing even when Trivy finds HIGH or CRITICAL vulnerabilities
    required: false
    default: false
    type: boolean
```

Automated `main` pushes always enforce the scan gate. Pull-request scans are enabled by default; set the repository variable `RUN_IMAGE_SCAN_ON_PR` to the literal string `false` to skip the complete scan stage for pull-request events only. Only a manually dispatched run with `skip_scan_failure: true` can continue after a qualifying Trivy finding.

Set default workflow permissions to `contents: read`, then grant `id-token: write` only to the ACR push job. This makes an OIDC token available only where Azure authentication is needed.

### ACR and OIDC configuration

Use GitHub repository **variables** (not secrets) for non-sensitive identity and registry identifiers:

```yaml
env:
  ACR_NAME: ${{ vars.ACR_NAME }}
  AZURE_CLIENT_ID: ${{ vars.AZURE_CLIENT_ID }}
  AZURE_TENANT_ID: ${{ vars.AZURE_TENANT_ID }}
  AZURE_SUBSCRIPTION_ID: ${{ vars.AZURE_SUBSCRIPTION_ID }}
```

`ACR_NAME` is the registry resource name only (for example, `axionprod`); the workflow derives the login server as `${ACR_NAME}.azurecr.io`. Do not store `AZURE_CLIENT_SECRET`, registry passwords, or Docker Hub credentials in the repository.

In the push job, authenticate with the fixed `azure/login@v3.1.0` action using the three Azure identity variables and `auth-type: OIDC`, then use `az acr login --name "$ACR_NAME"`. This lets Docker push using a short-lived Azure access token instead of a password.

Before enabling the workflow, configure the Entra application or managed identity as follows:

1. Add a federated identity credential with issuer `https://token.actions.githubusercontent.com`, audience `api://AzureADTokenExchange`, and a subject restricted to this repository's `main` branch.
2. Assign that identity the `AcrPush` role at the target ACR resource scope.
3. Set the four repository variables listed above. Limit who can edit Actions workflows and repository variables.

### Matrix stages

Define one matrix mapping each source directory to its ACR repository path. Preserve the existing image names: `axion-data-simulator`, `axion-ingestion-service`, `axion-telemetry-query-service`, and `axion-ui`.

1. **Lint** — Run Hadolint against each Dockerfile concurrently. Fail before building when a Dockerfile violates the selected policy.
2. **Build** — After lint succeeds, build each image concurrently with Buildx and export an OCI image archive as a uniquely named, short-retention GitHub Actions artifact. Builds never push to a registry. Use a distinct GitHub Actions cache scope per service.
3. **Scan** — Download and scan the exact archive produced by the build with Trivy. Scan HIGH and CRITICAL severities and upload the human-readable report as an artifact even when scanning fails. The Trivy step normally fails the job; configure `continue-on-error` only when the event is `workflow_dispatch` and `inputs.skip_scan_failure` is true. The entire stage is skipped for pull requests only when `RUN_IMAGE_SCAN_ON_PR=false`.
4. **Push** — After the scan stage succeeds (or is explicitly bypassed), download the same archive, load it, use Azure OIDC login plus `az acr login`, and push images concurrently to `${ACR_NAME}.azurecr.io/<image>`. Publish an immutable Git commit-SHA tag on every approved publish and update `latest` only on `main`.

Use a branch/workflow concurrency group to cancel superseded runs. Pin every third-party action to an exact release version (for example, `actions/checkout@v5.0.0`), rather than a floating major-version tag. Disable persisted Git checkout credentials. Keep Azure login and all Azure variables out of pull-request jobs; pull requests run only lint, build, and scan.

## Companion File Changes

- Delete the four service-local `docker-build-push.yml` workflows. This removes the Docker Hub credential dependency and prevents double builds or double publishing.
- Add `.dockerignore` to each image directory. Exclude Git metadata, CI files, editor and OS clutter, local virtual environments, package caches, coverage/test output, and other non-runtime files. Keep required application source, dependency manifests, and UI lockfiles available to their Dockerfiles.

## Verification

- Run `actionlint` against the new workflow.
- Confirm a pull request runs lint/build/scan without requesting an Azure OIDC token, logging into ACR, or pushing an image; confirm `RUN_IMAGE_SCAN_ON_PR=false` skips only its scan stage.
- Confirm a clean `main` push publishes each image once with a commit-SHA tag and `latest`.
- Confirm a HIGH/CRITICAL test finding fails an automated run and prevents ACR pushes.
- Confirm manual dispatch with `skip_scan_failure: true` records and uploads the scan report but allows the push stage.
- Confirm the Entra federation rejects tokens from a branch or repository outside the configured subject.
