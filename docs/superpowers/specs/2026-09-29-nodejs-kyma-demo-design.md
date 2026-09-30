# Node.js Kyma demo design

## Goal

Build a minimal Node.js Hello World service and a GitHub Actions pipeline that tests, builds, pushes, and deploys it to the `hello-kyma` cluster.

## Application

The service uses Node.js built-in `http` module and has no runtime dependencies.

- `GET /` returns `Hello World` as plain text with HTTP 200.
- `GET /healthz` returns `OK` as plain text with HTTP 200.
- Other paths return HTTP 404.
- The server listens on `PORT`, defaulting to `3000`.
- SIGTERM closes the HTTP server cleanly for Kubernetes termination.

A small test uses Node.js built-in test runner and sends real HTTP requests to an ephemeral server port.

## Container

A multi-stage Dockerfile uses a pinned Node.js LTS Alpine image. The runtime image runs as the built-in non-root `node` user, exposes port 3000, and starts the service directly with Node.js. A `.dockerignore` excludes Git metadata, local dependencies, tests, documentation, and kubeconfig files.

Image repository:

`gtlc-production.common.repositories.cloud.sap/demoappdeploytokyma`

The pipeline publishes both an immutable commit SHA tag and `latest`. Kubernetes deploys only the SHA tag.

## Kubernetes deployment

Plain manifests avoid Helm and Kustomize overhead for one application and one environment.

Resources:

- Namespace `demoapp`
- Deployment `hello-world` with one replica
- ClusterIP Service `hello-world` on port 80 targeting port 3000
- Kyma APIRule `hello-world` exposing the service without authentication

Public URL:

`https://demoapp.c-1282d6a.stage.kyma.ondemand.com`

The Deployment uses readiness and liveness probes against `/healthz`, conservative CPU and memory requests/limits, and the `hello-world-registry` image pull secret. The registry Secret is created or updated by the pipeline and is not committed.

## CI/CD workflow

The workflow runs on pushes to `main` and through manual dispatch. It uses the repository's GitHub Actions runner and performs these steps:

1. Check out source.
2. Install the pinned Node.js version.
3. Run `npm test`.
4. Log in to `gtlc-production.common.repositories.cloud.sap`.
5. Build and push the commit SHA image and `latest`.
6. Decode the kubeconfig GitHub secret into a temporary file.
7. Create or update namespace `demoapp`.
8. Create or update `hello-world-registry` from registry credentials.
9. Apply Kubernetes manifests.
10. set the Deployment image to the immutable SHA tag.
11. Wait for rollout completion.
12. Request the public URL and require the exact `Hello World` response.

Concurrency allows only one deployment to this environment at a time and cancels an older in-progress run.

## GitHub secrets

- `KUBECONFIG`: base64-encoded content of `/Users/I758015/Downloads/hello-kyma-long-lived.kubeconfig.yaml`
- `REGISTRY_USERNAME`: `reposgtlc`
- `REGISTRY_PASSWORD`: supplied registry password

Secrets are passed only to login, pull-secret creation, and kubeconfig setup. Workflow logs must not print their values.

## Error handling

The application returns 404 for unknown paths and handles graceful shutdown. GitHub Actions uses failing shell semantics. It stops on test, build, push, Kubernetes rollout, or endpoint verification failure.

## Verification

Local checks:

- Node test suite passes.
- Container builds for `linux/amd64`.
- Kubernetes manifests pass client-side rendering and server-side dry-run where supported.
- Workflow YAML parses.

Deployment checks:

- Rollout reports success.
- Pods become ready.
- Public endpoint returns HTTP 200 with body `Hello World`.

## Security

- Runtime process is non-root.
- Registry credentials and kubeconfig exist only as GitHub secrets and temporary runner files.
- No credential is committed.
- The public endpoint has no authentication by explicit requirement.
- The long-lived kubeconfig has cluster-admin access and must be rotated if exposed.
