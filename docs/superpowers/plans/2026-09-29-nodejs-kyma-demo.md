# Node.js Kyma Demo Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build, test, containerize, publish, and deploy a minimal Node.js Hello World service to the `demoapp` namespace in the hello-kyma cluster.

**Architecture:** A dependency-free Node.js HTTP server exposes application and health endpoints. A Docker image runs as a non-root user. Plain Kubernetes manifests deploy the image and expose it through a Kyma APIRule; one GitHub Actions workflow tests, publishes SHA and `latest` tags, applies manifests, and verifies rollout and public access.

**Tech Stack:** Node.js 22, built-in `node:http` and `node:test`, Docker, Kubernetes, Kyma APIRule v2, GitHub Actions

**Spec:** `docs/superpowers/specs/2026-09-29-nodejs-kyma-demo-design.md`

## Global Constraints

- Use no Node.js runtime dependencies.
- Deploy to namespace `demoapp`.
- Push to `gtlc-production.common.repositories.cloud.sap/demoappdeploytokyma`.
- Expose `https://demoapp.c-1282d6a.stage.kyma.ondemand.com` without authentication.
- Deploy an immutable commit SHA image tag; also publish `latest`.
- Run the container as a non-root user.
- Never commit registry credentials or kubeconfig content.
- Use GitHub secrets `KUBECONFIG`, `REGISTRY_USERNAME`, and `REGISTRY_PASSWORD`.
- Treat the long-lived kubeconfig as a cluster-admin credential and never print or commit it.

---

### Task 1: HTTP application

**Files:**
- Create: `package.json`
- Create: `server.js`
- Create: `test/server.test.js`

**Interfaces:**
- Produces: `createServer()` returning a Node.js `http.Server`
- Produces: executable server listening on `process.env.PORT || 3000`
- HTTP contract: `/` returns `200`, `text/plain`, `Hello World`; `/healthz` returns `200`, `text/plain`, `OK`; other paths return `404`

- [ ] **Step 1: Write the failing HTTP test**

```js
const { after, before, test } = require('node:test');
const assert = require('node:assert/strict');
const { createServer } = require('../server');

let server;
let baseUrl;

before(async () => {
  server = createServer();
  await new Promise(resolve => server.listen(0, '127.0.0.1', resolve));
  baseUrl = `http://127.0.0.1:${server.address().port}`;
});

after(() => new Promise(resolve => server.close(resolve)));

test('GET / returns Hello World', async () => {
  const response = await fetch(`${baseUrl}/`);
  assert.equal(response.status, 200);
  assert.match(response.headers.get('content-type'), /^text\/plain/);
  assert.equal(await response.text(), 'Hello World');
});

test('GET /healthz returns OK', async () => {
  const response = await fetch(`${baseUrl}/healthz`);
  assert.equal(response.status, 200);
  assert.equal(await response.text(), 'OK');
});

test('unknown path returns 404', async () => {
  const response = await fetch(`${baseUrl}/missing`);
  assert.equal(response.status, 404);
});
```

- [ ] **Step 2: Add the test command and verify RED**

Create `package.json`:

```json
{
  "name": "demoappdeploytokyma",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "start": "node server.js",
    "test": "node --test"
  },
  "engines": {
    "node": ">=22"
  }
}
```

Run: `npm test`

Expected: FAIL because `../server` does not exist.

- [ ] **Step 3: Implement the minimum server**

Create `server.js`:

```js
const http = require('node:http');

function createServer() {
  return http.createServer((request, response) => {
    const routes = { '/': 'Hello World', '/healthz': 'OK' };
    const body = routes[request.url];
    response.writeHead(body === undefined ? 404 : 200, { 'Content-Type': 'text/plain; charset=utf-8' });
    response.end(body);
  });
}

if (require.main === module) {
  const server = createServer().listen(process.env.PORT || 3000);
  process.on('SIGTERM', () => server.close());
}

module.exports = { createServer };
```

- [ ] **Step 4: Verify GREEN**

Run: `npm test`

Expected: 3 tests pass, 0 fail.

- [ ] **Step 5: Commit**

```bash
git add package.json server.js test/server.test.js
git commit -m "feat: add Hello World service"
```

---

### Task 2: Container image

**Files:**
- Create: `Dockerfile`
- Create: `.dockerignore`

**Interfaces:**
- Consumes: `package.json`, `server.js`
- Produces: Linux amd64 image listening on port 3000 and running as UID 1000

- [ ] **Step 1: Write the container behavior check**

Run before creating the Dockerfile:

```bash
docker build -t demoappdeploytokyma:test .
```

Expected: FAIL because `Dockerfile` does not exist.

- [ ] **Step 2: Add the minimal image definition**

Create `Dockerfile`:

```dockerfile
FROM node:22-alpine AS test
WORKDIR /app
COPY package.json server.js ./
COPY test ./test
RUN npm test

