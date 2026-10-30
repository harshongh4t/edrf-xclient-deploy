---
name: edrf-xclient-deploy
description: Deploy or troubleshoot the Trellix eDRF XClient container-security Helm chart on OpenShift. Use when the user wants to install, upgrade, debug a failing pod/image-pull, or uninstall the edrf-xclient DaemonSet, or asks about the kubeconfig/oc login, Artifactory pull secret, or ePO reporting for this chart.
---

# eDRF XClient — OpenShift Deployment

Guide the user through deploying the `edrf-xclient` Helm chart (a privileged DaemonSet that
reports container security data to ePO) on OpenShift. `DEPLOYMENT_GUIDE.md`, alongside this
file, is the full reference — complete command blocks for PowerShell, `cmd.exe`, and
Linux/Mac, with verification steps and expected output for every command. This file is the
operational logic: what order things must happen in, and how to diagnose it when they don't.

## Step 1 — Establish the topology

Ask whether the user is working from:
- A **jumpbox** with cluster access but possibly no internet egress to Artifactory — image
  digest lookups (Part C) must then run from a separate, Artifactory-reachable machine.
- A **single machine** with both cluster and internet access — no split required.

Diagnostic: a `curl`/`curl.exe` request to `artifactory.trellix.com` that **hangs** indicates
a network/firewall split, not an auth problem. An **instant error code** (`401`/`403`/`404`)
means stay on the same machine and fix credentials or the URL.

## Step 2 — Enforce the required sequence

Most failures in this deployment come from steps run out of order, not from missing
information. Hold the user to this sequence:

1. Install `oc` and `helm`; confirm both resolve on PATH.
2. Authenticate to the cluster (token login or `KUBECONFIG`); confirm with `oc whoami`.
3. **Create the namespace before creating any secret inside it.** `oc create secret` fails
   with `namespace not found` otherwise — and after fixing the namespace, it's easy to forget
   to re-run the secret command.
4. Generate the Artifactory identity token and create the pull secret with
   `oc create secret docker-registry`. Verify it exists
   (`oc get secret artifactory-pull-secret -n <ns>`) before moving on — never assume a
   command succeeded without checking.
5. Resolve the image digest and populate `values.yaml`: `image.repository`, `image.digest`,
   `imagePullSecret` (the secret's name, not its contents), `epo.ip`/`epo.user`/`epo.password`
   (or `epo.existingSecret`), and `deploymentDetails.clusterName`.
6. `helm upgrade --install ... --dry-run=client` first; resolve every `required` error before
   a real install.
7. Real install (same command, `--dry-run=client` removed).
8. Link the pull secret to the service account:
   `oc secrets link edrf-admin artifactory-pull-secret --for=pull -n <ns>`. This only
   succeeds after the install has created `edrf-admin`.
9. Verify pods, then check pod logs for actual ePO connectivity — `Running` only confirms the
   container started.

## Step 3 — Recognize these specific failure modes

- **Reference Token vs. Token ID.** Artifactory's identity-token screen shows both. The long
  **Reference token** string is the real credential. The short UUID **Token ID** is an
  internal label for finding/revoking the token later — using it as a password produces
  `401 Unauthorized` / `Authentication is required`.
- **`imagePullSecret` in `values.yaml` is a name, not a credential** — e.g.
  `"artifactory-pull-secret"`.
- **Never hand-build the `dockerconfigjson`/`config.json` file.** Its `auth` field must be
  base64 of `username:password` — a common mistake is pasting the raw token into `auth` with
  no username or base64 step. Always use `oc create secret docker-registry` instead.
- **`namespace not found` on secret creation** → the namespace wasn't created yet, or the
  secret command was never re-run after fixing the namespace. Check with
  `oc get secret -n <ns>`.
- **`ImagePullBackOff` with `Unable to retrieve some image pull secrets` or
  `invalid username/password: unknown: Authentication is required`** → the pull secret didn't
  exist when the pod was scheduled. After fixing the secret, force an immediate retry rather
  than waiting for kubelet's backoff timer:
  `oc delete pod -n <ns> -l name=edrf-xclient-container-security`.
- **Pod `Running` but not reporting to ePO** — a separate, application-level layer. Always
  check `oc logs <pod> -n <ns>` on each pod. Common causes: a placeholder password left in
  `epo.password`, wrong `epo.user`, or `epo.ip` unreachable from the worker node's network
  (these pods use `hostNetwork: true`, so test connectivity from the node via
  `oc debug node/<node-name>`, not from the pod network). Also check whether the agent is
  sitting in ePO's pending/unmanaged list awaiting approval.
- **`helm` path errors** (`cannot find Chart.yaml`, `expected a gzipped archive`) → the chart
  argument must be the folder (`./edrf-xclient`), never `Chart.yaml` itself. Check the current
  directory (`pwd` / `Get-Location` / `cd` with no args) before assuming the path is wrong.
- **Shell-specific syntax**: the backtick `` ` `` line-continuation only works in PowerShell
  (use `^` in `cmd.exe`, `\` in bash). PowerShell's `curl` is aliased to `Invoke-WebRequest` —
  use `curl.exe` for real curl syntax.

## Step 4 — When troubleshooting

Get the exact command and exact output before diagnosing — don't infer from a description.
Work top-down through the sequence in Step 2 to find which step is actually unsatisfied.
