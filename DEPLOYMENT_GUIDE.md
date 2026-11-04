# eDRF XClient — OpenShift Deployment Guide

**Audience:** engineers deploying eDRF XClient for the first time. This guide doesn't assume
`oc`, `helm`, Git Bash, or Artifactory access are already set up. Follow the parts in order,
because later steps depend on resources that earlier steps create.

**How deployment works now:** you receive a **bundle** (`edrf-xclient-bundle-<version>.tar.gz`)
containing the container image, the Helm chart, and a script, `deploy.sh`. **`deploy.sh` does
the install.** It runs `helm upgrade --install` for you, so the same command handles both a
fresh install and an upgrade of an existing release. Don't run `helm install` yourself.

This guide covers three shells: **PowerShell**, **Command Prompt (`cmd.exe`)**, and
**Linux/Mac (bash/zsh)**.

- Every `oc` and `helm` command is plain text that runs the same in all three shells, so it
  appears once with no shell label.
- A few steps really do differ by shell (setting an environment variable, extracting an
  archive, editing PATH, running `deploy.sh`). These appear as labeled blocks: **PowerShell**,
  **cmd.exe**, **Linux/Mac**. Use the one that matches your terminal.
- **`deploy.sh` is a bash script.** It doesn't run directly in `cmd.exe` or PowerShell. On
  Windows you need **Git for Windows**, which provides Git Bash (Part A3). Linux and Mac
  already have bash.
- PowerShell aliases `curl` to `Invoke-WebRequest`, so use `curl.exe` wherever `curl` appears
  below in PowerShell.

**Read Part 0 first.** It determines which machine you run the rest of this guide from, and
which deployment mode to use.

> **Namespace note:** the examples below use `edrf-tests` as a placeholder. This name is
> expected to change in a future release. Use whatever namespace your team has standardized on.

> **Credentials note:** never commit real passwords, tokens, or a `values.yaml` containing
> them to any git repository.

---

## Quick glossary

| Term | What it means here |
|---|---|
| **OpenShift / `oc`** | Red Hat's Kubernetes distribution. `oc` is its CLI (a superset of `kubectl`). |
| **Project / Namespace** | An isolated area in the cluster. "Project" is OpenShift's name for a Kubernetes Namespace. |
| **Helm / chart / release** | Helm is a package manager for Kubernetes. A **chart** is the package (the `edrf-xclient/` folder in the bundle). A **release** is one installed instance of that chart in a namespace (named `edrf-xclient`). |
| **Bundle** | `edrf-xclient-bundle-<version>.tar.gz`, the deliverable containing `image.tar`, the `edrf-xclient/` chart, and `deploy.sh`. |
| **`deploy.sh`** | The bash script in the bundle that performs the install or upgrade. |
| **Git Bash** | A bash shell for Windows, installed with Git for Windows. Required on Windows to run `deploy.sh`. |
| **DaemonSet** | A workload that runs exactly one pod on every node. This is how the chart deploys the XClient agent. |
| **Secret / ConfigMap** | Kubernetes objects for sensitive values (Secret) and non-sensitive config (ConfigMap). |
| **ServiceAccount (SA)** | The identity a pod runs as (`edrf-admin` here). |
| **SCC** | Security Context Constraint, OpenShift's permission gate for pods that need host access. This chart **creates its own reduced-privilege SCC** (`edrf-xclient-<namespace>`) instead of using the built-in `privileged` SCC. Creating an SCC needs cluster-level permission (Part B2). |
| **Artifactory** | Trellix's private container registry (`artifactory.trellix.com`), where the agent image is hosted. |
| **ePO** | Trellix ePolicy Orchestrator, the management server the XClient agent reports to. |

---

## Part 0 — Choose your machine and deployment mode

### 0.1 Which machine does what

This deployment touches two networks, and many corporate environments don't let one machine
reach both:

1. The **OpenShift cluster API**, needed for every `oc`, `helm`, and `deploy.sh` command.
2. **Artifactory**, on the general corporate network. You only need this for Part C (image
   digest lookup) in registry mode.

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
  - The only values to carry between machines are the two short strings Part C produces
    (image repository and digest).

### 0.2 Which deployment mode

`deploy.sh` supports two modes:

| Mode | Use when | You must provide |
|---|---|---|
| **Registry** (default) | The cluster's worker nodes can pull images from Artifactory (or an internal mirror) | A pull secret (Part E), plus the image repository and digest (Part C) |
| **Airgap** | The cluster can't reach any external registry | Docker on the jumpbox, plus a one-time registry route change by a cluster-admin (Part G2). No pull secret, repository, or digest. |

If you aren't sure, start with **registry mode**. If pods later fail with `ImagePullBackOff`
and a network timeout (not an authentication error), the nodes can't reach Artifactory and you
need airgap mode.

Every part below is tagged **(jumpbox)**, **(any machine)**, **(Artifactory-reachable
machine)**, **(registry mode)**, or **(airgap mode)**.

---

## Quick start — Full command sequence (Windows or Linux jumpbox)

This is the complete install/upgrade sequence on one page, for when the tools are installed
and you're logged in. It is the same sequence used to upgrade to build 50.99.0.144 in
registry mode. If anything here is unfamiliar or fails, follow the detailed parts below
(Parts A–H).

Replace every `<...>` placeholder with your own values (Parts B3, B4, C).

| | Windows jumpbox | Linux jumpbox |
|---|---|---|
| Extra tool needed | **Git for Windows** (Part A3), because `deploy.sh` is a bash script | None, because bash is built in |
| Shell used below | `cmd.exe` | bash |

### Step 1 — Pre-checks

The same commands on both:
```
oc whoami
oc get secret artifactory-pull-secret -n <namespace>
oc auth can-i create securitycontextconstraints
oc auth can-i create clusterroles
oc auth can-i create rolebindings -n <namespace>
```
**Expected:** your identity, one secret row of type `kubernetes.io/dockerconfigjson`, and `yes`
three times. If the secret doesn't exist, create it first (Parts D and E).

### Step 2 — Create a folder and extract the bundle

**Windows (`cmd.exe`):**
```cmd
mkdir C:\edrf-deploy\bundle
move C:\path\to\edrf-xclient-bundle-<version>.tar.gz C:\edrf-deploy\bundle\
cd /d C:\edrf-deploy\bundle
tar -xzf edrf-xclient-bundle-<version>.tar.gz
dir
```
**Linux:**
```bash
mkdir -p ~/edrf-deploy/bundle
mv /path/to/edrf-xclient-bundle-<version>.tar.gz ~/edrf-deploy/bundle/
cd ~/edrf-deploy/bundle
tar -xzf edrf-xclient-bundle-<version>.tar.gz
ls -la
```
**Expected:** `image.tar`, `edrf-xclient/`, `deploy.sh`.

### Step 3 — Update `values.yaml` and dry-run

Fill in `edrf-xclient/values.yaml` (ePO IP, user, password, image repository and digest, pull
secret name, cluster name; see Appendix A) using any text editor. On Windows that can be
`notepad edrf-xclient\values.yaml`. On Linux, use `vi edrf-xclient/values.yaml` or `nano`.
Don't commit this file anywhere, because it contains the ePO password.

Then dry-run, using the same command on both:
```
helm upgrade --install edrf-xclient ./edrf-xclient -n <namespace> --dry-run=client
```
**Expected:** rendered manifests, `STATUS: pending-upgrade`, `DESCRIPTION: Dry run complete`,
and no `required` errors. Nothing changes on the cluster.

### Step 4 — Install or upgrade with `deploy.sh`