FROM node:22-alpine
WORKDIR /app
ENV NODE_ENV=production PORT=3000
COPY --from=test /app/package.json /app/server.js ./
USER node
EXPOSE 3000
CMD ["node", "server.js"]
```

Create `.dockerignore`:

```text
.git
.github
.worktrees
node_modules
test
docs
*.kubeconfig
*kubeconfig*
```

- [ ] **Step 3: Build and test the image**

Run:

```bash
docker build --platform linux/amd64 -t demoappdeploytokyma:test .
CID=$(docker run -d -p 127.0.0.1::3000 demoappdeploytokyma:test)
PORT=$(docker port "$CID" 3000/tcp | sed 's/.*://')
trap 'docker rm -f "$CID"' EXIT
test "$(curl -fsS "http://127.0.0.1:$PORT/")" = "Hello World"
test "$(docker inspect -f '{{.Config.User}}' demoappdeploytokyma:test)" = "node"
```

Expected: image builds; endpoint returns `Hello World`; image user is `node`.

- [ ] **Step 4: Commit**

```bash
git add Dockerfile .dockerignore
git commit -m "build: containerize Hello World service"
```

---

### Task 3: Kubernetes resources

**Files:**
- Create: `k8s/deployment.yaml`
- Create: `k8s/service.yaml`
- Create: `k8s/apirule.yaml`

**Interfaces:**
- Consumes: image repository and SHA tag substituted by GitHub Actions
- Produces: Deployment and Service named `hello-world`, public APIRule hostname `demoapp.c-1282d6a.stage.kyma.ondemand.com`
- Requires: namespace and image pull Secret created by workflow

- [ ] **Step 1: Add a manifest validation check that initially fails**

Run:

```bash
kubectl apply --dry-run=client -f k8s/
```

Expected: FAIL because `k8s/` does not exist.

- [ ] **Step 2: Create Deployment**

Create `k8s/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  namespace: demoapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      imagePullSecrets:
        - name: hello-world-registry
      containers:
        - name: hello-world
          image: gtlc-production.common.repositories.cloud.sap/demoappdeploytokyma:IMAGE_TAG
          ports:
            - name: http
              containerPort: 3000
          readinessProbe:
            httpGet:
              path: /healthz
              port: http
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
          resources:
            requests:
              cpu: 10m
              memory: 32Mi
            limits:
              cpu: 100m
              memory: 64Mi
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
            runAsNonRoot: true
            seccompProfile:
              type: RuntimeDefault
```

- [ ] **Step 3: Create Service and APIRule**

Create `k8s/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-world
  namespace: demoapp
spec:
  selector:
    app: hello-world
  ports:
    - name: http
      port: 80
      targetPort: http
```

Create `k8s/apirule.yaml`:

```yaml
apiVersion: gateway.kyma-project.io/v2
kind: APIRule
metadata:
  name: hello-world
  namespace: demoapp
