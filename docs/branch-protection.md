# Branch protection and repository setup

Companion to [CONTRIBUTING.md](../CONTRIBUTING.md). These are the settings the
branching strategy relies on that cannot live in the repository itself.

## 1. Migrate the branches

The repository currently has a single `main` branch. The strategy needs `develop`
(default) and `master`.

```bash
export GH_HOST=github.tools.sap
REPO=GTLCInfra/demoappdeploytokyma

git fetch origin
git checkout -b develop origin/main && git push -u origin develop
git checkout -b master  origin/main && git push -u origin master

# Make develop the default so new PRs target it
gh repo edit "$REPO" --default-branch develop

# Re-target any open PRs onto develop, then retire main
gh pr list --repo "$REPO" --base main --json number \
  --jq '.[].number' | xargs -I{} gh pr edit {} --repo "$REPO" --base develop
git push origin --delete main
```

## 2. Apply protection

```bash
./scripts/apply-branch-protection.sh GTLCInfra/demoappdeploytokyma
```

Confirm the CI status-check name first — it is the **job name**, not the workflow
file name:

```bash
GH_HOST=github.tools.sap gh pr checks <pr-number>
```

Protection applied to `develop` and `master`:

- required status check: the `CI` workflow's test job, strict (branch must be up to date)
- 1 approving review, code-owner review required, stale reviews dismissed
- no force pushes, no deletions

## 3. Create the `production` environment

This is what gates the `v*` → prod deploy. It cannot be set from a file.

Settings → Environments → **New environment** → `production`:

- **Required reviewers**: the service owners
- **Deployment branches**: selected branches only — add the patterns `v*`
- (optional) wait timer

Without this environment the prod job runs unattended, which defeats the gate.

## 4. Protect the release branches

Add a branch ruleset covering `rel.*` and `v*`:

- block force pushes and deletions
- restrict creation to the release managers

Release branches are frozen artifacts; they should only be created by
`scripts/cut-release-candidate.sh` and `scripts/cut-release.sh`.

## 5. Secrets

| Secret | Scope | Notes |
|---|---|---|
| `KYMA_KUBECONFIG_DEV` | repo | base64 of the dev cluster kubeconfig |
| `KYMA_KUBECONFIG_QA` | repo | base64 of the qa cluster kubeconfig |
| `KYMA_KUBECONFIG_PROD` | `production` environment | base64 of the prod cluster kubeconfig |
| `REGISTRY_USERNAME` | repo | Artifactory |
| `REGISTRY_PASSWORD` | repo | Artifactory |

Scoping `KYMA_KUBECONFIG_PROD` to the `production` environment rather than the
repository means no non-prod workflow can read prod credentials.

```bash
base64 < kyma-dev.kubeconfig.yaml | tr -d '\n' \
  | GH_HOST=github.tools.sap gh secret set KYMA_KUBECONFIG_DEV \
      --repo GTLCInfra/demoappdeploytokyma
```

## 6. Runners

All jobs use `runs-on: self-hosted`, matching `hello-world-service-release-kyma`.
The Kyma API servers are not reachable from GitHub-hosted runners. The runner also
needs Docker for the image build.
