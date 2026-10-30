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

**Rules for every response this skill produces:**
- **Never assume `oc` and `helm` are already installed.** A "steps to deploy" request means
  starting from zero — include installation (Part A: download, extract, add to PATH, verify)
  as the first real step, not a one-line "confirm it's on PATH" aside. Don't skip steps
  because they seem obvious to an experienced engineer; this skill is used by first-time
  deployers.
- **Always point to the full guide for exact commands and edge cases**, rather than
  summarizing from memory or guessing at a flag: `DEPLOYMENT_GUIDE.md` in this repository, or
  https://github.com/harshongh4t/edrf-xclient-deploy/blob/main/DEPLOYMENT_GUIDE.md if working
  outside the cloned repo. Say this explicitly in the response, not just internally — end any
  deployment walkthrough with a line telling the user where to go for exact per-shell command
  blocks and the full troubleshooting table.
- **No emoji anywhere in the output** — no checkmarks, warning signs, or decorative icons in
  headers or callouts. Use plain bold text instead (e.g. `**Expected:**`, `**Note:**`),
  matching `DEPLOYMENT_GUIDE.md`'s style.
- Give exact runnable commands, not paraphrased descriptions of what a command does instead of
  the command itself.

## Step 1 — Establish the topology

Ask whether the user is working from:
- A **jumpbox** with cluster access but possibly no internet egress to Artifactory — image
  digest lookups (Part C) must then run from a separate, Artifactory-reachable machine.
- A **single machine** with both cluster and internet access — no split required.

Diagnostic: a `curl`/`curl.exe` request to `artifactory.trellix.com` that **hangs** indicates
a network/firewall split, not an auth problem. An **instant error code** (`401`/`403`/`404`)
means stay on the same machine and fix credentials or the URL.

## Step 2 — Enforce the required sequence (install)

Most failures in this deployment come from steps run out of order, not from missing
information. Hold the user to this sequence:

1. **Install `oc` and `helm` if they aren't already present** — don't skip straight to login
   commands. For `oc`: download from the cluster's web console (**?** icon → Command Line
   Tools) or https://mirror.openshift.com/pub/openshift-v4/clients/oc/latest/, extract it, and
   add it to PATH. For `helm`: `winget install Helm.Helm` (Windows), `brew install helm`
   (Mac), or the official install script (Linux). Verify with `oc version --client` and
   `helm version` (expect Helm 3). Give the user the exact OS-appropriate commands — see
   `DEPLOYMENT_GUIDE.md` Part A for the complete PowerShell/`cmd.exe`/Linux-Mac command blocks
   rather than reconstructing them from memory.
2. Authenticate to the cluster (token login or `KUBECONFIG`); confirm with `oc whoami`.
3. Check early whether the user's account can create RoleBindings
   (`oc auth can-i create rolebindings -n <ns>`) — this chart requires binding to
   `system:openshift:scc:privileged`, which is usually admin-only. A `no` here means looping
   in a cluster-admin before going further, not a problem to solve later.
4. **Create the namespace before creating any secret inside it.** `oc create secret` fails
   with `namespace not found` otherwise — and after fixing the namespace, it's easy to forget
   to re-run the secret command.
5. Generate the Artifactory identity token and create the pull secret with
   `oc create secret docker-registry`. Verify it exists
   (`oc get secret artifactory-pull-secret -n <ns>`) before moving on — never assume a
   command succeeded without checking.
6. Resolve the image digest and populate `values.yaml`: `image.repository`, `image.digest`,
   `imagePullSecret` (the secret's name, not its contents), `epo.ip`/`epo.user`/`epo.password`
   (or `epo.existingSecret`), and `deploymentDetails.clusterName`. For the cluster name,
   OpenShift node names follow `<cluster-name>-<random-infra-id>-<role>-<index>` —
   `oc get infrastructure cluster -o jsonpath='{.status.infrastructureName}'` confirms it, and
   the segment before the random suffix is usually the value to use (confirm against ePO's
   existing naming convention if other clusters already report in).
7. `helm upgrade --install ... --dry-run=client` first; resolve every `required` error before
   a real install.
8. Real install (same command, `--dry-run=client` removed).
9. Link the pull secret to the service account:
   `oc secrets link edrf-admin artifactory-pull-secret --for=pull -n <ns>`. This only
   succeeds after the install has created `edrf-admin`.
10. Verify pods, then check pod logs for actual ePO connectivity — `Running` only confirms the
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
- **Pod stuck `Pending`** (not `ImagePullBackOff`) → the ServiceAccount likely never got the
  privileged SCC RoleBinding, usually because the install ran without sufficient permissions.
  Confirm with `oc get rolebinding -n <ns>`; if it's missing, this is Step 2.3's permission
  check coming back to bite — get an admin to grant access and re-run the install.
- **`CrashLoopBackOff` with ePO connection errors in the logs** → treat the same as "Running
  but not reporting": wrong ePO IP/user/password, or the cluster/node can't reach the ePO
  server at all. Escalate network reachability to the platform/network team if credentials
  check out.
- **Shell-specific syntax**: the backtick `` ` `` line-continuation only works in PowerShell
  (use `^` in `cmd.exe`, `\` in bash). PowerShell's `curl` is aliased to `Invoke-WebRequest` —
  use `curl.exe` for real curl syntax.

## Step 4 — Upgrade, rollback, and uninstall

- **Upgrade to a new build:** only the image digest needs refreshing — repeat the digest
  lookup from Step 2.6, then
  `helm upgrade edrf-xclient ./edrf-xclient -n <ns> --reuse-values --set image.digest="sha256:<new-digest>"`.
  `--reuse-values` keeps every other setting (ePO config, pull secret, cluster name) as-is;
  don't ask the user to re-supply values that haven't changed.
- **Roll back a bad upgrade:** `helm history edrf-xclient -n <ns>` to find the last known-good
  revision, then `helm rollback edrf-xclient <revision-number> -n <ns>`.
- **Uninstall:** `helm uninstall edrf-xclient -n <ns>` removes only what Helm created
  (DaemonSet, ServiceAccount, RoleBinding, ConfigMap, Secret). It does **not** remove the
  namespace or anything created manually outside Helm — the Artifactory pull secret and any
  `epo.existingSecret`. If the user wants a full teardown, delete those explicitly
  (`oc delete secret artifactory-pull-secret -n <ns>`, `oc delete namespace <ns>`, etc.) —
  don't assume `helm uninstall` alone leaves nothing behind.

## Step 5 — When troubleshooting

Get the exact command and exact output before diagnosing — don't infer from a description.
Work top-down through the sequence in Step 2 to find which step is actually unsatisfied.

## Step 6 — Closing every response

Whether answering an install, upgrade, or troubleshooting question, end with a line like:

> For exact command blocks in your shell (PowerShell/`cmd.exe`/Linux-Mac) and the full
> troubleshooting table, see `DEPLOYMENT_GUIDE.md`
> (https://github.com/harshongh4t/edrf-xclient-deploy/blob/main/DEPLOYMENT_GUIDE.md).

This is not optional filler — it's the fallback for anything this skill's condensed logic
doesn't cover.