spec:
  gateway: kyma-system/kyma-gateway
  hosts:
    - demoapp.c-1282d6a.stage.kyma.ondemand.com
  service:
    name: hello-world
    port: 80
  rules:
    - path: /*
      methods: [GET]
      noAuth: true
```

- [ ] **Step 4: Validate manifests against the cluster**

Run:

```bash
kubectl --kubeconfig /Users/I758015/Downloads/hello-kyma-long-lived.kubeconfig.yaml apply --dry-run=server -f k8s/
```

Expected: all three resources validate. The namespace must exist for server-side validation; create it first with `kubectl create namespace demoapp` only after the user-approved deployment mutation.

- [ ] **Step 5: Commit**

```bash
git add k8s
git commit -m "feat: add Kyma deployment manifests"
```

---

### Task 4: GitHub Actions deployment pipeline

**Files:**
- Create: `.github/workflows/deploy.yml`
- Modify: `README.md`

**Interfaces:**
- Consumes GitHub secrets: `KUBECONFIG` as base64 text, `REGISTRY_USERNAME`, `REGISTRY_PASSWORD`
- Produces image tags `${GITHUB_SHA}` and `latest`; deploys `${GITHUB_SHA}`

- [ ] **Step 1: Verify workflow is absent**

Run:

```bash
test -f .github/workflows/deploy.yml
```

Expected: FAIL.

- [ ] **Step 2: Create workflow**

Create `.github/workflows/deploy.yml` with:

```yaml
name: Build and deploy

on:
  push:
    branches: [main]
  workflow_dispatch:

concurrency:
  group: hello-kyma-demo
  cancel-in-progress: true

permissions:
  contents: read

env:
  REGISTRY: gtlc-production.common.repositories.cloud.sap
  IMAGE: gtlc-production.common.repositories.cloud.sap/demoappdeploytokyma
  NAMESPACE: demoapp

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm test
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_PASSWORD }}
      - uses: docker/setup-buildx-action@v3
      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          platforms: linux/amd64
          tags: |
            ${{ env.IMAGE }}:${{ github.sha }}
            ${{ env.IMAGE }}:latest
      - name: Configure Kubernetes
        env:
          KUBECONFIG_BASE64: ${{ secrets.KUBECONFIG }}
        run: |
          printf '%s' "$KUBECONFIG_BASE64" | base64 --decode > "$RUNNER_TEMP/kubeconfig"
          chmod 600 "$RUNNER_TEMP/kubeconfig"
          echo "KUBECONFIG=$RUNNER_TEMP/kubeconfig" >> "$GITHUB_ENV"
      - name: Deploy
        env:
          REGISTRY_USERNAME: ${{ secrets.REGISTRY_USERNAME }}
          REGISTRY_PASSWORD: ${{ secrets.REGISTRY_PASSWORD }}
        run: |
          kubectl create namespace "$NAMESPACE" --dry-run=client -o yaml | kubectl apply -f -
          kubectl -n "$NAMESPACE" create secret docker-registry hello-world-registry \
            --docker-server="$REGISTRY" \
            --docker-username="$REGISTRY_USERNAME" \
            --docker-password="$REGISTRY_PASSWORD" \
            --dry-run=client -o yaml | kubectl apply -f -
          sed "s/IMAGE_TAG/${GITHUB_SHA}/g" k8s/deployment.yaml | kubectl apply -f -
          kubectl apply -f k8s/service.yaml -f k8s/apirule.yaml
          kubectl -n "$NAMESPACE" rollout status deployment/hello-world --timeout=180s
      - name: Verify public endpoint
        run: test "$(curl --fail --silent --show-error --retry 12 --retry-delay 5 https://demoapp.c-1282d6a.stage.kyma.ondemand.com/)" = "Hello World"
```

Do not add npm caching if no lockfile is produced; `actions/setup-node` requires a dependency lockfile for `cache: npm`. Since this app has no dependencies, omit the `cache` line in the final workflow.

- [ ] **Step 3: Document usage**

Replace `README.md` with concise sections covering local commands, public URL, deployment trigger, three required GitHub secrets, and this exact kubeconfig setup command:

```bash
base64 < /Users/I758015/Downloads/hello-kyma-long-lived.kubeconfig.yaml | tr -d '\n' | GH_HOST=github.tools.sap gh secret set KUBECONFIG --repo GTLCInfra/demoappdeploytokyma
```

Document `REGISTRY_USERNAME` and `REGISTRY_PASSWORD` by secret name only. Never include their values.

- [ ] **Step 4: Validate workflow syntax and secret references**

Run:

```bash
ruby -e 'require "yaml"; YAML.load_file(".github/workflows/deploy.yml", aliases: true); puts "valid YAML"'
grep -R "reposgtlc\|cmVmdGtu" . --exclude-dir=.git && exit 1 || true
```

Expected: YAML parses; no credential value is found.

- [ ] **Step 5: Commit**

```bash
git add .github/workflows/deploy.yml README.md
git commit -m "ci: build and deploy app to Kyma"
```

---

### Task 5: Configure repository secrets and perform end-to-end deployment

**Files:**
- No tracked file changes

**Interfaces:**
- Consumes local long-lived kubeconfig and supplied registry credentials
- Produces three GitHub Actions repository secrets and a successful workflow run

- [ ] **Step 1: Set repository secrets without printing values**

Run `gh secret set` for:

- `REGISTRY_USERNAME`
- `REGISTRY_PASSWORD`
- `KUBECONFIG`, using base64-encoded kubeconfig content

Use `GH_HOST=github.tools.sap` and repository `GTLCInfra/demoappdeploytokyma`.

- [ ] **Step 2: Push the feature branch and open a pull request**

Push `feature/nodejs-kyma-demo`. Open a pull request to `main` describing the application, pipeline, manifests, security model, and verification.

- [ ] **Step 3: Review before merge**

Review the branch diff for correctness, secret leakage, and unnecessary complexity. Fix every critical or important finding.

- [ ] **Step 4: Merge only with explicit user approval**

The automatic deployment triggers only on `main`. Do not merge without user approval.

- [ ] **Step 5: Verify deployment after merge**

Run:

```bash
GH_HOST=github.tools.sap gh run watch <run-id> --repo GTLCInfra/demoappdeploytokyma
kubectl --kubeconfig /Users/I758015/Downloads/hello-kyma-long-lived.kubeconfig.yaml -n demoapp rollout status deployment/hello-world --timeout=180s
curl --fail --silent --show-error https://demoapp.c-1282d6a.stage.kyma.ondemand.com/
```

Expected: workflow succeeds, rollout completes, endpoint returns `Hello World`.