**Windows (`cmd.exe`, one line):**
```cmd
"C:\Program Files\Git\bin\bash.exe" ./deploy.sh --namespace <namespace> --epo-ip <epo_ip> --epo-user <epo_user> --epo-password "<epo_password>" --cluster-name <cluster_name> --image-repo <image_repo> --image-digest sha256:<digest> --image-pull-secret artifactory-pull-secret
```
**Linux:**
```bash
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

- **Windows:** don't type `./deploy.sh` on its own in `cmd.exe`. Keep the command on one
  line, and use double quotes around the password.
- **Linux:** `chmod +x` is needed because the extracted script may not be executable
  (otherwise you get `Permission denied`). `bash deploy.sh ...` also works. Use single quotes
  around the password.

**Expected:**
```
==> Mode: registry (image: <image_repo>@sha256:<digest>)
==> Installing Helm chart
Release "edrf-xclient" has been upgraded. Happy Helming!
STATUS: deployed
REVISION: <n>
DESCRIPTION: Upgrade complete
==> Deployment complete. Check pod status:
```

### Step 5 — Verify

The same commands on both:
```
helm list -n <namespace>
oc rollout status ds/edrf-xclient-container-security -n <namespace>
oc get pods -n <namespace> -o wide -w
```
**Expected:** release `deployed` with a new revision, rollout `successfully rolled out`, and
one `1/1 Running` pod per Linux worker node with `RESTARTS 0`. An upgrade rollout can take
20–25 minutes (Part H1). Press Ctrl+C to stop `-w`.

### Windows vs. Linux at a glance

| Step | Windows (`cmd.exe`) | Linux |
|---|---|---|
| Tool prerequisite | Install Git for Windows | Nothing extra |
| Create folder | `mkdir C:\edrf-deploy\bundle` | `mkdir -p ~/edrf-deploy/bundle` |
| Move bundle | `move <src> C:\edrf-deploy\bundle\` | `mv <src> ~/edrf-deploy/bundle/` |
| Change folder | `cd /d C:\edrf-deploy\bundle` | `cd ~/edrf-deploy/bundle` |
| List files | `dir` | `ls -la` |
| Edit values | `notepad edrf-xclient\values.yaml` | `vi edrf-xclient/values.yaml` |
| Run script | `"C:\Program Files\Git\bin\bash.exe" ./deploy.sh ...` (one line) | `chmod +x deploy.sh` then `./deploy.sh ...` |
| Password quoting | `"<epo_password>"` | `'<epo_password>'` |
| `oc` / `helm` / `tar` commands | Same | Same |

---

## Part A (jumpbox) — Install the tools

| Tool | Needed for | Windows | Linux/Mac |
|---|---|---|---|
| `oc` | Everything | Install (A1) | Install (A1) |
| `helm` (v3) | `deploy.sh` calls it internally, plus the dry-run | Install (A2) | Install (A2) |
| Git for Windows (Git Bash) | Running `deploy.sh` | **Install (A3)** | Not needed (bash is built in) |
| `curl`, `tar` | Digest lookup, extracting the bundle | Built into Windows 10/11 | Built in |
| Docker CLI | **Airgap mode only** | Install (A4) | Install (A4) |

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

`deploy.sh` is a bash script. Running it straight from `cmd.exe` fails with
`'.' is not recognized as an internal or external command`. Git for Windows provides
**Git Bash**, which runs it.

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

### A4. Install the Docker CLI (airgap mode only)

Skip this in registry mode. In airgap mode, `deploy.sh` uses Docker to load `image.tar` and
push it to the cluster's internal registry.

- **Windows:** Docker Desktop (https://docs.docker.com/desktop/setup/install/windows-install/),
  or whatever your organization approves.
- **Linux:** your distribution's `docker` package or Docker Engine.
- **Mac:** Docker Desktop.

**Verify** (the daemon must be running, not just the CLI installed):
```
docker version
```

**Expected:** both a `Client` and a `Server` section. If only `Client` appears, start Docker
Desktop or the Docker service.

---

## Part B — Collect access and credentials

B1, B2, and B5 run on the jumpbox. B3 and B4 only need a browser and can run from any machine.

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

> **Windows and `deploy.sh`:** run `deploy.sh` from the **same** `cmd.exe` or PowerShell
> window where you logged in or set `KUBECONFIG` (Part G1 shows how). Git Bash launched from
> that window inherits your PATH and login. A Git Bash window opened separately from the
> Start menu won't have a `KUBECONFIG` set with `set` or `$env:` unless you made it permanent.

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

This chart creates namespaced resources (ServiceAccount, RoleBinding, Secret, ConfigMap,
DaemonSet) **and cluster-wide resources**: a custom **SecurityContextConstraints** and a
**ClusterRole**. Creating cluster-wide resources usually needs cluster-admin rights.

Check all three before going further:
```
oc auth can-i create rolebindings -n edrf-tests
oc auth can-i create clusterroles
oc auth can-i create securitycontextconstraints
```

**Expected:** `yes` for all three.

If any of them return `no`, stop here. Ask your OpenShift platform or cluster-admin team to
grant the permission, or to run `deploy.sh` (Part G) for you. Without it the install fails
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

### B4 (any machine, information only). ePO server details

The agent needs:
- ePO IP or hostname
- ePO username
- ePO password

Get these from the team that manages ePO for your environment. A dedicated service account is
better than a personal login.

You also need a **cluster name**, the label this cluster will have in ePO. Get the cluster's
infrastructure name with:
```
oc get infrastructure cluster -o jsonpath='{.status.infrastructureName}'
```
This returns something like `mycluster-ab1cd`, where the part before the random suffix
(`mycluster`) is usually the value to use. Node names follow the same pattern
(`mycluster-ab1cd-worker-a-xyz12`). If other clusters already report to your ePO, follow their
naming convention.

### B5 (jumpbox). Get and extract the bundle

Get `edrf-xclient-bundle-<version>.tar.gz` from your build or release team. Copy it to the
jumpbox, then extract it into a **new, empty folder** so it doesn't mix with older chart
copies. `tar` is built into Windows 10/11, so the extract command is the same everywhere.

**PowerShell:**
```powershell
mkdir C:\edrf-deploy\bundle
cd C:\edrf-deploy\bundle
tar -xzf C:\path\to\edrf-xclient-bundle-<version>.tar.gz
```
**cmd.exe:**
```cmd
mkdir C:\edrf-deploy\bundle
cd /d C:\edrf-deploy\bundle
tar -xzf C:\path\to\edrf-xclient-bundle-<version>.tar.gz
```
**Linux/Mac:**
```bash
mkdir -p ~/edrf-deploy/bundle
cd ~/edrf-deploy/bundle
tar -xzf /path/to/edrf-xclient-bundle-<version>.tar.gz
```

**Verify:**
```
dir          (Windows)
ls -la       (Linux/Mac)
```

**Expected:** three items:
```text
image.tar        # container image (only used in airgap mode)
edrf-xclient/    # Helm chart
deploy.sh        # install script
```

The chart layout, for reference:
```text
edrf-xclient/
├── Chart.yaml
├── values.yaml                  # default values (deploy.sh flags override these)
└── templates/
    ├── 01-serviceaccount.yaml   # ServiceAccount, custom SCC, ClusterRole, RoleBinding
    ├── 02-epo-details.yaml      # ConfigMap + Secret holding ePO connection details
    └── 03-deploy-edrf.yaml      # the DaemonSet that runs XClient on every Linux node
