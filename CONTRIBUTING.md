# Contributing

## Branching strategy

This repository follows the same branching strategy as the FIPC services, adapted
for deployment to SAP BTP Kyma. See the FIPC wiki entry:
<https://github.tools.sap/GTLCDEV-FossCompliance/Design/wiki/Branching-Strategy>

| Branch pattern | Environment | Namespace | Workflow | Deploy |
|---|---|---|---|---|
| any PR, `feature/**`, `bugfix/**`, `hotfix/**` | none | — | `ci.yaml` | build + test only |
| `develop` | dev | `dev` | `deploy-dev.yaml` | automatic |
| `master` | qa | `qa` | `deploy-qa.yaml` | automatic |
| `rel.YYYY.MM.DD[-hotfix.N]` | qa (release candidate) | `qa` | `deploy-qa.yaml` | automatic |
| `vYYYY.MM.DD[-hotfix.N]` | prod | `prod` | `deploy-prod.yaml` | automatic **after approval** |

`develop` is the default branch. All work starts from `develop` and lands there by PR.

### Flow

```
feature/xyz ──PR──> develop ──PR──> master
                       │              │
                    (dev env)      (qa env)
                                      │
                                  rel.2026.09.29   (qa, release candidate)
                                      │
                                  v2026.09.29      (prod, approval gate)
```

1. Branch from `develop`, open a PR back into it. CI builds and tests; nothing deploys.
2. Merge to `develop` — deploys to **dev**.
3. Merge `develop` into `master` — deploys to **qa**.
4. Cut a release candidate: `./scripts/cut-release-candidate.sh 2026.09.29` — also
   deploys to **qa**, but from a frozen branch so `master` can keep moving.
5. Promote: `./scripts/cut-release.sh 2026.09.29` — creates `v2026.09.29` and queues
   the **prod** deploy, which waits on a required reviewer.
6. Merge the release branch back into `master` and `develop`.

Hotfixes reuse the same shapes: `rel.2026.09.29-hotfix.1` then `v2026.09.29-hotfix.1`.

### Difference from FIPC

FIPC deploys `v*` to production with no human gate. Here the prod job is bound to the
`production` GitHub environment, so it pauses for a required reviewer before it touches
the cluster. Everything else — the branch names, the regexes, the environment mapping —
is the same contract.

## Environments

Each environment is a **separate BTP subaccount / Kyma cluster**, addressed by its own
base64-encoded kubeconfig secret:

| Env | Secret | Values file | Host |
|---|---|---|---|
| dev | `KYMA_KUBECONFIG_DEV` | `helm/demoapp/values-dev.yaml` | `demoapp-dev.c-1282d6a.stage.kyma.ondemand.com` |
| qa | `KYMA_KUBECONFIG_QA` | `helm/demoapp/values-qa.yaml` | `<QA_SHOOT>` — **not provisioned yet** |
| prod | `KYMA_KUBECONFIG_PROD` | `helm/demoapp/values-prod.yaml` | `<PROD_SHOOT>` — **not provisioned yet** |

Shared secrets: `REGISTRY_USERNAME`, `REGISTRY_PASSWORD`.

Until the QA and PROD subaccounts exist, `master` / `rel.*` / `v*` builds run their tests
and then **fail at the placeholder guard by design** (`scripts/check-env-placeholders.sh`).
Replace the `<QA_SHOOT>` / `<PROD_SHOOT>` placeholders in the values files to arm them.

Setting a kubeconfig secret:

```bash
base64 < kyma-dev.kubeconfig.yaml | tr -d '\n' \
  | GH_HOST=github.tools.sap gh secret set KYMA_KUBECONFIG_DEV --repo GTLCInfra/demoappdeploytokyma
```

## Deployment mechanics

Manifests are a Helm chart at `helm/demoapp`, with one values file per environment.
The pipeline always passes an immutable tag — `<branch>-<run-number>-<short-sha>` — and
the chart refuses to render without one. Each environment pushes to its own image
repository (`demoappdeploytokyma-dev|-qa|-prod`), mirroring FIPC's `fipc-dev|-qa|-prod`.

`helm upgrade --install --atomic` means a failed rollout reverts itself. On dev and qa a
failed APIRule wait or smoke test additionally triggers an explicit `helm rollback`
(FIPC runs its rollback stage on dev/qa only). Prod has no auto-rollback: it is gated by
a reviewer and rolled back deliberately.

Deploy locally against dev:

```bash
helm upgrade --install hello-world helm/demoapp -n dev --create-namespace \
  -f helm/demoapp/values-dev.yaml --set image.tag=<tag>
```

## Reviews

Each PR needs at least the service owner as reviewer — enforced via
[`.github/CODEOWNERS`](.github/CODEOWNERS).

## Not carried over from FIPC

- **BlackDuck FOSS scan** on dev/qa. FIPC runs `BlackDuckScan(...)` from the
  `gtlc-pipe-libs` Jenkins shared library; there is no GitHub Actions equivalent wired
  up here yet.
- **e2e tests** from the separate `e2e-test` repository on dev/qa. The smoke test in
  `scripts/smoke-test.sh` covers the endpoints this service actually has.
