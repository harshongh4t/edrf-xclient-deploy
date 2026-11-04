---
name: edrf-xclient-deploy
description: Deploy or troubleshoot the Trellix eDRF XClient container-security agent on OpenShift using the edrf-xclient bundle (deploy.sh and extract-coninfo.sh). Use when the user wants to install, upgrade, verify, debug (image pull failures, InvalidImageName, pods restarting, not reporting to ePO), collect logs from, or uninstall the edrf-xclient DaemonSet, or asks about oc/helm/Git Bash setup, oc login or KUBECONFIG, cluster permissions, the ePO SmartInstaller and epo-coninfo secret, the Artifactory pull secret, image digest, registry vs airgap mode, or useful oc commands for this deployment.
---

# eDRF XClient — OpenShift Deployment

Guide the user through deploying the eDRF XClient agent (a DaemonSet, one pod per Linux node,
reporting to ePO) on OpenShift. Deployment uses a **bundle**
(`edrf-xclient-bundle-<version>.tar.gz`) containing:

- `image.tar`, the container image (used only in airgap mode)
- `edrf-xclient/`, the Helm chart
- **`extract-coninfo.sh`**, which turns the ePO Linux SmartInstaller (`TrellixSmartInstall.sh`)
  into `epo-coninfo.yml`, a Secret named **`epo-coninfo`** holding `coninfo.xml` (the agent's
  ePO connection file)
- **`deploy.sh`**, which checks that `epo-coninfo` exists and then runs
  `helm upgrade --install`, so the same command does a fresh install and an upgrade

Users don't run `helm install` by hand.

**Since build 50.99.0.150, ePO details come only from the `epo-coninfo` secret.** `deploy.sh`
no longer accepts `--epo-ip`, `--epo-user`, or `--epo-password` (it fails with
`ERROR: Unknown option`), and the chart has no ePO values.