```

**You don't need to edit `values.yaml`.** `deploy.sh` passes every required value as a flag.
If you do edit it (for example, to run the dry-run in Part F without `--set` flags), keep
that copy out of git, because it would contain the ePO password.

Run all later commands from this bundle folder.

---

## Part C (Artifactory-reachable machine, registry mode) — Get the image repository and digest

Skip this part in airgap mode.

If Part 0 showed the jumpbox has no internet access, run this part on your workstation. You
only need `curl` and your Artifactory credentials.

You need two values for `deploy.sh`:
- `--image-repo`, e.g. `artifactory.trellix.com/edrf-xclient-container-security/trellix-edrf`
- `--image-digest`, e.g. `sha256:<64 hex characters>`

A typical Artifactory browse URL for a build looks like
`https://artifactory.trellix.com/artifactory/edrf-xclient-container-security/trellix-edrf/114/`,
where `114` is the build number. Use the build number that matches your bundle.

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

**If the response isn't clean JSON**, re-run with `-v` instead of `-s` and check the HTTP
status:
- **`200 OK` with `"children": [...]`** means `<build>` is a folder and `manifest.json` isn't
  the real filename. List the folder (remove `manifest.json` from the URL) and retry with the
  real name.
- **`401` / `403`** means the username or token is wrong, expired, or lacks read access. Check
  you used the **Reference Token** (Part B3).
- **`404`** means the repo key or build number is wrong. Confirm the path with your build team.
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

Create the namespace **before** the pull secret, because the secret is created inside it.

```
oc new-project edrf-tests
```
If it already exists:
```
oc project edrf-tests
```
**Verify:**
```
oc get ns edrf-tests
```

**Expected:** one row with `STATUS Active`.

---

## Part E (jumpbox, registry mode) — Create the Artifactory pull secret

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

## Part F (jumpbox) — Dry-run the chart (recommended)

