# eDRF XClient — OpenShift Deployment Guide

**Audience:** engineers deploying eDRF XClient for the first time. This guide doesn't assume
`oc`, `helm`, Git Bash, or Artifactory access are already set up. Follow the parts in order,
because later steps depend on resources that earlier steps create.

**How deployment works:** you receive a **bundle** (`edrf-xclient-bundle-<version>.tar.gz`)
containing the container image, the Helm chart, and two scripts:

- **`extract-coninfo.sh`** reads the ePO SmartInstaller (`TrellixSmartInstall.sh`) and produces
  a Kubernetes Secret manifest, `epo-coninfo.yml`. That secret holds `coninfo.xml`, the file
  the agent uses to register with ePO. **You must create this secret before deploying.**
- **`deploy.sh`** does the install. It checks that the `epo-coninfo` secret exists, then runs
  `helm upgrade --install` for you, so the same command handles both a fresh install and an
  upgrade of an existing release. Don't run `helm install` yourself.

> **Changed in build 50.99.0.150:** the agent no longer takes an ePO IP, user, and password.
> `deploy.sh` no longer accepts `--epo-ip`, `--epo-user`, or `--epo-password` (passing them
> fails with `ERROR: Unknown option`). ePO connection details now come only from the
> `epo-coninfo` secret (Part E). If you're upgrading from an older build, see Part L.

This guide covers three shells: **PowerShell**, **Command Prompt (`cmd.exe`)**, and
**Linux/Mac (bash/zsh)**.

- Every `oc` and `helm` command is plain text that runs the same in all three shells, so it
  appears once with no shell label.
