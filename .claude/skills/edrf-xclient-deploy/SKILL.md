---
name: edrf-xclient-deploy
description: Deploy or troubleshoot the Trellix eDRF XClient container-security agent on OpenShift using the edrf-xclient bundle and its deploy.sh script. Use when the user wants to install, upgrade, verify, debug (image pull failures, pods restarting, not reporting to ePO), collect logs from, or uninstall the edrf-xclient DaemonSet, or asks about oc/helm/Git Bash setup, oc login or KUBECONFIG, cluster permissions, the Artifactory pull secret, image digest, registry vs airgap mode, or useful oc commands for this deployment.
---

# eDRF XClient — OpenShift Deployment

Guide the user through deploying the eDRF XClient agent (a DaemonSet, one pod per Linux node,
reporting to ePO) on OpenShift. Deployment is done with a **bundle**
(`edrf-xclient-bundle-<version>.tar.gz`) containing `image.tar`, the `edrf-xclient/` Helm
chart, and **`deploy.sh`**. `deploy.sh` runs `helm upgrade --install` itself, so the same
command does a fresh install and an upgrade. Users don't run `helm install` by hand.

`DEPLOYMENT_GUIDE.md` (at this repo's root, or
https://github.com/harshongh4t/edrf-xclient-deploy/blob/main/DEPLOYMENT_GUIDE.md) is the full
reference, with command blocks for PowerShell, `cmd.exe`, and Linux/Mac. This file holds the
operating logic: the required order, the exact commands, and how to diagnose failures.

## Rules for every response

- **Assume the user is doing this for the first time.** Don't assume `oc`, `helm`, or Git Bash
  are installed, or that the user is logged in. A "how do I deploy" request starts from
  installing tools. Don't skip steps that seem obvious.
- **Windows users must install Git for Windows.** `deploy.sh` is a bash script and doesn't
  run in `cmd.exe` or PowerShell. Say this explicitly in every Windows walkthrough, give the
  install command, and give the Git Bash form of the `deploy.sh` command (Step 2.10). Never
  tell a Windows user to type `./deploy.sh` directly.
- **Ask which shell they use** (PowerShell, `cmd.exe`, or Linux/Mac) if it isn't clear, and
  give commands for that shell. If you can't tell, show all three for shell-specific steps.
- **Give exact runnable commands**, with placeholders like `<epo_ip>` clearly marked, never a
  description of what to run in place of the command.
- **Tell the user what to expect** after each important command (`**Expected:** ...`), so they
  can tell success from failure.
- **No emoji anywhere** in the output: no checkmarks, warning signs, or icons. Use plain bold
  labels such as `**Expected:**` and `**Note:**`.
- **Never put real passwords or tokens in files meant for git.** Remind users not to commit a
  `values.yaml` containing the ePO password.
- **Answer for both Windows and Linux jumpboxes.** Unless the user has said which one they use,
  give the full command sequence for both a Windows jumpbox (`cmd.exe` + Git Bash) and a Linux
  jumpbox, as in Step 2.12. Shared `oc`/`helm` commands can be shown once and labeled
  "same on both".
- **Always end with the pointer to `DEPLOYMENT_GUIDE.md`** (Step 7).

## Step 1 — Establish machine and mode

Ask, or explain how to decide:

**Machine.** Does the jumpbox reach both the OpenShift API and Artifactory? Test with
`curl -v --max-time 10 https://artifactory.trellix.com` (PowerShell: `curl.exe`). A fast
response, even an error page, means yes. A hang means the digest lookup (Step 2.8) must run on
a separate machine with Artifactory access. Everything else stays on the jumpbox.

**Mode.**
- **Registry mode** (default): worker nodes pull the image from Artifactory. Needs a pull
  secret, image repository, and digest. Use this unless told otherwise.
- **Airgap mode**: the cluster can't reach any external registry. Needs Docker on the jumpbox
  and a one-time cluster-admin command to expose the internal registry route. `deploy.sh`
  pushes `image.tar` into the cluster itself, so no pull secret, repo, or digest is needed.

## Step 2 — The required sequence

Most failures come from steps run out of order. Walk the user through these in order and have
them verify each one before moving on.

**2.1 Install `oc`.** Download it from the OpenShift web console (**?** icon → **Command Line
Tools**) or https://mirror.openshift.com/pub/openshift-v4/clients/oc/latest/, extract it, put
it in a permanent folder (e.g. `C:\oc`), and add that folder to PATH. Windows, permanent:
`[Environment]::SetEnvironmentVariable("Path", "$env:Path;C:\oc", "User")` in PowerShell.
Open a new terminal. **Verify:** `oc version --client` prints `Client Version: 4.x`. Full
per-shell steps: `DEPLOYMENT_GUIDE.md` Part A1.

**2.2 Install `helm` (v3).** Windows: `winget install Helm.Helm`. Mac: `brew install helm`.
Linux: the official `get-helm-3` script. Open a new terminal. **Verify:** `helm version` shows
`v3.x`.

**2.3 Windows only: install Git for Windows (provides Git Bash).**
```
winget install --id Git.Git -e
```
Or use the installer from https://git-scm.com/download/win with default options, or request it
from IT if installs are locked down. Open a new terminal. **Verify:**
```
"C:\Program Files\Git\bin\bash.exe" --version
```
(PowerShell: prefix with `&`.) **Expected:** `GNU bash, version 5.x`. If the path doesn't
exist, find Git with `where.exe git`. `bash.exe` is in the `bin` folder beside the `cmd`
folder holding `git.exe`. **Warn the user not to use plain `bash`**: on Windows that usually
starts WSL, which can't see the Windows `oc.exe`/`helm.exe` or the cluster login.

**2.4 Airgap mode only: install Docker** and make sure the daemon is running.
**Verify:** `docker version` shows both `Client` and `Server`.

**2.5 Log in to the cluster. Give both options:**
- **Token login:** web console → username (top-right) → **Copy login command** →
  **Display Token**, then run
  `oc login --token=sha256~xxxxxxxxxxxx --server=https://api.<your-cluster>:6443`.
  Tokens are short-lived, so an auth error later means logging in again.
- **Kubeconfig file:** set `KUBECONFIG` to the file path:
  - PowerShell: `$env:KUBECONFIG = "C:\path\to\kubeconfig"`
  - cmd.exe: `set KUBECONFIG=C:\path\to\kubeconfig`
  - Linux/Mac: `export KUBECONFIG=/path/to/kubeconfig`

  This only lasts for that window. To make it permanent on Windows:
  `[Environment]::SetEnvironmentVariable("KUBECONFIG", "C:\path\to\kubeconfig", "User")`.

**Verify:** `oc whoami` prints an identity, and `oc get nodes` shows every node `Ready`.
**Tell Windows users to run `deploy.sh` later from this same window**, so Git Bash inherits
the login and PATH.

**2.6 Check permissions.** The chart creates a **custom SCC and a ClusterRole**, which are
cluster-wide, as well as namespaced resources:
```
oc auth can-i create rolebindings -n <namespace>
oc auth can-i create clusterroles
oc auth can-i create securitycontextconstraints
```
**Expected:** `yes` for all three. Any `no` means a cluster-admin must grant access or run
`deploy.sh` for them. Stop here until that's resolved. Otherwise the install fails with
`forbidden`.

**2.7 Get and extract the bundle** into a new, empty folder (`tar` is built into Windows
10/11):
```
mkdir C:\edrf-deploy\bundle
cd C:\edrf-deploy\bundle
tar -xzf C:\path\to\edrf-xclient-bundle-<version>.tar.gz
```
(Linux/Mac: `mkdir -p ~/edrf-deploy/bundle && cd ~/edrf-deploy/bundle`, then the same `tar`.
In `cmd.exe`, use `cd /d` if changing drives.) **Expected:** `image.tar`, `edrf-xclient/`,
`deploy.sh`. Editing `values.yaml` is **not** required, because `deploy.sh` flags supply
everything.

**2.8 Registry mode: get the image repository and digest** for the bundle's build:
```
curl -s -u "<artifactory_user>:<reference_token>" "https://artifactory.trellix.com/artifactory/api/storage/edrf-xclient-container-security/trellix-edrf/<build>/manifest.json"
```
(PowerShell: `curl.exe`.) The digest is `sha256:` plus the `checksums.sha256` value. The
repository is `artifactory.trellix.com/edrf-xclient-container-security/trellix-edrf`. For
fallbacks and HTTP error meanings, see `DEPLOYMENT_GUIDE.md` Part C. Also collect the ePO
IP, user, and password, and the cluster name:
`oc get infrastructure cluster -o jsonpath='{.status.infrastructureName}'` returns something
like `mycluster-ab1cd`, and the part before the random suffix (`mycluster`) is usually the name
to use.

**2.9 Registry mode: create the namespace, then the pull secret, in that order:**
```
oc new-project <namespace>
oc create secret docker-registry artifactory-pull-secret --docker-server=artifactory.trellix.com --docker-username="<artifactory_user>" --docker-password="<reference_token>" -n <namespace>
oc get secret artifactory-pull-secret -n <namespace>
```
**Expected:** one row with type `kubernetes.io/dockerconfigjson`. If upgrading, the secret
probably exists already, so check first and reuse it.

**2.10 Dry-run (recommended), then run `deploy.sh`.** Dry-run from the bundle folder, as one
line in any shell:
```
helm upgrade --install edrf-xclient ./edrf-xclient -n <namespace> --set epo.ip="<epo_ip>" --set epo.user="<epo_user>" --set epo.password="<epo_password>" --set deploymentDetails.clusterName="<cluster_name>" --set image.repository="<image_repo>" --set image.digest="sha256:<digest>" --set imagePullSecret="artifactory-pull-secret" --dry-run=client
```
**Expected:** rendered manifests with no `required` errors. `STATUS: pending-upgrade` and
`Dry run complete` are normal, and nothing changes on the cluster.

Then run `deploy.sh`, in the form for the user's shell:

cmd.exe (one line):
```
"C:\Program Files\Git\bin\bash.exe" ./deploy.sh --namespace <namespace> --epo-ip <epo_ip> --epo-user <epo_user> --epo-password "<epo_password>" --cluster-name <cluster_name> --image-repo <image_repo> --image-digest sha256:<digest> --image-pull-secret artifactory-pull-secret
```
PowerShell (one line):
```
& "C:\Program Files\Git\bin\bash.exe" ./deploy.sh --namespace <namespace> --epo-ip <epo_ip> --epo-user <epo_user> --epo-password "<epo_password>" --cluster-name <cluster_name> --image-repo <image_repo> --image-digest sha256:<digest> --image-pull-secret artifactory-pull-secret
```
Linux/Mac or a Git Bash window:
```
chmod +x deploy.sh
./deploy.sh --namespace <namespace> --epo-ip <epo_ip> --epo-user <epo_user> --epo-password '<epo_password>' --cluster-name <cluster_name> --image-repo <image_repo> --image-digest sha256:<digest> --image-pull-secret artifactory-pull-secret
```
Airgap mode: drop `--image-repo`, `--image-digest`, and `--image-pull-secret`, and add
`--mode airgap`. Before the first airgap run, a cluster-admin must run
`oc patch configs.imageregistry.operator.openshift.io/cluster --patch "{\"spec\":{\"defaultRoute\":true}}" --type=merge`.

Call out these shell rules: keep the command on one line in `cmd.exe` and PowerShell (`\`
continuation is bash-only). Use double quotes around the password in `cmd.exe` and
PowerShell, and single quotes in bash. `--help` lists the script's flags.

**Expected output:** `==> Mode: registry (...)`, `==> Installing Helm chart`,
`STATUS: deployed`, `==> Deployment complete`.

**2.11 Verify.**
```
oc rollout status ds/edrf-xclient-container-security -n <namespace>
helm list -n <namespace>
oc get pods -n <namespace> -o wide
oc logs -n <namespace> <pod-name>
```
**Expected:** rollout `successfully rolled out`, release `deployed`, one pod per Linux worker
node, all `1/1 Running` with `RESTARTS 0`, logs showing MA provisioning against ePO and XClient
starting with no repeating errors, and the nodes visible in the ePO console. Tell the user:
- An upgrade rollout can take **20–25 minutes**. Each old pod gets up to **8 minutes** to stop
  gracefully, so `Terminating` for several minutes is normal. Don't force-delete before about
  10 minutes.
- Recheck `RESTARTS` after about an hour. A climbing count means the agent is crashing, even
  if the pod looks `Running`.

**2.12 Quick command sequence: Windows vs. Linux jumpbox.** When the user asks for "the
commands", "quick steps", or how to do it from Windows or Linux, give this sequence (after
confirming tools are installed and they're logged in). This is the sequence used for the
50.99.0.144 upgrade.

Pre-checks (same on both):
```
oc whoami
oc get secret artifactory-pull-secret -n <namespace>
oc auth can-i create securitycontextconstraints
oc auth can-i create clusterroles
oc auth can-i create rolebindings -n <namespace>
```

Extract the bundle, **Windows (`cmd.exe`):**
```
mkdir C:\edrf-deploy\bundle
move C:\path\to\edrf-xclient-bundle-<version>.tar.gz C:\edrf-deploy\bundle\
cd /d C:\edrf-deploy\bundle
tar -xzf edrf-xclient-bundle-<version>.tar.gz
dir
```
Extract the bundle, **Linux:**
```
mkdir -p ~/edrf-deploy/bundle
mv /path/to/edrf-xclient-bundle-<version>.tar.gz ~/edrf-deploy/bundle/
cd ~/edrf-deploy/bundle
tar -xzf edrf-xclient-bundle-<version>.tar.gz
ls -la
```

Update `edrf-xclient/values.yaml` (Windows: `notepad edrf-xclient\values.yaml`; Linux:
`vi edrf-xclient/values.yaml` or `nano`), then dry-run (same on both):
```
helm upgrade --install edrf-xclient ./edrf-xclient -n <namespace> --dry-run=client
```

Install or upgrade, **Windows (`cmd.exe`, one line, needs Git for Windows):**
```
"C:\Program Files\Git\bin\bash.exe" ./deploy.sh --namespace <namespace> --epo-ip <epo_ip> --epo-user <epo_user> --epo-password "<epo_password>" --cluster-name <cluster_name> --image-repo <image_repo> --image-digest sha256:<digest> --image-pull-secret artifactory-pull-secret
```
Install or upgrade, **Linux:**
```
chmod +x deploy.sh
./deploy.sh \
  --namespace <namespace> \
  --epo-ip <epo_ip> \
  --epo-user <epo_user> \
  --epo-password '<epo_password>' \
  --cluster-name <cluster_name> \
  --image-repo <image_repo> \
  --image-digest sha256:<digest> \
  --image-pull-secret artifactory-pull-secret
```
(On Linux, `chmod +x` avoids `Permission denied`, or use `bash deploy.sh ...`.)

Verify (same on both):
```
helm list -n <namespace>
oc rollout status ds/edrf-xclient-container-security -n <namespace>
oc get pods -n <namespace> -o wide -w
```

Differences at a glance:

| Step | Windows (`cmd.exe`) | Linux |
|---|---|---|
| Tool prerequisite | Git for Windows | Nothing extra |
| Create folder | `mkdir C:\edrf-deploy\bundle` | `mkdir -p ~/edrf-deploy/bundle` |
| Move bundle | `move` | `mv` |
| Change folder | `cd /d C:\edrf-deploy\bundle` | `cd ~/edrf-deploy/bundle` |
| List files | `dir` | `ls -la` |
| Run script | `"C:\Program Files\Git\bin\bash.exe" ./deploy.sh ...` (one line) | `chmod +x deploy.sh` then `./deploy.sh ...` |
| Password quoting | `"..."` | `'...'` |
| `oc` / `helm` / `tar` | Same | Same |

Never put the user's real ePO IP, password, cluster name, or digest into repo files. Use
placeholders.

## Step 3 — Recognize these failure modes

- **`'.' is not recognized as an internal or external command`**: `./deploy.sh` was typed in
  `cmd.exe` or PowerShell. Use the Git Bash form (2.10), and install Git for Windows (2.3) if
  needed.
- **`'--namespace' is not recognized...`, one error per line**: a multi-line `\` command was
  pasted into `cmd.exe`. Use the one-line form.
- **`oc: command not found` / `helm: command not found` inside `deploy.sh`**: bash can't see
  the Windows tools. Usually plain `bash` (WSL) was used instead of Git Bash by full path, or
  `oc`/`helm` aren't on the Windows PATH.
- **`Unauthorized` / `You must be logged in` from `deploy.sh`**: the login expired, or Git Bash
  was opened separately and didn't inherit `KUBECONFIG`. Run `oc whoami` and launch `deploy.sh`
  from that same window.
- **`forbidden` mentioning `securitycontextconstraints` or `clusterroles`**: missing
  cluster-wide permission (2.6). A cluster-admin must help.
- **Reference Token vs. Token ID**: Artifactory's token screen shows both. The long
  **Reference token** is the credential. The short UUID **Token ID** is only a label, and
  using it gives `401 Unauthorized` / `Authentication is required`.
- **Pull secret name vs. credential**: `--image-pull-secret` takes the secret's **name**
  (`artifactory-pull-secret`), never the token.
- **Never hand-build `config.json`** for the pull secret. The product README's
  `oc create secret generic --from-file=.dockerconfigjson=config.json` form needs a base64
  `auth` field that people often get wrong. Use `oc create secret docker-registry`.
- **`namespace not found` on secret creation**: create the namespace first, then re-run the
  secret command and verify it.
- **`ImagePullBackOff` with `Authentication is required` / `Unable to retrieve some image
  pull secrets`**: fix the secret, then force a retry with
  `oc delete pod -n <namespace> -l name=edrf-xclient-container-security`.
- **`ImagePullBackOff` with a timeout or `no route to host`**: the nodes can't reach
  Artifactory. Use airgap mode or get network access.
- **No pods created, or events say `unable to validate against any security context
  constraint`**: the SCC or RoleBinding is missing. Check `oc get scc edrf-xclient-<namespace>`
  and `oc get rolebinding -n <namespace>`, then re-run `deploy.sh` with enough permissions.
- **`RESTARTS` climbing on `Running` pods**: run `oc describe pod`. Under **Last State**,
  `OOMKilled` means it hit the 4Gi memory limit, and `Error` plus an exit code means the agent
  died. Get its last output with `oc logs <pod> --previous` and send both to the dev team.
- **Running but not in ePO / `CrashLoopBackOff` with ePO errors**: wrong ePO IP, user, or
  password, ePO unreachable from the node (pods use `hostNetwork: true`, so test from the node
  with `oc debug node/<node-name>`), or the agent is waiting for approval in ePO.
- **`oc cp` error `one of src or dest must be a local file specification`**: the Windows
  destination started with a drive letter. `cd` to the folder and use `./<file>`.
- **Chart path errors** (`cannot find Chart.yaml`): the user is in the wrong folder. Commands
  run from the bundle folder that contains `edrf-xclient/` and `deploy.sh`.

## Step 4 — Upgrade, rollback, uninstall

- **Upgrade:** extract the new bundle into a new folder, get the new digest (registry mode),
  reuse the existing pull secret, optionally dry-run, then run the same `deploy.sh` command
  with the new `--image-digest`. Verify as in 2.11, expecting a 20–25 minute rollout.
- **Rollback:** `helm history edrf-xclient -n <namespace>`, then
  `helm rollback edrf-xclient <revision> -n <namespace>`.
- **Uninstall:** `helm uninstall edrf-xclient -n <namespace>` removes everything the chart
  created, including the custom SCC and ClusterRole. It does **not** remove the namespace, the
  manually created pull secret, or agent data persisted on nodes under `/mnt/XClientLogs/`.
  For a full cleanup, also run
  `oc delete secret artifactory-pull-secret -n <namespace>` and
  `oc delete namespace <namespace>`.

## Step 5 — Useful commands (when the user asks for a command list)

**Cluster and namespace:**
```
oc whoami
oc get nodes
oc get projects
oc get ns <namespace>
```

**Pods and release:**
```
oc get pods -n <namespace>
oc get pods -n <namespace> -o wide
oc get pods -n <namespace> -o wide -w          # watch mode; Ctrl+C to stop
oc describe pod -n <namespace> <pod-name>
oc logs -n <namespace> <pod-name>
oc logs -n <namespace> <pod-name> --previous   # logs from before the last restart
oc rollout status ds/edrf-xclient-container-security -n <namespace>
helm list -n <namespace>
helm history edrf-xclient -n <namespace>
```

**Shell into a pod and check the agent:**
```
oc exec -it -n <namespace> <pod-name> -- /bin/bash
oc exec -it -n <namespace> <pod-name> -- /bin/sh      # if bash isn't in the image
```
Inside the pod:
```
ps -o pid,ppid,cmd -C xclient
ls -lrth /tmp
ls -lrth /var/lib/Trellix/XClient/
```

**Copy a log file out of a pod** (relative destination, especially on Windows):
```
oc cp -n <namespace> <pod-name>:/tmp/<log-file> ./<log-file>
```

**Persisted agent data on a node** (survives pod restarts):
```
oc debug node/<node-name>
chroot /host
ls -lrth /mnt/XClientLogs/
```

## Step 6 — When troubleshooting

Get the exact command and the exact output before diagnosing, rather than working from a
description. Work top-down through Step 2 to find the first step that isn't satisfied. Ask
which shell the command was run in, because many first-time failures are shell syntax (Step 3,
first four items).

## Step 7 — Close every response

End every install, upgrade, or troubleshooting answer with:

> For full per-shell command blocks (PowerShell / `cmd.exe` / Linux-Mac) and the complete
> troubleshooting table, see `DEPLOYMENT_GUIDE.md`
> (https://github.com/harshongh4t/edrf-xclient-deploy/blob/main/DEPLOYMENT_GUIDE.md).
