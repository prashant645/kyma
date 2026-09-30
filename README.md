# demoappdeploytokyma

Minimal Node.js Hello World service deployed to SAP BTP Kyma.

## Run locally

```bash
npm test
npm start
curl http://localhost:3000/
```

## Branching and deployment

This repository follows the FIPC branching strategy, adapted for Kyma.
The full contract is in **[CONTRIBUTING.md](CONTRIBUTING.md)**.

| Branch | Environment | Namespace | Deploy |
|---|---|---|---|
| any PR, `feature/**`, `bugfix/**`, `hotfix/**` | none | — | build + test only |
| `develop` | dev | `dev` | automatic |
| `master` | qa | `qa` | automatic |
| `rel.YYYY.MM.DD[-hotfix.N]` | qa (RC) | `qa` | automatic |
| `vYYYY.MM.DD[-hotfix.N]` | prod | `prod` | automatic, after reviewer approval |

Each environment is a separate BTP subaccount / Kyma cluster. Manifests are the Helm
chart in [helm/demoapp](helm/demoapp) with one values file per environment.

Cutting a release:

```bash
./scripts/cut-release-candidate.sh 2026.09.29   # rel.2026.09.29 -> qa
./scripts/cut-release.sh 2026.09.29             # v2026.09.29    -> prod (gated)
```

## Public URLs

| Env | URL |
|---|---|
| dev | `https://demoapp-dev.c-1282d6a.stage.kyma.ondemand.com` |
| qa | not provisioned yet — see CONTRIBUTING.md |
| prod | not provisioned yet — see CONTRIBUTING.md |

## Required repository secrets

`KYMA_KUBECONFIG_DEV`, `KYMA_KUBECONFIG_QA`, `KYMA_KUBECONFIG_PROD` (base64-encoded
kubeconfig content), plus `REGISTRY_USERNAME` and `REGISTRY_PASSWORD`.

```bash
base64 < kyma-dev.kubeconfig.yaml | tr -d '\n' \
  | GH_HOST=github.tools.sap gh secret set KYMA_KUBECONFIG_DEV \
      --repo GTLCInfra/demoappdeploytokyma
```

One-time repository setup (branch migration, protection, the `production`
environment): **[docs/branch-protection.md](docs/branch-protection.md)**.