- A few steps really do differ by shell (setting an environment variable, extracting an
  archive, editing PATH, running the bundle's scripts). These appear as labeled blocks:
  **PowerShell**, **cmd.exe**, **Linux/Mac**. Use the one that matches your terminal.
- **`deploy.sh` and `extract-coninfo.sh` are bash scripts.** They don't run directly in
  `cmd.exe` or PowerShell. On Windows you need **Git for Windows**, which provides Git Bash
  (Part A3). Linux and Mac already have bash.
- PowerShell aliases `curl` to `Invoke-WebRequest`, so use `curl.exe` wherever `curl` appears
  below in PowerShell.

**Read Part 0 first.** It determines which machine you run the rest of this guide from, and
which deployment mode to use.

> **Namespace note:** the examples below use `edrf-tests` as a placeholder. This name is
> expected to change in a future release. Use whatever namespace your team has standardized on.

> **Credentials note:** never commit real passwords, tokens, `TrellixSmartInstall.sh`,
> `epo-coninfo.yml`, or a `values.yaml` containing real values to any git repository.

---

## Quick glossary

| Term | What it means here |
|---|---|
| **OpenShift / `oc`** | Red Hat's Kubernetes distribution. `oc` is its CLI (a superset of `kubectl`). |
| **Project / Namespace** | An isolated area in the cluster. "Project" is OpenShift's name for a Kubernetes Namespace. |
| **Helm / chart / release** | Helm is a package manager for Kubernetes. A **chart** is the package (the `edrf-xclient/` folder in the bundle). A **release** is one installed instance of that chart in a namespace (named `edrf-xclient`). |
| **Bundle** | `edrf-xclient-bundle-<version>.tar.gz`, the deliverable containing `image.tar`, the `edrf-xclient/` chart, `deploy.sh`, and `extract-coninfo.sh`. |
| **`deploy.sh`** | The bash script in the bundle that performs the install or upgrade. |
| **`extract-coninfo.sh`** | The bash script in the bundle that turns the ePO SmartInstaller into the `epo-coninfo` secret manifest. |
| **SmartInstaller** | `TrellixSmartInstall.sh`, the Linux agent installer downloaded from the ePO console. It contains `coninfo.xml`. |
| **`coninfo.xml` / `epo-coninfo`** | `coninfo.xml` holds everything the agent needs to register with ePO (server address, certificates, keys). `epo-coninfo` is the Kubernetes Secret that stores it. The name is fixed, because the chart looks for exactly `epo-coninfo`. |
| **Git Bash** | A bash shell for Windows, installed with Git for Windows. Required on Windows to run the bundle's scripts. |
| **DaemonSet** | A workload that runs exactly one pod on every node. This is how the chart deploys the XClient agent. |
| **Secret** | A Kubernetes object for sensitive values. |
| **ServiceAccount (SA)** | The identity a pod runs as (`edrf-admin` here). |
| **SCC** | Security Context Constraint, OpenShift's permission gate for pods that need host access. This chart **creates its own reduced-privilege SCC** (`edrf-xclient-<namespace>`) instead of using the built-in `privileged` SCC. Creating an SCC needs cluster-level permission (Part B2). |
| **Artifactory** | Trellix's private container registry (`artifactory.trellix.com`), where the agent image is hosted. |
| **ePO** | Trellix ePolicy Orchestrator, the management server the XClient agent reports to. |

---

## Quick start — Full command sequence (Windows or Linux jumpbox)

This is the complete install/upgrade sequence on one page, for when the tools are installed
and you're logged in. It's the sequence used to deploy build 50.99.0.150 in registry mode. If
anything here is unfamiliar or fails, follow the detailed parts below (Parts A–I). For a
Linux jumpbox, **Appendix E** has the same sequence as a single copy-paste block with
variables set once at the top.

Replace every `<...>` placeholder with your own values. **Never run a command with a `<...>`
placeholder still in it.** For example, a literal `sha256:<digest>` makes every pod fail with
`InvalidImageName`.

| | Windows jumpbox | Linux jumpbox |
|---|---|---|
| Extra tool needed | **Git for Windows** (Part A3), because the bundle's scripts are bash | None, because bash is built in |
| Shell used below | `cmd.exe` | bash |

### Step 1 — Pre-checks

The same commands on both:
```
oc whoami
oc get ns <namespace>
oc get secret artifactory-pull-secret -n <namespace>
oc auth can-i create secrets -n <namespace>
oc auth can-i create securitycontextconstraints
oc auth can-i create clusterroles
oc auth can-i create rolebindings -n <namespace>
```
**Expected:** your identity, the namespace `Active`, the pull secret listed, and `yes` four
times. If the namespace or pull secret doesn't exist, create them first (Parts D and F).
`oc auth can-i create securitycontextconstraints` also prints a "not namespace scoped"
warning, which is normal.

### Step 2 — Create a folder and extract the bundle

**Windows (`cmd.exe`):**
```cmd
mkdir C:\edrf-deploy\bundle-<version>
move C:\path\to\edrf-xclient-bundle-<version>.tar.gz C:\edrf-deploy\bundle-<version>\
cd /d C:\edrf-deploy\bundle-<version>
tar -xzf edrf-xclient-bundle-<version>.tar.gz
dir
```
**Linux:**
```bash
mkdir -p ~/edrf-deploy/bundle-<version>
mv /path/to/edrf-xclient-bundle-<version>.tar.gz ~/edrf-deploy/bundle-<version>/
cd ~/edrf-deploy/bundle-<version>
tar -xzf edrf-xclient-bundle-<version>.tar.gz
ls -la
```
**Expected:** `image.tar`, `edrf-xclient/`, `deploy.sh`, `extract-coninfo.sh`. If `tar` creates
a subfolder named `edrf-xclient-bundle-<version>`, `cd` into it. Every later command runs
from the folder that contains `deploy.sh`.

### Step 3 — Create the `epo-coninfo` secret

Create and download the **Linux** SmartInstaller (`TrellixSmartInstall.sh`) from the ePO
on-prem console: **Menu → Systems → System Tree → New Systems → Create URL for client-side
agent download**, pick the **Non-Windows (Linux)** agent version and **All Agent Handlers**,
click **OK**, then open the generated URL and download the file (full steps in Part B4). Save
it into the bundle folder. Then:

**Windows (`cmd.exe`, one line):**
```cmd
"C:\Program Files\Git\bin\bash.exe" extract-coninfo.sh TrellixSmartInstall.sh <namespace>
```
**Linux:**
```bash
bash extract-coninfo.sh TrellixSmartInstall.sh <namespace>
```
**Expected:** `[1/5]` to `[5/5]`, then `Done.` and `Output file : epo-coninfo.yml`.

Then, the same on both:
```
oc apply -f epo-coninfo.yml
oc get secret epo-coninfo -n <namespace>
```
**Expected:** `secret/epo-coninfo created`, then one row named `epo-coninfo`.

### Step 4 — Get the image digest and dry-run

Get the digest for this build from Artifactory (Part C). Then dry-run, using the same command
on both:
```
helm upgrade --install edrf-xclient ./edrf-xclient -n <namespace> --set image.repository="<image_repo>" --set image.digest="sha256:<digest>" --set imagePullSecret="artifactory-pull-secret" --set deploymentDetails.clusterName="<cluster_name>" --dry-run=client
```
**Expected:** rendered manifests, `STATUS: pending-upgrade`, `DESCRIPTION: Dry run complete`,
and no `required` errors. Nothing changes on the cluster.

To check just the image line, append `| findstr "image:"` (Windows) or `| grep "image:"`
(Linux). It must end in `@sha256:` plus 64 hex characters, with no `<` or `>`.

### Step 5 — Install or upgrade with `deploy.sh`

**Windows (`cmd.exe`, one line):**
```cmd
"C:\Program Files\Git\bin\bash.exe" ./deploy.sh --namespace <namespace> --cluster-name <cluster_name> --image-repo <image_repo> --image-digest sha256:<digest> --image-pull-secret artifactory-pull-secret
```
**Linux:**
```bash
chmod +x deploy.sh
./deploy.sh \
  --namespace <namespace> \
  --cluster-name <cluster_name> \
  --image-repo <image_repo> \
  --image-digest sha256:<digest> \
  --image-pull-secret artifactory-pull-secret
```

- **Windows:** don't type `./deploy.sh` on its own in `cmd.exe`, and keep the command on one
  line.
- **Linux:** `chmod +x` is needed because the extracted script may not be executable
  (otherwise you get `Permission denied`). `bash deploy.sh ...` also works.

**Expected:**
```
==> Mode: registry (image: <image_repo>@sha256:<digest>)
==> Installing Helm chart
Release "edrf-xclient" has been upgraded. Happy Helming!
STATUS: deployed
REVISION: <n>
==> Deployment complete. Check pod status:
```

### Step 6 — Verify

The same commands on both:
```
helm list -n <namespace>
oc rollout status ds/edrf-xclient-container-security -n <namespace>
oc get pods -n <namespace> -o wide
oc exec -n <namespace> <pod-name> -- ls -l /var/McAfee/agent/coninfo.xml
oc logs -n <namespace> <pod-name>
```
**Expected:** release `deployed`, rollout `successfully rolled out` (an upgrade can take 20–25
minutes), one `1/1 Running` pod per Linux worker node with `RESTARTS 0`, `coninfo.xml` present
in the pod, and logs showing the agent provisioning against ePO. Then confirm the nodes appear
in the ePO console.

### Windows vs. Linux at a glance

| Step | Windows (`cmd.exe`) | Linux |
|---|---|---|
| Tool prerequisite | Install Git for Windows | Nothing extra |
| Create folder | `mkdir C:\edrf-deploy\bundle-<version>` | `mkdir -p ~/edrf-deploy/bundle-<version>` |
| Move bundle | `move <src> <dest>` | `mv <src> <dest>` |
| Change folder | `cd /d C:\edrf-deploy\bundle-<version>` | `cd ~/edrf-deploy/bundle-<version>` |
| List files | `dir` | `ls -la` |
| Run `extract-coninfo.sh` | `"C:\Program Files\Git\bin\bash.exe" extract-coninfo.sh ...` | `bash extract-coninfo.sh ...` |
| Run `deploy.sh` | `"C:\Program Files\Git\bin\bash.exe" ./deploy.sh ...` (one line) | `chmod +x deploy.sh` then `./deploy.sh ...` |
| Filter output | `\| findstr "image:"` | `\| grep "image:"` |
| `oc` / `helm` / `tar` commands | Same | Same |

---

## Part 0 — Choose your machine and deployment mode

### 0.1 Which machine does what

This deployment touches up to three systems, and many corporate environments don't let one
machine reach all of them:

1. The **OpenShift cluster API**, needed for every `oc`, `helm`, and `deploy.sh` command.
2. **Artifactory**, needed only for Part C (image digest lookup) in registry mode.
3. The **ePO console** (in a browser), needed only to download the SmartInstaller (Part B4).

A locked-down **jumpbox** dedicated to cluster administration often has no route to the
general internet. An everyday workstation usually has Artifactory access but no path to the
OpenShift API.

**Run this once, from your jumpbox:**
```
curl -v --max-time 10 https://artifactory.trellix.com
```
(PowerShell: `curl.exe` instead of `curl`.)

- **Fast response** (even an error page, as long as it doesn't hang) means the jumpbox has
  internet access. Run every part of this guide from the jumpbox.
- **Hangs until timeout** means you split the work:
  - **Part C only** (looking up the image repository and digest): run it from your workstation.
    You only need `curl` and your Artifactory credentials (Part B3) there.
  - **Everything else** stays on the jumpbox.
  - Values to carry between machines: the image repository and digest (two short strings), and
    `TrellixSmartInstall.sh` if you downloaded it on a different machine. Copy the
    SmartInstaller as a file and don't open it in an editor (Part B4).

### 0.2 Which deployment mode

`deploy.sh` supports two modes:

| Mode | Use when | You must provide |
|---|---|---|
| **Registry** (default) | The cluster's worker nodes can pull images from Artifactory (or an internal mirror) | A pull secret (Part F), plus the image repository and digest (Part C) |
| **Airgap** | The cluster can't reach any external registry | `skopeo`, `docker`, or `podman` on the jumpbox. You push `image.tar` to an internal registry yourself, then give `deploy.sh` that registry's repository and the pushed digest (Part H2). No pull secret. |

If you aren't sure, start with **registry mode**. If pods later fail with `ImagePullBackOff`
and a network timeout (not an authentication error), the nodes can't reach Artifactory and you
need airgap mode.

Both modes need the `epo-coninfo` secret (Part E).

Every part below is tagged **(jumpbox)**, **(any machine)**, **(Artifactory-reachable
machine)**, **(registry mode)**, or **(airgap mode)**.

---

## Part A (jumpbox) — Install the tools

| Tool | Needed for | Windows | Linux/Mac |
|---|---|---|---|
| `oc` | Everything | Install (A1) | Install (A1) |
| `helm` (v3) | `deploy.sh` calls it internally, plus the dry-run | Install (A2) | Install (A2) |
| Git for Windows (Git Bash) | Running `extract-coninfo.sh` and `deploy.sh` | **Install (A3)** | Not needed (bash is built in) |
| `curl`, `tar` | Digest lookup, extracting the bundle | Built into Windows 10/11 | Built in |
| `skopeo`, `docker`, or `podman` | **Airgap mode only** | Install one (A4) | Install one (A4) |

### A1. Install the `oc` CLI

**Option 1 — from the cluster's web console** (matches your cluster version exactly):
1. Log in to the OpenShift web console.
2. Click the **`?` (help)** icon (top-right) → **Command Line Tools**.
3. Download the **oc** CLI for your OS.

**Option 2 — from the official mirror:**
https://mirror.openshift.com/pub/openshift-v4/clients/oc/latest/ has separate subfolders for
Windows (`.zip`), Linux (`.tar.gz`), and Mac (`.tar.gz`).

**Extract the archive:**

**PowerShell:**
```powershell
Expand-Archive -Path "$env:USERPROFILE\Downloads\oc.zip" -DestinationPath "$env:USERPROFILE\Downloads\oc-extracted"
```
**cmd.exe** (Windows' built-in `tar.exe` also extracts `.zip` files. Create the destination
first, because `tar -C` needs it to exist):
```cmd
mkdir "%USERPROFILE%\Downloads\oc-extracted"
tar -xf "%USERPROFILE%\Downloads\oc.zip" -C "%USERPROFILE%\Downloads\oc-extracted"
```
**Linux/Mac** (download is `.tar.gz`):
```bash
mkdir -p ~/oc-extracted
tar -xzf oc.tar.gz -C ~/oc-extracted
```

**Move the binary to a permanent location:**

**PowerShell:**
```powershell
mkdir C:\oc
Copy-Item "$env:USERPROFILE\Downloads\oc-extracted\oc.exe" -Destination C:\oc\oc.exe
```
**cmd.exe:**
```cmd
mkdir C:\oc
copy "%USERPROFILE%\Downloads\oc-extracted\oc.exe" C:\oc\oc.exe
```
**Linux/Mac:**
```bash
mkdir -p ~/bin
cp ~/oc-extracted/oc ~/bin/oc
chmod +x ~/bin/oc
```

**Add it to PATH:**

**PowerShell** (permanent, and also applies to future `cmd.exe` and Git Bash sessions):
```powershell
[Environment]::SetEnvironmentVariable("Path", "$env:Path;C:\oc", "User")
```
**cmd.exe** (`setx` can truncate a very long existing PATH. If your PATH is already large, use
the PowerShell command above or the GUI method instead):
```cmd
setx PATH "%PATH%;C:\oc"
```
**Windows GUI** (works whatever your default shell is):
1. **Win + R** → `sysdm.cpl` → Enter.
2. **Advanced** tab → **Environment Variables**.
3. Under **User variables**, select `Path` → **Edit** → **New**.
4. Add `C:\oc` → OK on all dialogs.

**Linux/Mac:**
```bash
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc    # or ~/.zshrc
source ~/.bashrc
```

**Open a new terminal**, then verify:

**Windows:**
```
where.exe oc
oc version --client
```
**Linux/Mac:**
```bash
which oc
oc version --client
```

**Expected:** the path command prints the binary's location, and `oc version --client` prints
`Client Version: 4.x.x`.

**If `oc` is still not recognized:** PATH wasn't updated, or the terminal was opened before
the change. Redo the PATH step and open a fresh terminal.

### A2. Install the `helm` CLI

**Windows:**
```
winget install Helm.Helm
```
Or download from https://helm.sh/docs/intro/install/ and add the folder containing
`helm.exe` to PATH as in A1.

**Mac:**
```bash
brew install helm
```

**Linux:**
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod +x get_helm.sh
./get_helm.sh
```

**Open a new terminal**, then verify:
```
helm version
```

**Expected:** `version.BuildInfo{Version:"v3...`. This chart requires Helm 3.

### A3. Install Git for Windows (Windows only)

`deploy.sh` and `extract-coninfo.sh` are bash scripts. Running them straight from `cmd.exe`
fails with `'.' is not recognized as an internal or external command`. Git for Windows
provides **Git Bash**, which runs them, along with the tools `extract-coninfo.sh` needs (`awk`,
`dd`, `base64`, `tail`, and others).

**Install:**
```
winget install --id Git.Git -e
```
Or download the installer from https://git-scm.com/download/win and accept the defaults. If
software installs are restricted on the jumpbox, request Git for Windows from IT.

**Open a new terminal**, then verify:
```
"C:\Program Files\Git\bin\bash.exe" --version
```
(PowerShell: `& "C:\Program Files\Git\bin\bash.exe" --version`.)

**Expected:** `GNU bash, version 5.x...`.

If that path doesn't exist, Git may have been installed per-user. Find it with:
```
where.exe git
```
`bash.exe` is in the `bin` folder next to the `cmd` folder that contains `git.exe`. For
example, `...\Git\cmd\git.exe` means bash is at `...\Git\bin\bash.exe`. Use that path wherever
this guide shows `C:\Program Files\Git\bin\bash.exe`.

> **Do not use plain `bash` on Windows.** Typing `bash` on its own usually starts **WSL**
> (Windows Subsystem for Linux), not Git Bash. WSL can't see your Windows `oc.exe` and
> `helm.exe` or your cluster login, so `deploy.sh` fails with `oc: command not found`. Always
> call Git Bash by its full path, as shown above.

### A4. Install an image tool (airgap mode only)

Skip this in registry mode. In airgap mode you push `image.tar` to your internal registry
yourself, with any one of:

- **`skopeo`** (recommended, because it needs no background daemon). Linux:
  `sudo dnf install skopeo` or `sudo apt install skopeo`.
- **`podman`**. Linux: `sudo dnf install podman` or `sudo apt install podman`. Windows: Podman
  Desktop.
- **`docker`**. Linux: Docker Engine. Windows/Mac: Docker Desktop (or whatever your
  organization approves).

**Verify** whichever one you installed:
```
skopeo --version
podman version
docker version
```
For `docker`, the output must include both a `Client` and a `Server` section. If only `Client`
appears, start Docker Desktop or the Docker service.

---

## Part B — Collect access and credentials

B1, B2, and B5 run on the jumpbox. B3 and B4 need a browser and can run from any machine.

### B1 (jumpbox). OpenShift cluster login

Ask whoever provisioned your access which method applies.

**Option 1 — token login:**
1. Log in to the web console.
2. Click your username (top-right) → **Copy login command**.
3. In the new tab, click **Display Token**.
4. Copy and run the resulting command (the same in all three shells):
   ```
   oc login --token=sha256~xxxxxxxxxxxx --server=https://api.<your-cluster>:6443
   ```

Tokens from this flow are short-lived. If `oc whoami` later fails with an auth error, repeat
these steps to get a fresh token.

**Option 2 — kubeconfig file:**

Some teams issue a `kubeconfig` file directly (often for a shared service account) instead of
individual logins.

1. Save the file somewhere stable, e.g. `C:\Users\<you>\Documents\kubeconfig`.
2. Point `oc` at it with the `KUBECONFIG` environment variable. A variable set in one shell
   window isn't visible in another unless you make it permanent.

   **PowerShell** (current session):
   ```powershell
   $env:KUBECONFIG = "C:\Users\<you>\Documents\kubeconfig"
   ```
   Permanent (also applies to future `cmd.exe` and Git Bash sessions):
   ```powershell
   [Environment]::SetEnvironmentVariable("KUBECONFIG", "C:\Users\<you>\Documents\kubeconfig", "User")
   ```

   **cmd.exe** (current session):
   ```cmd
   set KUBECONFIG=C:\Users\<you>\Documents\kubeconfig
   ```
   `set` only affects the current window. To make it permanent, use the PowerShell command
   above or the GUI method from A1 with the variable name `KUBECONFIG`.

   **Linux/Mac** (current session):
   ```bash
   export KUBECONFIG=/home/<you>/kubeconfig
   ```
   Permanent:
   ```bash
   echo 'export KUBECONFIG=/home/<you>/kubeconfig' >> ~/.bashrc    # or ~/.zshrc
   ```

> **Windows and the bundle scripts:** run `deploy.sh` from the **same** `cmd.exe` or
> PowerShell window where you logged in or set `KUBECONFIG` (Part H1 shows how). Git Bash
> launched from that window inherits your PATH and login. A Git Bash window opened separately
> from the Start menu won't have a `KUBECONFIG` set with `set` or `$env:` unless you made it
> permanent.

**Verify login (either option, the same in every shell):**
```
oc whoami
oc get nodes
```

**Expected:** `oc whoami` prints your identity (or the account in the kubeconfig, e.g.
`system:admin`), and every node shows `STATUS = Ready`. A hang here is a network, VPN, or DNS
problem reaching the API server, not a login problem.

**If `oc whoami` fails:**
- *Token login:* the token has probably expired, so repeat Option 1.
- *Kubeconfig login:* check that the variable is set (`echo $env:KUBECONFIG` /
  `echo %KUBECONFIG%` / `echo $KUBECONFIG`) and that the file exists at that path
  (`Test-Path $env:KUBECONFIG` / `dir "%KUBECONFIG%"` / `ls -la "$KUBECONFIG"`). See which
  identity and cluster it points to with `oc config current-context`.

### B2 (jumpbox). Confirm cluster permissions

You create two secrets by hand (`epo-coninfo` and the pull secret). The chart creates
namespaced resources (ServiceAccount, RoleBinding, DaemonSet) **and cluster-wide resources**: a
custom **SecurityContextConstraints** and a **ClusterRole**. Creating cluster-wide resources
usually needs cluster-admin rights.

Check all four before going further:
```
oc auth can-i create secrets -n edrf-tests
oc auth can-i create rolebindings -n edrf-tests
oc auth can-i create clusterroles
oc auth can-i create securitycontextconstraints
```

**Expected:** `yes` for all four. The SCC check also prints
`Warning: resource 'securitycontextconstraints' is not namespace scoped`, which is normal.

If any of them return `no`, stop here. Ask your OpenShift platform or cluster-admin team to
grant the permission, or to run `deploy.sh` (Part H) for you. Without it the install fails
partway with a `forbidden` error.

### B3 (any machine with a browser). Artifactory credentials (registry mode)

You need an **identity token** (or API key), not your normal login password. Artifactory
doesn't accept SSO passwords for CLI or API pulls.

1. Log in at https://artifactory.trellix.com.
2. Click your username or avatar (top-right) → **Edit Profile**.
3. Find **Generate an Identity Token** (newer JFrog UI) or **Generate API Key** (older UI).
4. Generate one and save it in a password manager right away. Most Artifactory versions only
   show it once.

**The identity token screen shows several fields. Only one of them is the credential:**

| Field shown on screen | Example shape | Use it? |
|---|---|---|
| **Reference token** | long base64 string, typically starting `cmVmdGtu...` | Yes, this is the credential. Use it as `<artifactory_token>` throughout this guide. |
| Username | your email | This is `<artifactory_user>`. |
| Scope / Audience | e.g. `applied-permissions/user`, `*@*` | Informational only. |
| Expires In | e.g. `11 month(s) 30 day(s)` | Tells you when to generate a new one. |
| **Token ID** | short UUID, e.g. `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` | No, this isn't a credential. Using it as a password fails with `401 Unauthorized`. |

### B4 (any machine with a browser). ePO SmartInstaller and cluster name

**SmartInstaller.** The agent gets its ePO connection details from `coninfo.xml`, which is
packed inside the ePO **Linux SmartInstaller**. You create it in the ePO on-prem console by
generating an agent deployment URL, then download it from that URL:

1. Log in to the **ePO on-prem console** that these cluster nodes should report to.
2. Select **Menu → Systems → System Tree**, then select **New Systems**.
3. Select **Create URL for client-side agent download**.
4. Enter a custom **URL name** (any name that identifies this deployment, for example
   `edrf-xclient-<cluster_name>`).
5. From the list, select the **Agent version** for **Non-Windows (Linux)** operating systems.
   Don't pick a Windows agent; `extract-coninfo.sh` only works with the Linux SmartInstaller.
6. Under Agent Handlers, select **All Agent Handlers**.
7. Click **OK**. ePO shows the **Agent Deployment URL**. You can use this URL to download the
   SmartInstaller yourself or send it to whoever will run the deployment.
8. Open the URL in a browser and download the file **`TrellixSmartInstall.sh`** (several MB).
9. Copy it into the bundle folder on the jumpbox (Part B5).

> **Don't open or edit `TrellixSmartInstall.sh` in a text editor.** It's a shell script with a
> binary payload appended, and saving it from an editor can corrupt it. Copy it only as a
> file. Treat it as sensitive, because it contains your ePO server's certificates and keys.

If you don't have ePO console access, ask your ePO admin for the Linux SmartInstaller.

**Cluster name.** You also need a **cluster name**, the label this cluster will have in ePO.
Get the cluster's infrastructure name with:
```
oc get infrastructure cluster -o jsonpath='{.status.infrastructureName}'
```
This returns something like `mycluster-ab1cd`, where the part before the random suffix
(`mycluster`) is usually the value to use. Node names follow the same pattern
(`mycluster-ab1cd-worker-a-xyz12`). If other clusters already report to your ePO, follow
their naming convention.

### B5 (jumpbox). Get and extract the bundle

Get `edrf-xclient-bundle-<version>.tar.gz` from your build or release team. Copy it to the
jumpbox, then extract it into a **new, empty folder** so it doesn't mix with older bundles.
`tar` is built into Windows 10/11, so the extract command is the same everywhere.

**PowerShell:**
```powershell
mkdir C:\edrf-deploy\bundle-<version>
cd C:\edrf-deploy\bundle-<version>
tar -xzf C:\path\to\edrf-xclient-bundle-<version>.tar.gz
```
**cmd.exe:**
```cmd
mkdir C:\edrf-deploy\bundle-<version>
cd /d C:\edrf-deploy\bundle-<version>
tar -xzf C:\path\to\edrf-xclient-bundle-<version>.tar.gz
```
**Linux/Mac:**
```bash
mkdir -p ~/edrf-deploy/bundle-<version>
cd ~/edrf-deploy/bundle-<version>
tar -xzf /path/to/edrf-xclient-bundle-<version>.tar.gz
```

**Verify:**
```
dir          (Windows)
ls -la       (Linux/Mac)
```

**Expected:** four items:
```text
image.tar             # container image (only used in airgap mode)
edrf-xclient/         # Helm chart
deploy.sh             # install script (registry and airgap modes)
extract-coninfo.sh    # builds the epo-coninfo secret manifest (run before deploy.sh)
```
If you see a single folder named `edrf-xclient-bundle-<version>` instead, `cd` into it.
Everything else in this guide runs from the folder that contains `deploy.sh`.

Now copy `TrellixSmartInstall.sh` (Part B4) into this same folder.

The chart layout, for reference:
```text
edrf-xclient/
├── Chart.yaml
├── values.yaml                  # default values (deploy.sh flags override these)
└── templates/
    ├── 01-serviceaccount.yaml   # ServiceAccount, custom SCC, ClusterRole, RoleBinding
    └── 02-deploy-edrf.yaml      # the DaemonSet; mounts the epo-coninfo secret
```

**You don't need to edit `values.yaml`.** `deploy.sh` passes every required value as a flag
(Appendix A lists them). If you do edit it, keep that copy out of git.

---

## Part C (Artifactory-reachable machine, registry mode) — Get the image repository and digest

Skip this part in airgap mode (Part H2 covers getting the digest there).

**The digest changes with every build.** Never reuse the digest from a previous build.

If Part 0 showed the jumpbox has no internet access, run this part on your workstation. You
only need `curl` and your Artifactory credentials.

You need two values for `deploy.sh`:
- `--image-repo`, e.g. `artifactory.trellix.com/edrf-xclient-container-security/trellix-edrf`
- `--image-digest`, e.g. `sha256:<64 hex characters>`

### Find the build folder

In a browser, open Artifactory and browse to `edrf-xclient-container-security` →
`trellix-edrf`. There's one folder per build. Find the folder for your bundle's build and
note its exact name (`<build>` below).

### Method 1 (preferred) — Artifactory Storage API

A Docker image digest is the SHA256 checksum of its manifest. Artifactory's Storage API returns
it directly:

```
curl -s -u "<artifactory_user>:<artifactory_token>" "https://artifactory.trellix.com/artifactory/api/storage/edrf-xclient-container-security/trellix-edrf/<build>/manifest.json"
```
(PowerShell: `curl.exe`.)

A working response:
```json
{
  "repo": "edrf-xclient-container-security",
  "path": "/trellix-edrf/<build>/manifest.json",
  "checksums": {
    "sha1": "...",
    "md5": "...",
    "sha256": "<64 hex characters>"
  }
}
```
Your digest is `sha256:` followed by the `sha256` value. Your image repository is:
```
artifactory.trellix.com/edrf-xclient-container-security/trellix-edrf
```

**Without curl:** in the Artifactory web UI, click `manifest.json` inside the build folder.
The file's info panel shows its **SHA-256** checksum, which is the same value.

**If the curl response isn't clean JSON**, re-run with `-v` instead of `-s` and check the HTTP
status:
- **`200 OK` with `"children": [...]`** means `<build>` is a folder and `manifest.json` isn't
  the real filename. List the folder (remove `manifest.json` from the URL) and retry with the
  real name.
- **`401` / `403`** means the username or token is wrong, expired, or lacks read access. Check
  you used the **Reference Token** (Part B3).
- **`404`** means the repo key or build folder is wrong. Confirm the path in the browser.
- **`302` redirect or an HTML login page** means this auth method is rejected for API calls.
  Check with your Artifactory admins.

### Method 2 (fallback) — Docker Registry API

If the Docker repository name differs from the browse-UI name:

**PowerShell:**
```powershell
curl.exe -sI -u "<artifactory_user>:<artifactory_token>" -H "Accept: application/vnd.docker.distribution.manifest.v2+json" "https://artifactory.trellix.com/artifactory/api/docker/<docker-repo-name>/v2/trellix-edrf/manifests/<build>" | Select-String -Pattern "docker-content-digest"
```
**cmd.exe:**
```cmd
curl -sI -u "<artifactory_user>:<artifactory_token>" -H "Accept: application/vnd.docker.distribution.manifest.v2+json" "https://artifactory.trellix.com/artifactory/api/docker/<docker-repo-name>/v2/trellix-edrf/manifests/<build>" | findstr /i "docker-content-digest"
```
**Linux/Mac:**
```bash
curl -sI -u "<artifactory_user>:<artifactory_token>" -H "Accept: application/vnd.docker.distribution.manifest.v2+json" "https://artifactory.trellix.com/artifactory/api/docker/<docker-repo-name>/v2/trellix-edrf/manifests/<build>" | grep -i docker-content-digest
```
**Expected:** `Docker-Content-Digest: sha256:...`. The repository is then
`artifactory.trellix.com/<docker-repo-name>/trellix-edrf`.

---

## Part D (jumpbox) — Create the namespace

Create the namespace **before** anything else on the cluster. The `epo-coninfo` secret (Part
E) and the pull secret (Part F) are created inside it, and both must exist before `deploy.sh`
runs. `deploy.sh` passes `--create-namespace` to Helm, but that happens too late to hold the
secrets.

```
oc new-project edrf-tests
```
If it already exists (for example, you're upgrading), skip this. To check:
```
oc get ns edrf-tests
```

**Expected:** one row with `STATUS Active`.

---

## Part E (jumpbox) — Create the ePO `epo-coninfo` secret (required, both modes)

**This step is mandatory.** Every pod mounts `coninfo.xml` from the `epo-coninfo` secret at
startup to register with ePO. `deploy.sh` checks for the secret and refuses to run without it
(`ERROR: Secret 'epo-coninfo' not found`). If the secret is removed later, pods can't start.

Do this once per namespace. Redo it only if you need to point the agents at a different ePO
server, or the ePO server's certificates change (Part L).

### E1. Put the SmartInstaller in the bundle folder

Copy `TrellixSmartInstall.sh` from Part B4 into the bundle folder (the one containing
`extract-coninfo.sh`). Check it's there:
```
dir TrellixSmartInstall.sh        (Windows)
ls -la TrellixSmartInstall.sh     (Linux/Mac)
```
**Expected:** a file of several MB.

### E2. Generate `epo-coninfo.yml`

`extract-coninfo.sh` takes two arguments: the SmartInstaller file and your namespace. It
extracts `coninfo.xml` and writes `epo-coninfo.yml` (a Secret manifest) into the **current
folder**. It doesn't change anything on the cluster.

**cmd.exe** (one line):
```cmd
"C:\Program Files\Git\bin\bash.exe" extract-coninfo.sh TrellixSmartInstall.sh edrf-tests
```
**PowerShell** (one line):
```powershell
& "C:\Program Files\Git\bin\bash.exe" extract-coninfo.sh TrellixSmartInstall.sh edrf-tests
```
**Linux/Mac or Git Bash:**
```bash
bash extract-coninfo.sh TrellixSmartInstall.sh edrf-tests
```

**Expected:**
```
[1/5] Reading SmartInstaller: TrellixSmartInstall.sh
      bootstrap_size = <n> bytes
[2/5] Checking available disk space...
[3/5] Locating archive marker...
      Archive starts at line <n>
[4/5] Extracting coninfo.xml...
      coninfo.xml size: <n> bytes
[5/5] Generating epo-coninfo.yml...

Done.
  Output file : epo-coninfo.yml
  Secret name : epo-coninfo
  Namespace   : edrf-tests
```
`coninfo.xml` is typically a few KB. The script needs free disk space of about twice the
SmartInstaller's size for a temporary copy, which it deletes afterwards.

Errors from this script are listed in Part K.

### E3. Create the secret on the cluster

```
oc apply -f epo-coninfo.yml
oc get secret epo-coninfo -n edrf-tests
```
**Expected:** `secret/epo-coninfo created` (or `configured` if it already existed), then one
row: `epo-coninfo   Opaque   1`.

The script's own output suggests `kubectl apply`. `oc apply` does exactly the same thing.

- **Don't rename the secret.** The chart references exactly `epo-coninfo`.
- **Delete `epo-coninfo.yml` afterwards**, or store it somewhere secure. It contains your ePO
  connection details. The secret on the cluster stays. Never commit it, or
  `TrellixSmartInstall.sh`, to git.

---

## Part F (jumpbox, registry mode) — Create the Artifactory pull secret

Skip this part in airgap mode.

OpenShift needs its own copy of your Artifactory credentials to pull the private image. Create
it once per namespace. If you're upgrading an existing deployment, it probably already exists,
so run the verify command first.

```
oc create secret docker-registry artifactory-pull-secret --docker-server=artifactory.trellix.com --docker-username="<artifactory_user>" --docker-password="<artifactory_token>" -n edrf-tests
```

Use the **Reference Token** as the password (Part B3). If the token contains characters such as
`&`, `|`, `^`, or `!`, `cmd.exe` can misread them even inside quotes, so run this command from
PowerShell instead.

**Verify:**
```
oc get secret artifactory-pull-secret -n edrf-tests
```

**Expected:** one row, `TYPE kubernetes.io/dockerconfigjson`.

The secret's **name**, `artifactory-pull-secret`, is what you pass to `deploy.sh` as
`--image-pull-secret`. Never pass the token itself.

> **Note:** the product README creates this secret with
> `oc create secret generic <name> --from-file=.dockerconfigjson=config.json --type=kubernetes.io/dockerconfigjson`.
> That produces the same secret type, but you have to build `config.json` by hand, and its
> `auth` field must be base64 of `username:token`. Getting that wrong is a common cause of
> `401` pull failures. `oc create secret docker-registry` builds it correctly for you.

---

## Part G (jumpbox) — Dry-run the chart (recommended)

A dry-run catches missing or wrong values before anything changes on the cluster. Run it from
the bundle folder with the same values you'll give `deploy.sh`. This is a single line and works
in all three shells:

```
helm upgrade --install edrf-xclient ./edrf-xclient -n edrf-tests --set image.repository="<image_repo>" --set image.digest="sha256:<digest>" --set imagePullSecret="artifactory-pull-secret" --set deploymentDetails.clusterName="<cluster_name>" --dry-run=client
```
(Airgap mode: use your internal registry's repository and digest from Part H2, and drop the
`imagePullSecret` setting.)

**Expected:** a long list of rendered manifests ending without errors. The output includes
these lines, which are **normal for a dry-run**. Nothing was changed on the cluster:
```
Release "edrf-xclient" has been upgraded. Happy Helming!
STATUS: pending-upgrade
DESCRIPTION: Dry run complete
```

**Check the manifests for:**
- a `SecurityContextConstraints` named `edrf-xclient-edrf-tests`, a `ClusterRole`, and a
  `RoleBinding`
- in the DaemonSet, a volume named `coninfo` with `secretName: epo-coninfo`, mounted at
  `/var/McAfee/agent/coninfo.xml`
- the `image:` line ending in your real digest

**The dry-run doesn't validate the digest itself.** If you accidentally leave a placeholder
like `sha256:<digest>` in the command, the dry-run still passes, but every pod later fails with
`InvalidImageName`. To check just the image line, append a filter:
```
... --dry-run=client | findstr "image:"       (Windows)
... --dry-run=client | grep "image:"          (Linux/Mac)
```
It must end in `@sha256:` followed by exactly 64 hex characters, with no `<` or `>`.

An `execution error ... is required` means a value is missing or misspelled. Fix it and re-run.

---

## Part H (jumpbox) — Run `deploy.sh`

`deploy.sh` first checks that the `edrf-xclient/` chart folder is next to it and that the
`epo-coninfo` secret exists in the namespace. Then it runs `helm upgrade --install`. It accepts
only these flags:

| Flag | Registry mode | Airgap mode |
|---|---|---|
| `--namespace` | Required | Required |
| `--cluster-name` | Required | Required |
| `--image-repo` | Required | Required (your internal registry) |
| `--image-digest` | Required | Required (the digest you pushed) |
| `--image-pull-secret` | Required | Not used |
| `--mode` | Optional (`registry` is the default) | `airgap` |

Any other flag (including the old `--epo-ip`, `--epo-user`, `--epo-password`) stops the
script with `ERROR: Unknown option`.

### H1. Registry mode

Run from the bundle folder. Replace every `<...>` placeholder. **Pick the block for your
shell.**

**cmd.exe** (one line; launches Git Bash from this window so it inherits your login):
```cmd
"C:\Program Files\Git\bin\bash.exe" ./deploy.sh --namespace edrf-tests --cluster-name <cluster_name> --image-repo <image_repo> --image-digest sha256:<digest> --image-pull-secret artifactory-pull-secret
```

**PowerShell** (one line; note the leading `&`):
```powershell
& "C:\Program Files\Git\bin\bash.exe" ./deploy.sh --namespace edrf-tests --cluster-name <cluster_name> --image-repo <image_repo> --image-digest sha256:<digest> --image-pull-secret artifactory-pull-secret
```

**Linux/Mac, or inside a Git Bash window:**
```bash
chmod +x deploy.sh
./deploy.sh \
  --namespace edrf-tests \
  --cluster-name <cluster_name> \
  --image-repo <image_repo> \
  --image-digest sha256:<digest> \
  --image-pull-secret artifactory-pull-secret
```

**Shell rules that cause most first-time failures:**
- **Never type `./deploy.sh` directly in `cmd.exe` or PowerShell.** Always go through Git
  Bash, as shown above.
- In `cmd.exe` and PowerShell, keep the command on **one line**. The `\` line continuation
  only works in bash.

**Expected output:**
```
==> Mode: registry (image: <image_repo>@sha256:<digest>)
==> Installing Helm chart
Release "edrf-xclient" has been upgraded. Happy Helming!
NAME: edrf-xclient
NAMESPACE: edrf-tests
STATUS: deployed
REVISION: <n>
DESCRIPTION: Upgrade complete

==> Deployment complete. Check pod status:
    oc get pods -n edrf-tests
```
On a first install, `Release "edrf-xclient" does not exist. Installing it now.` and
`REVISION: 1` appear instead. Both are success. (`--mode registry` is the default, and
`image.tar` isn't used in this mode.)

### H2. Airgap mode

> Airgap mode is documented from the product README and the bundle's `deploy.sh`. It hasn't
> yet been validated end-to-end on a test cluster for this guide.

In airgap mode **you push `image.tar` to an internal registry yourself**, then give
`deploy.sh` that registry's repository and the pushed image's digest. No pull secret is used.
(The comment at the top of `deploy.sh` still says the registry is auto-discovered, but the
script's code requires `--image-repo` and `--image-digest`.)

**Prerequisites:** `oc` logged in (Part B1), the `epo-coninfo` secret created (Part E), and
`skopeo`, `docker`, or `podman` on the jumpbox (Part A4).

#### H2.1. Choose the registry

The most common choice is the **OpenShift internal registry**, which OpenShift trusts by
default. Using it involves **two different addresses**:

| Address | Example | Used for |
|---|---|---|
| **External route** | `default-route-openshift-image-registry.apps.<cluster-domain>` | **Pushing** from the jumpbox |
| **Internal service** | `image-registry.openshift-image-registry.svc:5000` | **Pulling** by the pods, so this is what you pass to `deploy.sh` |

The image path in either case is `<address>/<namespace>/trellix-edrf`.

A cluster-admin exposes the external route once (cluster-wide):
```
oc patch configs.imageregistry.operator.openshift.io/cluster --patch "{\"spec\":{\"defaultRoute\":true}}" --type=merge
```
(The escaped double quotes work in `cmd.exe` and PowerShell. In bash you can use
`--patch '{"spec":{"defaultRoute":true}}'`.)

Get the route hostname:
```
oc get route default-route -n openshift-image-registry -o jsonpath="{.spec.host}"
```

If you use a different internal registry (for example, a corporate mirror), use its address for
both pushing and `--image-repo`, and create a pull secret for it if it requires
authentication.

#### H2.2. Push `image.tar` and capture the digest

Run from the bundle folder. Replace `<route>` with the hostname from H2.1. Use **one** tool.

**skopeo** (recommended; Linux):
```bash
skopeo copy --dest-creds "$(oc whoami):$(oc whoami -t)" --digestfile digest.txt \
  docker-archive:image.tar \
  docker://<route>/edrf-tests/trellix-edrf:bundle
cat digest.txt
```

**podman:**
```bash
oc registry login --registry=<route> --to=auth.json
podman load -i image.tar
podman tag trellix-edrf:bundle <route>/edrf-tests/trellix-edrf:bundle
podman push --authfile auth.json --digestfile digest.txt <route>/edrf-tests/trellix-edrf:bundle
cat digest.txt
```

**docker:**
```bash
oc registry login --registry=<route>
docker load -i image.tar
docker tag trellix-edrf:bundle <route>/edrf-tests/trellix-edrf:bundle
docker push <route>/edrf-tests/trellix-edrf:bundle
```
`docker push` prints a final line such as `bundle: digest: sha256:<64 hex> size: <n>`.

- **Expected:** a digest of the form `sha256:<64 hex characters>`. That's your
  `--image-digest`.
- **`x509: certificate signed by unknown authority`:** the route uses a certificate the
  jumpbox doesn't trust. Add `--dest-tls-verify=false` (skopeo) or `--tls-verify=false`
  (podman), or configure Docker's `insecure-registries`. Only do this if your security policy
  allows it.
- **Note:** `oc whoami -t` only works with a token login. With a kubeconfig that uses client
  certificates, use `oc registry login` (podman/docker) or ask your admin for registry
  credentials.

Optionally confirm the image landed:
```
oc get imagestream trellix-edrf -n edrf-tests
```

#### H2.3. Deploy

Pass the **internal service** address, not the route:

**cmd.exe:**
```cmd
"C:\Program Files\Git\bin\bash.exe" ./deploy.sh --namespace edrf-tests --cluster-name <cluster_name> --mode airgap --image-repo image-registry.openshift-image-registry.svc:5000/edrf-tests/trellix-edrf --image-digest sha256:<digest>
```
**PowerShell:**
```powershell
& "C:\Program Files\Git\bin\bash.exe" ./deploy.sh --namespace edrf-tests --cluster-name <cluster_name> --mode airgap --image-repo image-registry.openshift-image-registry.svc:5000/edrf-tests/trellix-edrf --image-digest sha256:<digest>
```
**Linux/Mac or Git Bash:**
```bash
./deploy.sh \
  --namespace edrf-tests \
  --cluster-name <cluster_name> \
  --mode airgap \
  --image-repo image-registry.openshift-image-registry.svc:5000/edrf-tests/trellix-edrf \
  --image-digest sha256:<digest>
```

**Expected:** `==> Mode: airgap`, a reminder of the push commands (informational only), then
`STATUS: deployed` and `==> Deployment complete`. No `--image-pull-secret` is needed, because
pods in a namespace can pull images pushed to that namespace in the internal registry.

---

## Part I (jumpbox) — Verify the deployment

### I1. Wait for the rollout to finish

```
oc rollout status ds/edrf-xclient-container-security -n edrf-tests
```

**Expected:** `daemon set "edrf-xclient-container-security" successfully rolled out`.

**On an upgrade, this can take 20–25 minutes.** Old pods are replaced one node at a time, and
each old pod is given up to **8 minutes (480 seconds)** to stop the agent gracefully. A pod
in `Terminating` for several minutes is normal. Don't force-delete it unless it's been stuck
for more than about 10 minutes.

### I2. Check pods and the release

```
helm list -n edrf-tests
oc get pods -n edrf-tests -o wide
```

**Expected:**
- `helm list` shows `STATUS deployed` with the new revision number.
- One pod per Linux worker node, all `1/1 Running`, `RESTARTS 0`. After an upgrade, the pod
  names and `AGE` should be new.

To watch pods change live, add `-w` (press Ctrl+C to stop):
```
oc get pods -n edrf-tests -o wide -w
```

### I3. Confirm `coninfo.xml` reached the pod

```
oc exec -n edrf-tests <pod-name> -- ls -l /var/McAfee/agent/coninfo.xml
```

**Expected:** the file is listed, with a size matching the `coninfo.xml size` that
`extract-coninfo.sh` printed.

### I4. Check the agent logs

```
oc logs -n edrf-tests <pod-name>
```
Or for all pods at once:
```
oc logs -n edrf-tests -l name=edrf-xclient-container-security --tail=100
```

**Expected:** the McAfee Agent provisions against ePO using `coninfo.xml`, then MA and XClient
start, with no repeating errors. `Running` on its own only means the container started, not
that the agent is reporting.

### I5. Confirm in ePO

Check that every worker node appears in the ePO console with a recent last-communication time.

### I6. Recheck restarts after about an hour

```
oc get pods -n edrf-tests
```

`RESTARTS` should still be `0`. If it's climbing, the agent is crashing and Kubernetes is
restarting it, even though the pod looks `Running` most of the time. See Part K.

---

## Part J (jumpbox) — Collect agent logs from a pod

Use this when the dev or support team asks for agent logs.

**1. Open a shell inside the pod:**
```
oc exec -it -n edrf-tests <pod-name> -- /bin/bash
```
(Use `/bin/sh` if bash isn't in the image.)

**2. Inside the pod, find the logs and check the agent process:**
```bash
ps -o pid,ppid,cmd -C xclient
ls -l /var/McAfee/agent/coninfo.xml
ls -lrth /tmp
ls -lrth /var/lib/Trellix/XClient/
ls -lrth /var/McAfee/agent/logs/
exit
```

**3. Copy a file out of the pod to your current folder.** Use a **relative** destination such
as `./<file>`:
```
oc cp -n edrf-tests <pod-name>:/tmp/<log-file-name> ./<log-file-name>
```

> **Windows:** don't use a destination that starts with a drive letter (for example
> `C:\edrf-deploy\file.log`). `oc cp` splits arguments on `:`, so the drive-letter colon
> makes it fail with `error: one of src or dest must be a local file specification`. `cd` to
> the destination folder first and use `./<file>`.

The agent data, MA logs, and TA config also persist on each **node** under
`/mnt/XClientLogs/` (`xclient`, `fexgt`, `ma-logs`), so they survive pod restarts. To browse
them from the node:
```
oc debug node/<node-name>
chroot /host
ls -lrth /mnt/XClientLogs/
```

---

## Part K (jumpbox) — Troubleshooting

### K1. Running the scripts

| Symptom | Likely cause | Action |
|---|---|---|
| `'.' is not recognized as an internal or external command` | `./deploy.sh` was typed directly in `cmd.exe` or PowerShell | Run it through Git Bash (Part H1). Install Git for Windows first if needed (Part A3). |
| `'--namespace' is not recognized...` (one error per line) | A multi-line command with `\` continuation was pasted into `cmd.exe` | Use the one-line `cmd.exe` form |
| `The system cannot find the path specified` for `bash.exe` | Git for Windows isn't installed, or is installed somewhere else | Part A3: install it, or find the real path with `where.exe git` |
| `oc: command not found` or `helm: command not found` from inside `deploy.sh` | bash can't see your Windows tools. Common causes are using WSL (plain `bash`) instead of Git Bash, or `oc`/`helm` not on the Windows PATH | Call Git Bash by its full path (Part A3); confirm `where.exe oc` and `where.exe helm` work in a new window |
| `deploy.sh` fails with `Unauthorized` / `You must be logged in` | Login expired, or `KUBECONFIG` wasn't inherited by Git Bash | Re-run `oc whoami` in the same window, then launch `deploy.sh` from that window (Part B1 note) |

### K2. `extract-coninfo.sh` errors

| Error | Cause | Action |
|---|---|---|
| `File not found: TrellixSmartInstall.sh` | The SmartInstaller isn't in the current folder, or has a different name | `dir` / `ls` to check; pass the actual filename or path |
| `'bootstrap_size' not found - is this a Trellix SmartInstaller?` or `Marker '__ARCHIVE_FOLLOWS__' not found` | Not the **Linux** SmartInstaller, or the file was corrupted (for example, opened and saved in an editor, or downloaded as an HTML error page) | Download the Linux SmartInstaller again (Part B4) and copy it as a file |
| `bootstrap_size is 0`, `Payload is empty`, `Extracted coninfo.xml is empty`, or `not valid XML` | Truncated or corrupted download | Download it again |
| `Required tools not found: ...` | Run with a shell that lacks `awk`, `dd`, `base64`, etc. | On Windows, use Git Bash by full path (Part A3) |
| `Not enough disk space` | Less free space than twice the SmartInstaller's size | Free space or run from a folder on another drive |
| `exceeds the 1 MB Kubernetes Secret limit` | Unusually large `coninfo.xml` | Contact the dev team |
| `namespace argument is required` | Only one argument was given | Run `extract-coninfo.sh TrellixSmartInstall.sh <namespace>` |

### K3. `deploy.sh` errors

| Error | Cause | Action |
|---|---|---|
| `ERROR: Secret 'epo-coninfo' not found in namespace '<ns>'` | Part E not done, done in a different namespace, or the namespace name is misspelled | Do Part E with the same namespace, then rerun |
| `ERROR: Unknown option: --epo-ip` (or `--epo-user`, `--epo-password`) | Old command from build 144 or earlier | Remove all `--epo-*` flags. ePO details now come from `epo-coninfo` (Part E). |
| `ERROR: --image-repo is required in registry mode` (or `--image-digest`, `--image-pull-secret`, `--namespace`, `--cluster-name`) | A flag is missing or misspelled | Check against the table in Part H |
| `ERROR: edrf-xclient chart directory not found in bundle` | `deploy.sh` was moved away from the `edrf-xclient/` folder | Keep the extracted bundle together |
| `forbidden` mentioning `securitycontextconstraints` or `clusterroles` | Your account can't create cluster-wide resources | Part B2: a cluster-admin grants access or runs `deploy.sh` |
| `execution error ... is required` | A required chart value is empty | Run the Part G dry-run to see which value |

### K4. Pods after deployment

| Symptom | Likely cause | Action |
|---|---|---|
| `InvalidImageName` | The digest contains placeholder text (`sha256:<digest>`) or a typo | Get the real digest (Part C or H2.2) and rerun `deploy.sh` |
| `ImagePullBackOff` with `manifest unknown` / `not found` | The digest is from a different build, or the repository path is wrong | Get the digest from the correct build folder (Part C) |
| `ImagePullBackOff` with `Authentication is required` or `Unable to retrieve some image pull secrets` | The pull secret is missing, in the wrong namespace, or has the Token ID instead of the Reference Token | Fix it (Parts B3, F), then force a retry: `oc delete pod -n edrf-tests -l name=edrf-xclient-container-security` |
| `ImagePullBackOff` with a timeout or `no route to host` | The worker nodes can't reach Artifactory | Use airgap mode (Part H2), or ask the network team to open access |
| `ContainerCreating` for a long time; `oc describe pod` shows `secret "epo-coninfo" not found` | The secret was deleted or created in another namespace | Re-run `oc apply -f epo-coninfo.yml` (Part E3), then `oc delete pod -n edrf-tests <pod-name>` |
| Pods not created at all, or DaemonSet events mention `unable to validate against any security context constraint` | The ServiceAccount isn't allowed to use the custom SCC | Check `oc get scc edrf-xclient-edrf-tests` and `oc get rolebinding -n edrf-tests`. If missing, re-run `deploy.sh` with sufficient permissions (Part B2). |
| Old pod stuck in `Terminating` | Normal for up to 8 minutes (graceful agent stop) | Wait. Only if it's still there after about 10 minutes, run `oc delete pod -n edrf-tests <pod-name> --force --grace-period=0` |
| Pod `Running` but `RESTARTS` keeps increasing | The agent is crashing and being restarted | `oc describe pod -n edrf-tests <pod-name>`: under **Last State**, `Reason: OOMKilled` means it hit the 4Gi memory limit, and `Error` with an exit code means the agent died. Get its last output with `oc logs -n edrf-tests <pod-name> --previous` and send both to the dev team. |
| Pod `Running`, but not visible in the ePO console | `coninfo.xml` from the wrong ePO server, provisioning failed, ePO unreachable from the node, or the agent is waiting for approval in ePO | Check I3 and I4. Confirm the SmartInstaller came from the right ePO. Test reachability from the node with `oc debug node/<node-name>` (pods use `hostNetwork: true`). Ask your ePO admin to check the pending or unmanaged systems list. |
| `error: one of src or dest must be a local file specification` from `oc cp` | A Windows destination path starting with a drive letter (`C:\...`) | `cd` to the destination folder and use `./<file>` (Part J) |

For anything not covered above, `oc describe pod <pod-name> -n edrf-tests` and
`oc logs <pod-name> -n edrf-tests` usually show the cause directly.

---

## Part L (jumpbox) — Upgrading to a new build

An upgrade is the same process as an install:

1. Get the new bundle and extract it into a **new** folder (Part B5).
2. Registry mode: get the **new** digest (Part C). The pull secret from Part F is reused, so
   don't recreate it. Airgap mode: push the new `image.tar` and capture its digest (Part H2.2).
3. The `epo-coninfo` secret (Part E) is reused. `deploy.sh` only checks that it exists.
   Recreate it only if the agents must report to a different ePO server, or the ePO server's
   certificates changed. In that case, download a fresh SmartInstaller and redo Part E.
   `oc apply` updates the existing secret, then restart the pods with
   `oc rollout restart ds/edrf-xclient-container-security -n edrf-tests`.
4. Optionally dry-run with the new digest (Part G).
5. Run `deploy.sh` (Part H) with the new `--image-digest`.
6. Verify (Part I). Expect the rollout to take 20–25 minutes.

`deploy.sh` upgrades the existing `edrf-xclient` release in place and creates a new revision.

### Upgrading from build 50.99.0.144 or earlier

Older builds took ePO details as `deploy.sh` flags and stored them in an `epo-details`
ConfigMap and Secret. When moving to 50.99.0.150 or later:

1. **Before** running the new `deploy.sh`, create the `epo-coninfo` secret (Part E). The
   new `deploy.sh` refuses to run without it.
2. Drop `--epo-ip`, `--epo-user`, and `--epo-password` from your command.
3. The upgrade automatically removes the old `epo-details` ConfigMap and Secret, which held
   the ePO password. That's expected.

---

## Part M (jumpbox) — Rolling back

```
helm history edrf-xclient -n edrf-tests
```
Find the last good revision, then:
```
helm rollback edrf-xclient <revision-number> -n edrf-tests
```
Then verify as in Part I.

**Note:** rolling back from 50.99.0.150 or later to 50.99.0.144 or earlier brings back the old
ePO settings stored in that revision. The `epo-coninfo` secret isn't touched by a rollback.

---

## Part N (jumpbox) — Uninstalling

```
helm uninstall edrf-xclient -n edrf-tests
```

This removes everything the chart created: the DaemonSet, ServiceAccount, custom SCC,
ClusterRole, and RoleBinding. It does **not** remove what you created by hand, the namespace,
or agent data on the nodes:

```
oc delete secret epo-coninfo -n edrf-tests
oc delete secret artifactory-pull-secret -n edrf-tests
oc delete namespace edrf-tests
```
(Deleting the namespace also deletes both secrets.)

Agent data persists on each node under `/mnt/XClientLogs/`. Remove it only if you want a
completely clean node (via `oc debug node/<node-name>`, then `chroot /host`). Don't do this
if you might reinstall and want the existing agent identity kept.

---

## Quick reference — useful read-only commands

```
oc whoami                                   # who you're logged in as
oc get nodes                                # cluster nodes and status
oc get projects                             # projects you can see
oc get ns edrf-tests                        # does the namespace exist
oc get secret epo-coninfo -n edrf-tests     # ePO connection secret
oc get secret artifactory-pull-secret -n edrf-tests
oc get pods -n edrf-tests -o wide           # pods, nodes, restarts
oc get pods -n edrf-tests -o wide -w        # watch live (Ctrl+C to stop)
oc describe pod -n edrf-tests <pod-name>    # events and last state
oc logs -n edrf-tests <pod-name>            # agent logs
oc logs -n edrf-tests <pod-name> --previous # logs from before the last restart
oc exec -n edrf-tests <pod-name> -- ls -l /var/McAfee/agent/coninfo.xml
oc rollout status ds/edrf-xclient-container-security -n edrf-tests
helm list -n edrf-tests                     # release status and revision
helm status edrf-xclient -n edrf-tests      # release status summary
helm history edrf-xclient -n edrf-tests     # revision history
helm get values edrf-xclient -n edrf-tests --all   # values in use
```

---

## Appendix A — Chart values reference

`deploy.sh` sets these for you from its flags. They're listed here for anyone running Helm
directly (Appendix B) or reviewing `values.yaml`. **There are no ePO values in the chart.**
ePO details come only from the `epo-coninfo` secret.

| Field | Required | `deploy.sh` flag | Description |
|---|---|---|---|
| `image.repository` | Yes | `--image-repo` | Full image registry path |
| `image.digest` | Yes | `--image-digest` | SHA256 digest of the image |
| `image.pullPolicy` | No | none (default `IfNotPresent`) | Kubernetes image pull policy |
| `imagePullSecret` | No | `--image-pull-secret` | Name of the pull secret. Required in registry mode for a private registry. Leave empty (`""`) in airgap mode: the DaemonSet then omits `imagePullSecrets`. |
| `deploymentDetails.clusterName` | Yes | `--cluster-name` | Name identifying this cluster in ePO and telemetry |
| `deploymentDetails.deploymentMode` | Yes | none (default `Trellix Managed`) | How the deployment is managed |
| `serviceAccount.name` | No | none (default `edrf-admin`) | ServiceAccount the pods run as |
| `securityContext.capabilities` | No | none | Linux capabilities added to the container, and allowed by the custom SCC |
| `resources` | No | none (default requests 2Gi / 500m, limits 4Gi / 1000m) | Pod CPU and memory |

Required fields use Helm's `required` function, so an install or upgrade fails immediately if
any of them is empty.

---

## Appendix B — Installing with Helm directly (without `deploy.sh`)

Use this only if you can't run `deploy.sh` (for example, no bash available). You still need the
namespace (Part D), the `epo-coninfo` secret (Part E), and in registry mode the pull secret
(Part F). Run it from the bundle folder, as a single line:

```
helm upgrade --install edrf-xclient ./edrf-xclient -n edrf-tests --create-namespace --set image.repository="<image_repo>" --set image.digest="sha256:<digest>" --set imagePullSecret="artifactory-pull-secret" --set deploymentDetails.clusterName="<cluster_name>"
```

Unlike `deploy.sh`, Helm doesn't check for the `epo-coninfo` secret. If it's missing, the
install succeeds but the pods stay in `ContainerCreating`. Verify as in Part I.

---

## Appendix C — Container image, ePO provisioning, and runtime internals

Background for troubleshooting with the dev team. You don't need this to deploy.

**Image:**
- **Base image:** RapidFort curated, FIPS-enabled UBI 10 minimal
  (`artifactory.trellix.com/rapidfort/rfcurated/ubi:ubi10-minimal-fips-rfcurated`), a hardened,
  low-CVE base.
- **Contents:** `tini`, runtime dependencies, the McAfee Agent (MA) and XClient RPMs, DXL (Data
  Exchange Layer, installed from the TA package bundle: `contrib.ini`, `DXL.zip`,
  `docker-install-dxl.sh`), and `entrypoint.sh`, the in-container supervisor.
- **Built by:** the Orbit pipeline `orbit/ContainerSecurity_ImageBuild.groovy` (build context
  `rhel10/`). The customer bundle is produced by a separate, manually triggered pipeline,
  `orbit/ContainerSecurity_BundlePackage.groovy`.

**How ePO provisioning works:**
1. The ePO Linux SmartInstaller (`TrellixSmartInstall.sh`) is a shell script with an archive
   appended after a `__ARCHIVE_FOLLOWS__` marker. The archive contains a bootstrap package
   followed by `coninfo.xml`.
2. `extract-coninfo.sh` reads `bootstrap_size` from the installer, skips past the bootstrap
   package, extracts `coninfo.xml`, checks it's XML and under the 1 MB Secret limit, and writes
   it base64-encoded into `epo-coninfo.yml`, a Secret named `epo-coninfo` with the key
   `coninfo.xml`.
3. The DaemonSet mounts that key into every pod, read-only, at
   `/var/McAfee/agent/coninfo.xml`.
4. At startup, the container's `provision-container.sh` reads it and provisions the McAfee
   Agent against ePO.

**Run model.** `tini` is the container's init process and supervises `entrypoint.sh`, which:
1. provisions the McAfee Agent (MA/TA) against ePO,
2. starts MA and XClient and waits for their daemons to come up,
3. health-checks both in a loop, and exits (causing a pod restart) if either one dies, and
4. on `SIGTERM`, stops XClient and then MA gracefully before the container exits.

Step 3 is why a crashing agent shows up as a climbing `RESTARTS` count rather than a pod that
stays down (Part I6).

**DaemonSet design:**
- **Linux nodes only:** `nodeSelector: kubernetes.io/os: linux`. The workload needs eBPF, the
  CRI-O socket, and host `/sys` and `/lib/modules`, so it can't run on Windows nodes.
- **Security context:** runs as `runAsUser: 0` but **not privileged** (`privileged: false`).
  All Linux capabilities are dropped, then only the ones the eBPF sensor needs are added back
  (`SYS_ADMIN`, `BPF`, `PERFMON`, `SYS_RESOURCE`, `NET_ADMIN`, and others listed in
  `values.yaml`). The same list feeds the chart's custom SCC, so the two always match. (The
  product README still describes privileged containers and the built-in `privileged` SCC.
  The chart itself uses this reduced-privilege setup.)
- **`hostNetwork: true`:** the pod shares the node's network stack. That's why ePO
  reachability has to be tested from the node (`oc debug node/<node-name>`).
- **`hostPID: true`:** the pod shares the host's process namespace, giving XClient host-level
  process visibility. Because of this, `tini` isn't PID 1, so it runs in subreaper mode (`-s`)
  to clean up orphaned processes. **Caution:** `entrypoint.sh` matches processes by name
  (`masvc|macmnsvc|macompatsvc` and `xclient`), so a host process with a matching command line
  is also visible to it.
- **`terminationGracePeriodSeconds: 480`:** gives `entrypoint.sh` time to stop XClient
  gracefully (up to about 450 seconds, matching the host systemd unit's `TimeoutStopSec=450`).
  Without it Kubernetes would kill the pod after 30 seconds. This is why pods stay in
  `Terminating` for several minutes.
- **Identity environment variables:** `CLUSTER_NAME` and `DEPLOYMENT_MODE` (from
  `deploymentDetails`), `POD_NAME`, `NODE_NAME`, `NODE_IP`, and
  `TRELLIX_KUBERNETES_ENVIRONMENT=1`.
- **Host OS reporting:** the host's `/etc/os-release` is mounted at `/host-etc-os-release`, so
  the agent reports the node's OS, not the container's.
- **CRI-O socket:** mounted with `type: Socket`, so Kubernetes checks that the host path really
  is a socket.
- **Persistent data:** XClient data, the decryption key, MA logs, and TA config are stored on
  each node under `/mnt/XClientLogs/` (`xclient`, `fexgt`, `ma-logs`).

---

## Appendix D — Plain Kubernetes and minikube

This guide targets OpenShift. For other platforms:

**Kubernetes 1.23+ without Helm:** use `02-create-sa-k8s.yml` from the product source, which
applies the required Pod Security Admission (PSA) labels in place of an SCC. The
`epo-coninfo` secret is still required. `extract-coninfo.sh` and `kubectl apply -f
epo-coninfo.yml` work the same way.

**Local testing on minikube:** use `edrf-xclient-minikube.yml`, a single flattened manifest
(namespace, ServiceAccount, ConfigMap, Secret, DaemonSet) from the product source:
```bash
minikube start --container-runtime=cri-o   # the DaemonSet mounts the CRI-O socket
minikube image load <image_name:tag>       # no pull secret is used
# edit the placeholders in the manifest (ePO details and <image_name:tag>) first
# CLUSTER_NAME and DEPLOYMENT_MODE are hardcoded in the manifest; edit them if needed
kubectl apply -f edrf-xclient-minikube.yml
```
The product README's minikube manifest still uses ePO IP/user/password placeholders. Check
with the dev team whether it has been updated for the `coninfo.xml` method.

> **minikube only: mount the BPF filesystem first.** XClient pins its eBPF maps under
> `/sys/fs/bpf`, which must be a mounted BPF filesystem. Real cluster nodes mount it
> automatically. A minikube node (for example on WSL) doesn't, and the loader fails with
> `libbpf: ... is not on BPF FS ... failed to create: Invalid argument(-22)`. Mount it, then
> restart the pod:
> ```bash
> minikube ssh -- "stat -f -c %T /sys/fs/bpf | grep -qi bpf && echo 'already bpffs' || sudo mount -t bpf bpf /sys/fs/bpf"
> minikube ssh -- "mount | grep /sys/fs/bpf"   # expect: bpf on /sys/fs/bpf type bpf
> kubectl delete pod -l name=edrf-xclient-container-security
> ```
> The mount is lost on `minikube stop` or `delete`, so re-run it after each restart.

---

## Appendix E — Linux jumpbox copy-paste command sequence (registry mode)

The complete Linux sequence as one block, in the form used for the build 50.99.0.150
deployment. Save it as a `.txt` file, replace the placeholders, and paste it block by block
into a single terminal.

**Before you start:**
- `oc` and `helm` (v3) installed, and you're logged in (Parts A and B1).
- The namespace and the `artifactory-pull-secret` already exist (Parts D and F).
- You have the Linux SmartInstaller from ePO (Part B4) and the image digest for your build
  (Part C).
- Replace **every** `<...>` placeholder. A leftover `sha256:<digest>` makes every pod fail
  with `InvalidImageName`.
- Never commit your filled-in copy, `TrellixSmartInstall.sh`, or `epo-coninfo.yml` to git.

```text
------------------------------------------------------------------------------------
STEP 0 - Set variables (edit every value before running)
------------------------------------------------------------------------------------

NS=<namespace>
CLUSTER=<cluster_name>
REPO=artifactory.trellix.com/edrf-xclient-container-security/trellix-edrf
PULL_SECRET=artifactory-pull-secret
VERSION=<bundle_version>
BUNDLE_DIR=~/edrf-deploy/bundle-$VERSION

# Real digest for this build from Artifactory (Part C):
# sha256: followed by 64 hex characters, no < or >
DIGEST=sha256:<64_char_digest>


------------------------------------------------------------------------------------
STEP 1 - Pre-checks
------------------------------------------------------------------------------------

oc whoami
oc get ns $NS
oc get secret $PULL_SECRET -n $NS
oc auth can-i create securitycontextconstraints
oc auth can-i create rolebindings -n $NS
oc auth can-i create secrets -n $NS

# Expected: your username, namespace Active, the pull secret listed, "yes" three times.
# ("not namespace scoped" warning on the SCC check is normal.)


------------------------------------------------------------------------------------
STEP 2 - Extract the bundle into a new folder
------------------------------------------------------------------------------------

mkdir -p $BUNDLE_DIR
mv /path/to/edrf-xclient-bundle-$VERSION.tar.gz $BUNDLE_DIR/
cd $BUNDLE_DIR
tar -xzf edrf-xclient-bundle-$VERSION.tar.gz
ls -la

# Expected: image.tar, edrf-xclient/, deploy.sh, extract-coninfo.sh
# If tar created a subfolder instead, go into it:
#   cd edrf-xclient-bundle-$VERSION


------------------------------------------------------------------------------------
STEP 3 - Create the epo-coninfo secret
------------------------------------------------------------------------------------

# 3a. Download the LINUX SmartInstaller (TrellixSmartInstall.sh) from the ePO console
#     (Menu -> Systems -> System Tree -> New Systems -> Create URL for client-side agent
#     download -> Non-Windows (Linux) agent version -> All Agent Handlers -> OK, then open
#     the URL and download the file; see Part B4) and copy it into this folder.
#     Do not open it in an editor.
ls -la TrellixSmartInstall.sh

# 3b. Generate epo-coninfo.yml
bash extract-coninfo.sh TrellixSmartInstall.sh $NS

# Expected: [1/5] ... [5/5], then "Done." and "Output file : epo-coninfo.yml"

# 3c. Create the secret on the cluster and verify
oc apply -f epo-coninfo.yml
oc get secret epo-coninfo -n $NS

# Expected: secret/epo-coninfo created, then one row "epo-coninfo   Opaque   1"


------------------------------------------------------------------------------------
STEP 4 - Dry-run (changes nothing on the cluster)
------------------------------------------------------------------------------------

# Safety check: prints STOP if DIGEST still has the placeholder
echo "$DIGEST" | grep -Eq '^sha256:[0-9a-f]{64}$' && echo "Digest OK" || echo "STOP: set DIGEST in Step 0"

helm upgrade --install edrf-xclient ./edrf-xclient -n $NS \
  --set image.repository="$REPO" \
  --set image.digest="$DIGEST" \
  --set imagePullSecret="$PULL_SECRET" \
  --set deploymentDetails.clusterName="$CLUSTER" \
  --dry-run=client

# Expected: STATUS: pending-upgrade, DESCRIPTION: Dry run complete, no "required" errors.

# Check only the image line (must end in your real 64-character digest):
helm upgrade --install edrf-xclient ./edrf-xclient -n $NS \
  --set image.repository="$REPO" \
  --set image.digest="$DIGEST" \
  --set imagePullSecret="$PULL_SECRET" \
  --set deploymentDetails.clusterName="$CLUSTER" \
  --dry-run=client | grep "image:"


------------------------------------------------------------------------------------
STEP 5 - Deploy
------------------------------------------------------------------------------------

chmod +x deploy.sh
./deploy.sh \
  --namespace $NS \
  --cluster-name $CLUSTER \
  --image-repo $REPO \
  --image-digest $DIGEST \
  --image-pull-secret $PULL_SECRET

# Expected:
#   ==> Mode: registry (image: ...@sha256:...)
#   ==> Installing Helm chart
#   STATUS: deployed
#   ==> Deployment complete


------------------------------------------------------------------------------------
STEP 6 - Verify
------------------------------------------------------------------------------------

helm list -n $NS

# Can take 20-25 minutes on an upgrade (old pods take up to 8 minutes each to stop)
oc rollout status ds/edrf-xclient-container-security -n $NS

oc get pods -n $NS -o wide

# Expected: one pod per worker node, all 1/1 Running, RESTARTS 0

# Pick the first agent pod
POD=$(oc get pods -n $NS -l name=edrf-xclient-container-security -o jsonpath='{.items[0].metadata.name}')
echo $POD

oc exec -n $NS $POD -- ls -l /var/McAfee/agent/coninfo.xml
oc logs -n $NS $POD

# Expected: coninfo.xml present; logs show the agent provisioning against ePO
# with no repeating errors. Then confirm the nodes in the ePO console.


------------------------------------------------------------------------------------
STEP 7 - Clean up local sensitive files
------------------------------------------------------------------------------------

rm -f epo-coninfo.yml


------------------------------------------------------------------------------------
USEFUL COMMANDS (any time)
------------------------------------------------------------------------------------

oc get pods -n $NS -o wide -w                     # watch pods live (Ctrl+C to stop)
oc describe pod -n $NS $POD                       # events and last state
oc logs -n $NS $POD --previous                    # logs before the last restart
oc exec -it -n $NS $POD -- /bin/bash              # shell into the pod
helm history edrf-xclient -n $NS                  # revision history
helm rollback edrf-xclient <revision> -n $NS      # roll back to a revision

# Copy a log file out of a pod (inside the pod, find it with: ls -lrth /tmp)
oc cp -n $NS $POD:/tmp/<log-file> ./<log-file>
```

The variables only last for the current terminal. If you open a new terminal, re-run Step 0
first.