A dry-run catches missing or wrong values before anything changes on the cluster. Run it from
the bundle folder with the same values you'll give `deploy.sh`. This is a single line and works
in all three shells:

```
helm upgrade --install edrf-xclient ./edrf-xclient -n edrf-tests --set epo.ip="<epo_ip>" --set epo.user="<epo_user>" --set epo.password="<epo_password>" --set deploymentDetails.clusterName="<cluster_name>" --set image.repository="<image_repo>" --set image.digest="sha256:<digest>" --set imagePullSecret="artifactory-pull-secret" --dry-run=client
```

**Expected:** a long list of rendered manifests ending without errors. The output includes
these lines, which are **normal for a dry-run**. Nothing was changed on the cluster:
```
Release "edrf-xclient" has been upgraded. Happy Helming!
STATUS: pending-upgrade
DESCRIPTION: Dry run complete
```
The manifests should include a `SecurityContextConstraints` named `edrf-xclient-edrf-tests`, a
`ClusterRole`, a `RoleBinding`, and a `DaemonSet` whose `image:` line ends in your digest.

An `execution error ... is required` means a value is missing or misspelled. Fix it and re-run.

---

## Part G (jumpbox) — Run `deploy.sh`

### G1. Registry mode

Run from the bundle folder (the one containing `deploy.sh`). Replace every `<...>`
placeholder. **Pick the block for your shell.**

**cmd.exe** (one line; launches Git Bash from this window so it inherits your login):
```cmd
"C:\Program Files\Git\bin\bash.exe" ./deploy.sh --namespace edrf-tests --epo-ip <epo_ip> --epo-user <epo_user> --epo-password "<epo_password>" --cluster-name <cluster_name> --image-repo <image_repo> --image-digest sha256:<digest> --image-pull-secret artifactory-pull-secret
```

**PowerShell** (one line; note the leading `&`):
```powershell
& "C:\Program Files\Git\bin\bash.exe" ./deploy.sh --namespace edrf-tests --epo-ip <epo_ip> --epo-user <epo_user> --epo-password "<epo_password>" --cluster-name <cluster_name> --image-repo <image_repo> --image-digest sha256:<digest> --image-pull-secret artifactory-pull-secret
```

**Linux/Mac, or inside a Git Bash window:**
```bash
chmod +x deploy.sh
./deploy.sh \
  --namespace edrf-tests \
  --epo-ip <epo_ip> \
  --epo-user <epo_user> \
  --epo-password '<epo_password>' \
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
- In `cmd.exe`, quote the password with **double quotes**. In bash, use **single quotes**
  (so characters such as `$` and `!` aren't interpreted).
- To see every flag the script accepts, run it with `--help`, e.g.
  `"C:\Program Files\Git\bin\bash.exe" ./deploy.sh --help`.

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
`REVISION: 1` appear instead. Both are success.

### G2. Airgap mode

In airgap mode the script loads `image.tar`, finds the cluster's internal registry route, logs
in with `oc registry login`, pushes the image, captures its digest, and installs, so you don't
pass `--image-repo`, `--image-digest`, or `--image-pull-secret`.

**Before the first airgap run:**
1. Docker is installed and running on the jumpbox (Part A4).
2. A cluster-admin exposes the internal registry route (one-time, cluster-wide):
   ```
   oc patch configs.imageregistry.operator.openshift.io/cluster --patch "{\"spec\":{\"defaultRoute\":true}}" --type=merge
   ```
   (The escaped double quotes work in `cmd.exe` and PowerShell. In bash you can use
   `--patch '{"spec":{"defaultRoute":true}}'`.)

**cmd.exe:**
```cmd
"C:\Program Files\Git\bin\bash.exe" ./deploy.sh --namespace edrf-tests --epo-ip <epo_ip> --epo-user <epo_user> --epo-password "<epo_password>" --cluster-name <cluster_name> --mode airgap
```
**PowerShell:**
```powershell
& "C:\Program Files\Git\bin\bash.exe" ./deploy.sh --namespace edrf-tests --epo-ip <epo_ip> --epo-user <epo_user> --epo-password "<epo_password>" --cluster-name <cluster_name> --mode airgap
```
**Linux/Mac or Git Bash:**
```bash
./deploy.sh --namespace edrf-tests --epo-ip <epo_ip> --epo-user <epo_user> --epo-password '<epo_password>' --cluster-name <cluster_name> --mode airgap
```

---

## Part H (jumpbox) — Verify the deployment

### H1. Wait for the rollout to finish

```
oc rollout status ds/edrf-xclient-container-security -n edrf-tests
```

**Expected:** `daemon set "edrf-xclient-container-security" successfully rolled out`.

**On an upgrade, this can take 20–25 minutes.** Old pods are replaced one node at a time, and
each old pod is given up to **8 minutes (480 seconds)** to stop the agent gracefully. A pod
in `Terminating` for several minutes is normal. Don't force-delete it unless it's been stuck
for more than about 10 minutes.

### H2. Check pods and the release

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

### H3. Check the agent logs

```
oc logs -n edrf-tests <pod-name>
```
Or for all pods at once:
```
oc logs -n edrf-tests -l name=edrf-xclient-container-security --tail=100
```

**Expected:** the McAfee Agent provisions against ePO, then MA and XClient start, with no
repeating errors. `Running` on its own only means the container started, not that the agent
is reporting.

### H4. Confirm in ePO

Check that every worker node appears in the ePO console with a recent last-communication time.

### H5. Recheck restarts after about an hour

```
oc get pods -n edrf-tests
```

`RESTARTS` should still be `0`. If it's climbing, the agent is crashing and Kubernetes is
restarting it, even though the pod looks `Running` most of the time. See Part J.

---

## Part I (jumpbox) — Collect agent logs from a pod

Use this when the dev or support team asks for agent logs.

**1. Open a shell inside the pod:**
```
oc exec -it -n edrf-tests <pod-name> -- /bin/bash
```
(Use `/bin/sh` if bash isn't in the image.)

**2. Inside the pod, find the logs and check the agent process:**
```bash
ps -o pid,ppid,cmd -C xclient
ls -lrth /tmp
ls -lrth /var/lib/Trellix/XClient/
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