`DEPLOYMENT_GUIDE.md` (at this repo's root, or
https://github.com/harshongh4t/edrf-xclient-deploy/blob/main/DEPLOYMENT_GUIDE.md) is the full
reference, with command blocks for PowerShell, `cmd.exe`, and Linux/Mac. This file holds the
operating logic: the required order, the exact commands, and how to diagnose failures.

## Rules for every response

- **Assume the user is doing this for the first time.** Don't assume `oc`, `helm`, or Git Bash
  are installed, or that the user is logged in. A "how do I deploy" request starts from
  installing tools. Don't skip steps that seem obvious.
- **Windows users must install Git for Windows.** Both `extract-coninfo.sh` and `deploy.sh`
  are bash scripts and don't run in `cmd.exe` or PowerShell. Say this explicitly in every
  Windows walkthrough, give the install command, and give the Git Bash form of each script
  command. Never tell a Windows user to type `./deploy.sh` directly.
- **Always include the `epo-coninfo` step** (Step 2.9) before `deploy.sh`. It's mandatory, and
  `deploy.sh` refuses to run without it.
- **Answer for both Windows and Linux jumpboxes.** Unless the user has said which one they use,
  give the command sequence for both a Windows jumpbox (`cmd.exe` + Git Bash) and a Linux
  jumpbox, as in Step 2.15. Shared `oc`/`helm` commands can be shown once and labeled
  "same on both".
- **Give exact runnable commands**, with placeholders like `<digest>` clearly marked, never a
  description of what to run in place of the command. **Warn that every `<...>` placeholder
  must be replaced.** A literal `sha256:<digest>` passes the dry-run but makes every pod fail
  with `InvalidImageName`.
- **Tell the user what to expect** after each important command (`**Expected:** ...`), so they
  can tell success from failure.
- **No emoji anywhere** in the output: no checkmarks, warning signs, or icons. Use plain bold
  labels such as `**Expected:**` and `**Note:**`.
- **Treat ePO and registry material as sensitive.** Never put real passwords, tokens,
  `TrellixSmartInstall.sh`, `epo-coninfo.yml`, or a filled-in `values.yaml` in files meant for
  git. Tell users to delete `epo-coninfo.yml` once applied.
- **Always end with the pointer to `DEPLOYMENT_GUIDE.md`** (Step 7).

## Step 1 — Establish machine and mode

Ask, or explain how to decide:

**Machine.** Does the jumpbox reach both the OpenShift API and Artifactory? Test with
`curl -v --max-time 10 https://artifactory.trellix.com` (PowerShell: `curl.exe`). A fast
response, even an error page, means yes. A hang means the digest lookup (Step 2.11) must run
on a separate machine with Artifactory access. Everything else stays on the jumpbox. The ePO
console is needed in a browser to download the SmartInstaller.

**Mode.**
- **Registry mode** (default): worker nodes pull the image from Artifactory. Needs a pull
  secret, image repository, and digest. Use this unless told otherwise.
- **Airgap mode**: the cluster can't reach any external registry. The user pushes `image.tar`
  to an internal registry themselves with `skopeo`, `docker`, or `podman`, then passes that
  registry's repo and the pushed digest to `deploy.sh --mode airgap`. No pull secret.
  Documented from the product README only, and not yet validated on a test cluster.

Both modes need the `epo-coninfo` secret.

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

**2.4 Airgap mode only: install `skopeo` (recommended), `podman`, or `docker`.**
**Verify:** `skopeo --version`, `podman version`, or `docker version` (Docker must show both
`Client` and `Server`).

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
**Tell Windows users to run the bundle scripts later from this same window**, so Git Bash
inherits the login and PATH.

**2.6 Check permissions:**
```
oc auth can-i create secrets -n <namespace>
oc auth can-i create rolebindings -n <namespace>
oc auth can-i create clusterroles
oc auth can-i create securitycontextconstraints
```
**Expected:** `yes` for all four. The SCC check also prints a "not namespace scoped" warning,
which is normal. Any `no` means a cluster-admin must grant access or run `deploy.sh` for them.
Stop here until that's resolved, because the chart creates a cluster-wide custom SCC and
ClusterRole.

**2.7 Get and extract the bundle** into a new, empty folder (`tar` is built into Windows
10/11):
```
mkdir C:\edrf-deploy\bundle-<version>
cd /d C:\edrf-deploy\bundle-<version>
tar -xzf C:\path\to\edrf-xclient-bundle-<version>.tar.gz
dir
```
(Linux/Mac: `mkdir -p ~/edrf-deploy/bundle-<version> && cd ~/edrf-deploy/bundle-<version>`,
then the same `tar`, then `ls -la`.)

**Expected:** `image.tar`, `edrf-xclient/`, `deploy.sh`, `extract-coninfo.sh`. If `tar` created
a subfolder, `cd` into it. **All later commands run from the folder containing `deploy.sh`.**
Editing `values.yaml` is **not** required.

**2.8 Create the namespace** (skip if it exists; check with `oc get ns <namespace>`):
```
oc new-project <namespace>
```
It must exist before the two secrets. `deploy.sh`'s `--create-namespace` runs too late.

**2.9 Create the `epo-coninfo` secret (mandatory, both modes):**
1. Create and download the **Linux** SmartInstaller, `TrellixSmartInstall.sh`, from the
   **ePO on-prem console** these nodes should report to. Give the user these exact steps:
   1. Select **Menu → Systems → System Tree**, then select **New Systems**.
   2. Select **Create URL for client-side agent download**.
   3. Enter a custom **URL name**.
   4. Select the **Agent version** for **Non-Windows (Linux)** operating systems from the
      list (not a Windows agent; `extract-coninfo.sh` only accepts the Linux SmartInstaller).
   5. Select **All Agent Handlers**.
   6. Click **OK**. ePO shows the **Agent Deployment URL**, which can be used to download the
      SmartInstaller or sent to whoever runs the deployment.
   7. Open that URL and download **`TrellixSmartInstall.sh`** (several MB).

   Copy it into the bundle folder. **Tell the user not to open or save it in a text editor**,
   because it has a binary payload that an editor can corrupt. If they lack ePO access,
   their ePO admin can generate the URL for them. Full steps: DEPLOYMENT_GUIDE.md Part B4.
2. Generate the manifest:
   - cmd.exe: `"C:\Program Files\Git\bin\bash.exe" extract-coninfo.sh TrellixSmartInstall.sh <namespace>`
   - PowerShell: `& "C:\Program Files\Git\bin\bash.exe" extract-coninfo.sh TrellixSmartInstall.sh <namespace>`
   - Linux/Mac: `bash extract-coninfo.sh TrellixSmartInstall.sh <namespace>`

   **Expected:** `[1/5]` through `[5/5]`, a `coninfo.xml size` of a few KB, then `Done.` and
   `Output file : epo-coninfo.yml`.
3. Apply and verify (same on both):
   ```
   oc apply -f epo-coninfo.yml
   oc get secret epo-coninfo -n <namespace>
   ```
   **Expected:** `secret/epo-coninfo created`, then one row `epo-coninfo   Opaque   1`. The
   script prints `kubectl apply`, and `oc apply` is equivalent. The secret name must stay
   `epo-coninfo`. Delete `epo-coninfo.yml` afterwards.

**2.10 Registry mode: create the pull secret** (once per namespace; when upgrading, check it
exists and reuse it):
```
oc create secret docker-registry artifactory-pull-secret --docker-server=artifactory.trellix.com --docker-username="<artifactory_user>" --docker-password="<reference_token>" -n <namespace>
oc get secret artifactory-pull-secret -n <namespace>
```
**Expected:** one row with type `kubernetes.io/dockerconfigjson`.

**2.11 Get the image digest. It changes with every build, so never reuse an old one.**
- **Registry mode:** in Artifactory, browse `edrf-xclient-container-security` →
  `trellix-edrf` and find the folder for this build. Then:
  ```
  curl -s -u "<artifactory_user>:<reference_token>" "https://artifactory.trellix.com/artifactory/api/storage/edrf-xclient-container-security/trellix-edrf/<build-folder>/manifest.json"
  ```
  (PowerShell: `curl.exe`.) The digest is `sha256:` plus the `checksums.sha256` value (64 hex
  characters). Without curl, click `manifest.json` in the Artifactory UI and copy its SHA-256.
  The repository is `artifactory.trellix.com/edrf-xclient-container-security/trellix-edrf`.
- **Airgap mode:** push `image.tar` and capture the digest (`DEPLOYMENT_GUIDE.md` Part H2).
  For the OpenShift internal registry: a cluster-admin enables the route once
  (`oc patch configs.imageregistry.operator.openshift.io/cluster --patch "{\"spec\":{\"defaultRoute\":true}}" --type=merge`),
  then the user gets its host with
  `oc get route default-route -n openshift-image-registry -o jsonpath="{.spec.host}"` and
  pushes there. With skopeo, for example:
  `skopeo copy --dest-creds "$(oc whoami):$(oc whoami -t)" --digestfile digest.txt docker-archive:image.tar docker://<route>/<namespace>/trellix-edrf:bundle`.
  `docker push` prints the digest itself, and podman supports `--digestfile`. **Pods pull
  through the internal service address, not the route**, so `--image-repo` must be
  `image-registry.openshift-image-registry.svc:5000/<namespace>/trellix-edrf`.

Also collect the cluster name:
`oc get infrastructure cluster -o jsonpath='{.status.infrastructureName}'` returns something
like `mycluster-ab1cd`, and the part before the random suffix (`mycluster`) is usually the name
to use.

**2.12 Dry-run (recommended).** From the bundle folder, as one line in any shell:
```
helm upgrade --install edrf-xclient ./edrf-xclient -n <namespace> --set image.repository="<image_repo>" --set image.digest="sha256:<digest>" --set imagePullSecret="artifactory-pull-secret" --set deploymentDetails.clusterName="<cluster_name>" --dry-run=client
```
(Airgap: internal-service repo, and drop `imagePullSecret`.) **Expected:** rendered
manifests, no `required` errors, `STATUS: pending-upgrade` and `Dry run complete` (normal,
nothing changes). Have the user confirm the DaemonSet has a `coninfo` volume with
`secretName: epo-coninfo`. **The dry-run doesn't validate the digest**, so have them check the
image line by appending `| findstr "image:"` (Windows) or `| grep "image:"` (Linux). It must
end in `@sha256:` plus 64 hex characters, with no `<` or `>`.

**2.13 Run `deploy.sh`.** It accepts only `--namespace`, `--cluster-name`, `--image-repo`,
`--image-digest`, `--image-pull-secret` (registry mode only), and `--mode`.

Registry mode, cmd.exe (one line):
```
"C:\Program Files\Git\bin\bash.exe" ./deploy.sh --namespace <namespace> --cluster-name <cluster_name> --image-repo <image_repo> --image-digest sha256:<digest> --image-pull-secret artifactory-pull-secret
```
PowerShell (one line):
```
& "C:\Program Files\Git\bin\bash.exe" ./deploy.sh --namespace <namespace> --cluster-name <cluster_name> --image-repo <image_repo> --image-digest sha256:<digest> --image-pull-secret artifactory-pull-secret
```
Linux/Mac or a Git Bash window:
```
chmod +x deploy.sh
./deploy.sh --namespace <namespace> --cluster-name <cluster_name> --image-repo <image_repo> --image-digest sha256:<digest> --image-pull-secret artifactory-pull-secret
```
Airgap mode: replace `--image-pull-secret ...` with `--mode airgap`, and use the
internal-service `--image-repo` and the pushed digest.

Call out these shell rules: keep the command on one line in `cmd.exe` and PowerShell (`\`
continuation is bash-only). On Linux, `chmod +x` avoids `Permission denied`, or use
`bash deploy.sh ...`.

**Expected output:** `==> Mode: registry (...)`, `==> Installing Helm chart`,
`STATUS: deployed`, `==> Deployment complete`.

**2.14 Verify.**
```
oc rollout status ds/edrf-xclient-container-security -n <namespace>
helm list -n <namespace>
oc get pods -n <namespace> -o wide
oc exec -n <namespace> <pod-name> -- ls -l /var/McAfee/agent/coninfo.xml
oc logs -n <namespace> <pod-name>
```
**Expected:** rollout `successfully rolled out`, release `deployed`, one pod per Linux worker
node, all `1/1 Running` with `RESTARTS 0`, `coninfo.xml` present in the pod, logs showing MA
provisioning against ePO and XClient starting with no repeating errors, and the nodes visible
in the ePO console. Tell the user:
- An upgrade rollout can take **20–25 minutes**. Each old pod gets up to **8 minutes** to stop
  gracefully, so `Terminating` for several minutes is normal. Don't force-delete before about
  10 minutes.
- Recheck `RESTARTS` after about an hour. A climbing count means the agent is crashing, even
  if the pod looks `Running`.

**2.15 Quick command sequence: Windows vs. Linux jumpbox.** When the user asks for "the
commands", "quick steps", or how to do it from Windows or Linux, give this sequence (after
confirming tools are installed and they're logged in).

Pre-checks (same on both):
```
oc whoami
oc get ns <namespace>
oc get secret artifactory-pull-secret -n <namespace>
oc auth can-i create secrets -n <namespace>
oc auth can-i create securitycontextconstraints
```

Extract the bundle, **Windows (`cmd.exe`):**
```
mkdir C:\edrf-deploy\bundle-<version>
move C:\path\to\edrf-xclient-bundle-<version>.tar.gz C:\edrf-deploy\bundle-<version>\
cd /d C:\edrf-deploy\bundle-<version>
tar -xzf edrf-xclient-bundle-<version>.tar.gz
dir
```
Extract the bundle, **Linux:**
```
mkdir -p ~/edrf-deploy/bundle-<version>
mv /path/to/edrf-xclient-bundle-<version>.tar.gz ~/edrf-deploy/bundle-<version>/
cd ~/edrf-deploy/bundle-<version>
tar -xzf edrf-xclient-bundle-<version>.tar.gz
ls -la
```

Create the `epo-coninfo` secret (with `TrellixSmartInstall.sh` copied into the folder).
Windows (`cmd.exe`):
```
"C:\Program Files\Git\bin\bash.exe" extract-coninfo.sh TrellixSmartInstall.sh <namespace>
```
Linux:
```
bash extract-coninfo.sh TrellixSmartInstall.sh <namespace>
```
Then, on both:
```
oc apply -f epo-coninfo.yml
oc get secret epo-coninfo -n <namespace>
```

Dry-run with the real digest (same on both; check that the image line has no placeholder):
```
helm upgrade --install edrf-xclient ./edrf-xclient -n <namespace> --set image.repository="<image_repo>" --set image.digest="sha256:<digest>" --set imagePullSecret="artifactory-pull-secret" --set deploymentDetails.clusterName="<cluster_name>" --dry-run=client
```

Install or upgrade, **Windows (`cmd.exe`, one line, needs Git for Windows):**
```
"C:\Program Files\Git\bin\bash.exe" ./deploy.sh --namespace <namespace> --cluster-name <cluster_name> --image-repo <image_repo> --image-digest sha256:<digest> --image-pull-secret artifactory-pull-secret
```
Install or upgrade, **Linux:**
```
chmod +x deploy.sh
./deploy.sh \
  --namespace <namespace> \
  --cluster-name <cluster_name> \
  --image-repo <image_repo> \
  --image-digest sha256:<digest> \
  --image-pull-secret artifactory-pull-secret
```

Verify (same on both):
```
helm list -n <namespace>
oc rollout status ds/edrf-xclient-container-security -n <namespace>
oc get pods -n <namespace> -o wide
oc exec -n <namespace> <pod-name> -- ls -l /var/McAfee/agent/coninfo.xml
```

Differences at a glance:

| Step | Windows (`cmd.exe`) | Linux |
|---|---|---|
| Tool prerequisite | Git for Windows | Nothing extra |
| Create folder | `mkdir C:\edrf-deploy\bundle-<version>` | `mkdir -p ~/edrf-deploy/bundle-<version>` |
| Move bundle | `move` | `mv` |
| Change folder | `cd /d C:\edrf-deploy\bundle-<version>` | `cd ~/edrf-deploy/bundle-<version>` |
| List files | `dir` | `ls -la` |
| Run `extract-coninfo.sh` | `"C:\Program Files\Git\bin\bash.exe" extract-coninfo.sh ...` | `bash extract-coninfo.sh ...` |
| Run `deploy.sh` | `"C:\Program Files\Git\bin\bash.exe" ./deploy.sh ...` (one line) | `chmod +x deploy.sh` then `./deploy.sh ...` |
| Filter output | `\| findstr "image:"` | `\| grep "image:"` |
| `oc` / `helm` / `tar` | Same | Same |

Never put the user's real ePO details, cluster name, or digest into repo files. Use
placeholders.

## Step 3 — Recognize these failure modes

**Running the scripts:**
- **`'.' is not recognized as an internal or external command`**: `./deploy.sh` was typed in
  `cmd.exe` or PowerShell. Use the Git Bash form, and install Git for Windows (2.3) if needed.
- **`'--namespace' is not recognized...`, one error per line**: a multi-line `\` command was
  pasted into `cmd.exe`. Use the one-line form.
- **`oc: command not found` / `helm: command not found` inside `deploy.sh`**: bash can't see
  the Windows tools. Usually plain `bash` (WSL) was used instead of Git Bash by full path, or
  `oc`/`helm` aren't on the Windows PATH.
- **`Unauthorized` / `You must be logged in` from `deploy.sh`**: the login expired, or Git Bash
  was opened separately and didn't inherit `KUBECONFIG`. Run `oc whoami` and launch `deploy.sh`
  from that same window.

**`extract-coninfo.sh`:**
- **`'bootstrap_size' not found` or `Marker '__ARCHIVE_FOLLOWS__' not found`**: not the Linux
  SmartInstaller, or it was corrupted (opened in an editor, or downloaded as an HTML error
  page). Download it again.
- **`Payload is empty`, `coninfo.xml is empty`, `not valid XML`**: truncated or corrupted
  download. Download it again.
- **`Required tools not found`**: not run with Git Bash. Use the full `bash.exe` path.
- **`Not enough disk space`**: needs about twice the SmartInstaller's size free.
- **`File not found`**: the SmartInstaller isn't in the current folder.

**`deploy.sh`:**
- **`ERROR: Secret 'epo-coninfo' not found in namespace '<ns>'`**: do 2.9 in the same
  namespace, then rerun.
- **`ERROR: Unknown option: --epo-ip`** (or `--epo-user`, `--epo-password`): an old command from
  build 144 or earlier. Remove all `--epo-*` flags.
- **`ERROR: --image-digest is required in registry mode`** (or another `--... is required`): a
  flag is missing or misspelled.
- **`forbidden` mentioning `securitycontextconstraints` or `clusterroles`**: missing
  cluster-wide permission (2.6). A cluster-admin must help.

**Pods:**
- **`InvalidImageName`**: the digest still has placeholder text or a typo. Get the real digest
  (2.11) and rerun `deploy.sh`.
- **`ImagePullBackOff` with `manifest unknown`**: the digest is from the wrong build.
- **`ImagePullBackOff` with `Authentication is required` / `Unable to retrieve some image
  pull secrets`**: fix the pull secret, then force a retry with
  `oc delete pod -n <namespace> -l name=edrf-xclient-container-security`.
- **Reference Token vs. Token ID**: Artifactory's token screen shows both. The long
  **Reference token** is the credential. The short UUID **Token ID** is only a label, and
  using it gives `401 Unauthorized` / `Authentication is required`.
- **Pull secret name vs. credential**: `--image-pull-secret` takes the secret's **name**
  (`artifactory-pull-secret`), never the token.
- **Never hand-build `config.json`** for the pull secret. The product README's
  `oc create secret generic --from-file=.dockerconfigjson=config.json` form needs a base64
  `auth` field that people often get wrong. Use `oc create secret docker-registry`.
- **`ImagePullBackOff` with a timeout or `no route to host`**: the nodes can't reach
  Artifactory. Use airgap mode or get network access.
- **`ContainerCreating` for a long time, with `secret "epo-coninfo" not found` in `oc
  describe pod`**: re-run `oc apply -f epo-coninfo.yml`, then delete the pod.
- **No pods created, or events say `unable to validate against any security context
  constraint`**: the SCC or RoleBinding is missing. Check `oc get scc edrf-xclient-<namespace>`
  and `oc get rolebinding -n <namespace>`, then re-run `deploy.sh` with enough permissions.
- **`RESTARTS` climbing on `Running` pods**: run `oc describe pod`. Under **Last State**,
  `OOMKilled` means it hit the 4Gi memory limit, and `Error` plus an exit code means the agent
  died. Get its last output with `oc logs <pod> --previous` and send both to the dev team.
- **Running but not in ePO**: check that `coninfo.xml` is in the pod and look at `oc logs` for
  provisioning errors. Confirm the SmartInstaller came from the right ePO server. Test ePO
  reachability from the node with `oc debug node/<node-name>` (pods use `hostNetwork: true`).
  The agent may also be waiting for approval in ePO.
- **`oc cp` error `one of src or dest must be a local file specification`**: the Windows
  destination started with a drive letter. `cd` to the folder and use `./<file>`.
- **Chart path errors** (`cannot find Chart.yaml`, `chart directory not found`): the user is in
  the wrong folder, or `deploy.sh` was separated from `edrf-xclient/`.

## Step 4 — Upgrade, rollback, uninstall

- **Upgrade:** extract the new bundle into a new folder, then get the **new** digest
  (registry) or push the new `image.tar` (airgap). Reuse the existing pull secret and
  `epo-coninfo` secret. Optionally dry-run, then run `deploy.sh` with the new
  `--image-digest`. Verify as in 2.14, expecting a 20–25 minute rollout. Recreate
  `epo-coninfo` only if switching ePO servers or the ePO certificates changed: redo 2.9
  (`oc apply` updates it), then run
  `oc rollout restart ds/edrf-xclient-container-security -n <namespace>`.
- **Upgrading from build 50.99.0.144 or earlier:** create `epo-coninfo` (2.9) **first**, and
  drop the `--epo-*` flags. The upgrade removes the old `epo-details` ConfigMap and Secret
  automatically.
- **Rollback:** `helm history edrf-xclient -n <namespace>`, then
  `helm rollback edrf-xclient <revision> -n <namespace>`.
- **Uninstall:** `helm uninstall edrf-xclient -n <namespace>` removes everything the chart
  created (DaemonSet, ServiceAccount, custom SCC, ClusterRole, RoleBinding). It does **not**
  remove the `epo-coninfo` secret, the pull secret, the namespace, or agent data persisted on
  nodes under `/mnt/XClientLogs/`. For a full cleanup, also run
  `oc delete secret epo-coninfo -n <namespace>`,
  `oc delete secret artifactory-pull-secret -n <namespace>`, and
  `oc delete namespace <namespace>`.

## Step 5 — Useful commands (when the user asks for a command list)

**Cluster and namespace:**
```
oc whoami
oc get nodes
oc get projects
oc get ns <namespace>
oc get secret epo-coninfo -n <namespace>
oc get secret artifactory-pull-secret -n <namespace>
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
oc rollout restart ds/edrf-xclient-container-security -n <namespace>
helm list -n <namespace>
helm status edrf-xclient -n <namespace>
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
ls -l /var/McAfee/agent/coninfo.xml
ls -lrth /tmp
ls -lrth /var/lib/Trellix/XClient/
ls -lrth /var/McAfee/agent/logs/
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
which shell the command was run in, because many first-time failures are shell syntax. Check
pasted commands for leftover `<...>` placeholders, a common and silent mistake.

## Step 7 — Close every response

End every install, upgrade, or troubleshooting answer with:

> For full per-shell command blocks (PowerShell / `cmd.exe` / Linux-Mac) and the complete
> troubleshooting table, see `DEPLOYMENT_GUIDE.md`
> (https://github.com/harshongh4t/edrf-xclient-deploy/blob/main/DEPLOYMENT_GUIDE.md).