## Part J (jumpbox) — Troubleshooting

| Symptom | Likely cause | Action |
|---|---|---|
| `'.' is not recognized as an internal or external command` | `./deploy.sh` was typed directly in `cmd.exe` or PowerShell | Run it through Git Bash (Part G1). Install Git for Windows first if needed (Part A3). |
| `'--namespace' is not recognized...` (one error per line) | A multi-line command with `\` continuation was pasted into `cmd.exe` | Use the one-line `cmd.exe` form in Part G1 |
| `The system cannot find the path specified` for `bash.exe` | Git for Windows isn't installed, or is installed somewhere else | Part A3: install it, or find the real path with `where.exe git` |
| `oc: command not found` or `helm: command not found` from inside `deploy.sh` | bash can't see your Windows tools. Common causes are using WSL (plain `bash`) instead of Git Bash, or `oc`/`helm` not on the Windows PATH | Call Git Bash by its full path (Part A3); confirm `where.exe oc` and `where.exe helm` work in a new window (Parts A1–A2) |
| `deploy.sh` fails with `Unauthorized` / `You must be logged in` | Login expired, or `KUBECONFIG` wasn't inherited by Git Bash | Re-run `oc whoami` in the same window, then launch `deploy.sh` from that window (Part B1 note) |
| `forbidden` mentioning `securitycontextconstraints` or `clusterroles` | Your account can't create cluster-wide resources | Part B2: a cluster-admin grants access or runs `deploy.sh` |
| `execution error ... is required` | A required value is missing or misspelled | Check every `deploy.sh` flag; run the Part F dry-run to see which value |
| `namespace "<ns>" not found` when creating a secret | The namespace doesn't exist yet | Part D, then re-run the secret command and verify with `oc get secret -n <ns>` |
| `ImagePullBackOff` with `Authentication is required` or `Unable to retrieve some image pull secrets` | The pull secret is missing, in the wrong namespace, or has the Token ID instead of the Reference Token | Fix it (Parts B3, E), then force a retry: `oc delete pod -n edrf-tests -l name=edrf-xclient-container-security` |
| `ImagePullBackOff` with a timeout or `no route to host` | The worker nodes can't reach Artifactory | Use airgap mode (Part G2), or ask the network team to open access |
| Pods not created at all, or the DaemonSet events mention `unable to validate against any security context constraint` | The ServiceAccount isn't allowed to use the custom SCC | Check `oc get scc edrf-xclient-edrf-tests` and `oc get rolebinding -n edrf-tests`. If missing, re-run `deploy.sh` with sufficient permissions (Part B2). |
| Old pod stuck in `Terminating` | Normal for up to 8 minutes (graceful agent stop) | Wait. Only if it's still there after about 10 minutes, run `oc delete pod -n edrf-tests <pod-name> --force --grace-period=0` |
| Pod `Running` but `RESTARTS` keeps increasing | The agent is crashing and being restarted | `oc describe pod -n edrf-tests <pod-name>`: under **Last State**, `Reason: OOMKilled` means it hit the 4Gi memory limit, and `Error` with an exit code means the agent died. Get its last output with `oc logs -n edrf-tests <pod-name> --previous` and send both to the dev team. |
| `CrashLoopBackOff`, logs mention ePO connection failures | Wrong ePO IP, user, or password, or the node can't reach ePO | Re-check Part B4 values; test reachability from the node with `oc debug node/<node-name>` (pods use `hostNetwork: true`) |
| Pod `Running`, but not visible in the ePO console | Wrong ePO credentials, ePO unreachable from the node, or the agent is waiting for approval in ePO | Check `oc logs`; ask your ePO admin to check the pending or unmanaged systems list |
| `error: one of src or dest must be a local file specification` from `oc cp` | A Windows destination path starting with a drive letter (`C:\...`) | `cd` to the destination folder and use `./<file>` (Part I) |

For anything not covered above, `oc describe pod <pod-name> -n edrf-tests` and
`oc logs <pod-name> -n edrf-tests` usually show the cause directly.

---

## Part K (jumpbox) — Upgrading to a new build

An upgrade is the same process as an install:

1. Get the new bundle and extract it into a **new** folder (Part B5).
2. Registry mode: get the new digest (Part C). The pull secret from Part E is reused, so don't
   recreate it.
3. Optionally dry-run with the new digest (Part F).
4. Run the same `deploy.sh` command (Part G) with the new `--image-digest`.
5. Verify (Part H). Expect the rollout to take 20–25 minutes.

`deploy.sh` upgrades the existing `edrf-xclient` release in place and creates a new revision.

---

## Part L (jumpbox) — Rolling back

```
helm history edrf-xclient -n edrf-tests
```
Find the last good revision, then:
```
helm rollback edrf-xclient <revision-number> -n edrf-tests
```
Then verify as in Part H.

---

## Part M (jumpbox) — Uninstalling

```
helm uninstall edrf-xclient -n edrf-tests
```

This removes everything the chart created: the DaemonSet, ServiceAccount, custom SCC,
ClusterRole, RoleBinding, ConfigMap, and Secret. It does **not** remove the namespace, the pull
secret you created in Part E, or agent data on the nodes:

```
oc delete secret artifactory-pull-secret -n edrf-tests
oc delete namespace edrf-tests
```

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
oc get pods -n edrf-tests -o wide           # pods, nodes, restarts
oc get pods -n edrf-tests -o wide -w        # watch live (Ctrl+C to stop)
oc describe pod -n edrf-tests <pod-name>    # events and last state
oc logs -n edrf-tests <pod-name>            # agent logs
oc logs -n edrf-tests <pod-name> --previous # logs from before the last restart
oc rollout status ds/edrf-xclient-container-security -n edrf-tests
helm list -n edrf-tests                     # release status and revision
helm history edrf-xclient -n edrf-tests     # revision history
helm get values edrf-xclient -n edrf-tests --all   # values in use (includes ePO password)
helm status edrf-xclient -n edrf-tests      # release status summary
```

---

## Appendix A — Chart values reference

`deploy.sh` sets these for you from its flags. They're listed here for anyone running Helm
directly (Appendix B) or reviewing `values.yaml`.

| Field | Required | `deploy.sh` flag | Description |
|---|---|---|---|
| `epo.ip` | Yes | `--epo-ip` | IP address or hostname of the ePO server |
| `epo.user` | Yes | `--epo-user` | ePO admin username |
| `epo.password` | Yes* | `--epo-password` | ePO admin password. *Not required if `epo.existingSecret` is set. |
| `epo.existingSecret` | No | none | Name of an existing secret holding the ePO password (key `epo-passwd`), used in place of `epo.password` |
| `image.repository` | Yes | `--image-repo` | Full image registry path |
| `image.digest` | Yes | `--image-digest` | SHA256 digest of the image |
| `imagePullSecret` | No | `--image-pull-secret` | Name of the pull secret. Required in registry mode. Leave empty (`""`) in airgap mode: the DaemonSet then omits `imagePullSecrets`, and OpenShift trusts images in its own internal registry. |
| `deploymentDetails.clusterName` | Yes | `--cluster-name` | Name identifying this cluster in ePO and telemetry |
| `deploymentDetails.deploymentMode` | Yes | none (defaults to `Trellix Managed`) | How the deployment is managed |

Required fields use Helm's `required` function, so an install or upgrade fails immediately if
any of them is empty.

---

## Appendix B — Installing with Helm directly (without `deploy.sh`)

Use this only if you can't run `deploy.sh` (for example, no bash available) and you're in
registry mode. Run it from the bundle folder after Parts A–F, as a single line:

```
helm upgrade --install edrf-xclient ./edrf-xclient -n edrf-tests --create-namespace --set epo.ip="<epo_ip>" --set epo.user="<epo_user>" --set epo.password="<epo_password>" --set deploymentDetails.clusterName="<cluster_name>" --set image.repository="<image_repo>" --set image.digest="sha256:<digest>" --set imagePullSecret="artifactory-pull-secret"
```

Passing the ePO password with `--set` at install time, instead of saving it in
`values.yaml`, keeps it out of files that might end up in git. Verify as in Part H.

---

## Appendix C — Container image and runtime internals

Background for troubleshooting with the dev team. You don't need this to deploy.

**Image:**
- **Base image:** RapidFort curated, FIPS-enabled UBI 10 minimal
  (`artifactory.trellix.com/rapidfort/rfcurated/ubi:ubi10-minimal-fips-rfcurated`), a hardened,
  low-CVE base.
- **Contents:** `tini`, runtime dependencies, the McAfee Agent (MA) and XClient RPMs, DXL (Data
  Exchange Layer, installed from the TA package bundle), and `entrypoint.sh`, the in-container
  supervisor.
- **Built by:** the Orbit pipeline `orbit/ContainerSecurity_ImageBuild.groovy` (build context
  `rhel10/`). The customer bundle is produced by a separate, manually triggered pipeline,
  `orbit/ContainerSecurity_BundlePackage.groovy`.

**Run model.** `tini` is the container's init process and supervises `entrypoint.sh`, which:
1. provisions the McAfee Agent (MA/TA) against ePO,
2. starts MA and XClient and waits for their daemons to come up,
3. health-checks both in a loop, and exits (causing a pod restart) if either one dies, and
4. on `SIGTERM`, stops XClient and then MA gracefully before the container exits.

Step 3 is why a crashing agent shows up as a climbing `RESTARTS` count rather than a pod that
stays down (Part H5).

**DaemonSet design:**
- **Linux nodes only:** `nodeSelector: kubernetes.io/os: linux`. The workload needs eBPF, the
  CRI-O socket, and host `/sys` and `/lib/modules`, so it can't run on Windows nodes.
- **Security context:** runs as `runAsUser: 0` but **not privileged** (`privileged: false`).
  All Linux capabilities are dropped, then only the ones the eBPF sensor needs are added back
  (`SYS_ADMIN`, `BPF`, `PERFMON`, `SYS_RESOURCE`, `NET_ADMIN`, and others listed in
  `values.yaml`). The same list feeds the chart's custom SCC, so the two always match.
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
applies the required Pod Security Admission (PSA) labels in place of an SCC.

**Local testing on minikube:** use `edrf-xclient-minikube.yml`, a single flattened manifest
(namespace, ServiceAccount, ConfigMap, Secret, DaemonSet) from the product source:
```bash
minikube start --container-runtime=cri-o   # the DaemonSet mounts the CRI-O socket
minikube image load <image_name:tag>       # no pull secret is used
# edit the <epo_ip>/<epo_username>/<epo_password> and <image_name:tag> placeholders first
# CLUSTER_NAME and DEPLOYMENT_MODE are hardcoded in the manifest; edit them if needed
kubectl apply -f edrf-xclient-minikube.yml
```

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
