# eDRF XClient — OpenShift Deployment Guide

**Audience:** engineers deploying this chart for the first time. No prior assumption that
`oc`, `helm`, or Artifactory access is already set up. Follow the parts in order — later
steps depend on resources earlier steps create.

This guide covers three shells: **PowerShell**, **Command Prompt (`cmd.exe`)**, and
**Linux/Mac (bash/zsh)**.

- Every `oc` and `helm` command is plain text that runs identically in all three shells —
  shown once, with no shell label.
- A small number of steps genuinely differ by shell — setting an environment variable,
  extracting an archive, editing PATH. These appear as three labeled blocks: **PowerShell**,
  **cmd.exe**, **Linux/Mac**. Use the one matching your terminal.
- Chart paths are written as `./edrf-xclient` (forward slash) throughout — this resolves
  correctly in all three shells.
- One shell-specific quirk: PowerShell aliases `curl` to `Invoke-WebRequest`. Use `curl.exe`
  explicitly in PowerShell wherever `curl` appears below (called out again in Part C).

**Read Part 0 first** — it determines which machine you run the rest of this guide from.

> **Namespace note:** examples below use `edrf-tests`, a placeholder. This name is expected
> to change in a future release — use whatever namespace your team has standardized on.

---

## Quick glossary

| Term | What it means here |
|---|---|
| **OpenShift / `oc`** | Red Hat's Kubernetes distribution; `oc` is its CLI (a superset of `kubectl`). |
| **Project / Namespace** | An isolated area in the cluster. "Project" is OpenShift's name for a Kubernetes Namespace — same thing. |
| **Helm / chart / release** | Helm is a package manager for Kubernetes. A **chart** is the package (`edrf-xclient.zip`). A **release** is one installed instance of that chart in a namespace (named `edrf-xclient` below). |
| **DaemonSet** | A workload that runs exactly one pod on every node — how the chart deploys the XClient agent. |
| **Secret / ConfigMap** | Kubernetes objects for sensitive values (Secret) and non-sensitive config (ConfigMap). |
| **ServiceAccount (SA)** | The identity a pod runs as; here it's granted the privileged permissions the agent needs. |
| **SCC** | Security Context Constraint — OpenShift's permission gate for privileged containers. This chart requires the `privileged` SCC. |
| **Artifactory** | Trellix's private container registry (`artifactory.trellix.com`), where the agent image is hosted. |
| **ePO** | Trellix ePolicy Orchestrator — the management server the XClient agent reports to. |

---

## Part 0 — Determine which machine does what

This deployment touches two networks, and many corporate environments don't let one machine
reach both:

1. The **OpenShift cluster API** — needed for every `oc` and `helm` command (Parts A, B, D–M).
2. **Artifactory**, on the general corporate network — needed only for Part C (image digest
   lookup).

A locked-down **jumpbox** dedicated to cluster administration often has no route to the
general internet. An everyday workstation typically has internet/Artifactory access but no
path to the OpenShift API. Confirm which situation applies before proceeding.

**Run this once, from your jumpbox:**
```
curl -v --max-time 10 https://artifactory.trellix.com
```
(PowerShell: `curl.exe` instead of `curl`.)

- **Fast response** (even an error page, as long as it's not a hang) → the jumpbox has
  internet access. Run every part of this guide, including Part C, from the jumpbox.
- **Hangs until timeout** → split the work:
  - **Part C only** (resolving `image.repository` / `image.digest`) — run from your everyday
    workstation. Only `curl` and your Artifactory credentials (Part B3) are needed there.
  - **Everything else (Parts A, B, D–M)** — stays on the jumpbox, the only machine that can
    reach the OpenShift API.
  - The only values that need to move between machines are the two short strings Part C
    produces (`image.repository`, `image.digest`) — paste them into the `helm install`
    command in Part G.

Every part below is tagged **(jumpbox)**, **(any machine)**, or
**(Artifactory-reachable machine)**.

---

## Part A (jumpbox) — Install the tools

You need the `oc` CLI, the `helm` CLI, and `curl` (built into Windows 10+, Linux, and Mac —
nothing to install).

### A1. Install the `oc` CLI

**Option 1 — from the cluster's web console** (matches your cluster version exactly):
1. Log in to the OpenShift web console.
2. Click the **`?` (help)** icon (top-right) → **Command Line Tools**.
3. Download the **oc** CLI for your OS.

**Option 2 — from the official mirror:**
https://mirror.openshift.com/pub/openshift-v4/clients/oc/latest/ — separate subfolders for
Windows (`.zip`), Linux (`.tar.gz`), and Mac (`.tar.gz`).

**Extract the archive:**

**PowerShell:**
```powershell
Expand-Archive -Path "$env:USERPROFILE\Downloads\oc.zip" -DestinationPath "$env:USERPROFILE\Downloads\oc-extracted"
```
**cmd.exe** (Windows' built-in `tar.exe` extracts `.zip` files too — create the destination
first, since `tar -C` requires it to already exist):
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

**Confirm the binary runs before touching PATH:**

**Windows (PowerShell or cmd.exe):**
```
C:\oc\oc.exe version --client
```
**Linux/Mac:**
```bash
~/bin/oc version --client
```

**Expected:** `Client Version: 4.x.x`.

**Add it to PATH:**

**PowerShell** (permanent; this also applies to future `cmd.exe` sessions, since it's a real
Windows user environment variable):
```powershell
[Environment]::SetEnvironmentVariable("Path", "$env:Path;C:\oc", "User")
```
**cmd.exe** (`setx` can truncate a very long existing PATH — prefer the PowerShell command
above or the GUI method if your PATH is already large):
```cmd
setx PATH "%PATH%;C:\oc"
```
**Windows GUI** (works regardless of your default shell):
1. **Win + R** → `sysdm.cpl` → Enter.
2. **Advanced** tab → **Environment Variables**.
3. Under **User variables**, select `Path` → **Edit** → **New**.
4. Add `C:\oc` → OK on all dialogs.

**Linux/Mac:**
```bash
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc    # or ~/.zshrc
source ~/.bashrc
```

Open a new terminal, then verify:

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

**Expected:** the path command prints the binary's location, and `oc version --client` runs
without a full path.

*(Alternative on Windows with Chocolatey: `choco install openshift-cli`.)*

**If `oc` is still not recognized** after opening a new terminal: PATH wasn't updated, or
you're in a terminal session opened before the change. `where.exe oc` / `which oc` returning
nothing confirms a PATH problem — redo the PATH step and open a fresh terminal.

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

**Verify:**
```
helm version
```

**Expected:** `version.BuildInfo{Version:"v3...`. This chart requires Helm 3.

### A3. Nothing else to install

`curl` is sufficient for the image digest lookup in Part C — Docker and `skopeo` are not
required.

---

## Part B — Collect access and credentials

B1, B2, and B5 run on the jumpbox; B3 and B4 only require a browser and can run from any
machine.

### B1 (jumpbox). OpenShift cluster login

Confirm with whoever provisioned your access which method applies.

**Option 1 — token login:**
1. Log in to the web console.
2. Click your username (top-right) → **Copy login command**.
3. In the new tab, click **Display Token**.
4. Copy and run the resulting command:
   ```
   oc login --token=sha256~xxxxxxxxxxxx --server=https://api.<your-cluster>:6443
   ```
   Identical across all three shells.

Tokens from this flow are short-lived — if `oc whoami` later fails with an auth error, repeat
these steps for a fresh token.

**Option 2 — kubeconfig file:**

Some teams issue a `kubeconfig` file directly (often for a shared service account) instead of
individual logins.

1. Save the file somewhere stable, e.g. `C:\Users\<you>\Documents\kubeconfig`.
2. Point `oc` at it via the `KUBECONFIG` environment variable. Setting it in one shell does
   not make it visible in another (each is a separate session-level setting unless made
   permanent).

   **PowerShell** (current session):
   ```powershell
   $env:KUBECONFIG = "C:\Users\<you>\Documents\kubeconfig"
   ```
   Verify:
   ```powershell
   echo $env:KUBECONFIG
   ```
   Permanent (also applies to future `cmd.exe` sessions):
   ```powershell
   [Environment]::SetEnvironmentVariable("KUBECONFIG", "C:\Users\<you>\Documents\kubeconfig", "User")
   ```

   **cmd.exe** (current session):
   ```cmd
   set KUBECONFIG=C:\Users\<you>\Documents\kubeconfig
   ```
   Verify:
   ```cmd
   echo %KUBECONFIG%
   ```
   `set` only affects the current window — make it permanent with the PowerShell command
   above (run once, applies to future `cmd.exe` windows too) or the GUI method from A1, using
   variable name `KUBECONFIG`.

   **Linux/Mac** (current session):
   ```bash
   export KUBECONFIG=/home/<you>/kubeconfig
   ```
   Verify:
   ```bash
   echo $KUBECONFIG
   ```
   Permanent:
   ```bash
   echo 'export KUBECONFIG=/home/<you>/kubeconfig' >> ~/.bashrc    # or ~/.zshrc
   ```

**Verify login (either option, identical in every shell):**
```
oc whoami
```

**Expected:** your identity for token login, or the account baked into the kubeconfig (e.g.
`system:admin`).

Confirm the cluster itself is reachable:
```
oc cluster-info
oc get nodes
```

**Expected:** all nodes show `STATUS = Ready`. A hang or error here is a network/VPN/DNS issue
reaching the API server, not a login problem.

**If `oc whoami` fails:**
- *Token login:* the token likely expired — repeat Option 1.
- *Kubeconfig login:* confirm the variable is set (`echo $env:KUBECONFIG` / `echo %KUBECONFIG%`
  / `echo $KUBECONFIG`) and the file exists at that path
  (`Test-Path $env:KUBECONFIG` / `dir "%KUBECONFIG%"` / `ls -la "$KUBECONFIG"`). Check which
  identity/cluster it's configured for with `oc config get-contexts` and
  `oc config current-context`.

**Reference — read-only commands for sanity-checking the cluster at any point:**
```
oc version              # client + server version
oc whoami                # current identity
oc project               # current project/namespace
oc get projects          # every project you can see
oc get nodes             # cluster nodes
oc get pods -A           # pods across all namespaces
oc get namespaces        # list namespaces
oc get clusteroperators  # cluster operator health
```

### B2 (jumpbox). Confirm cluster permissions

This chart creates a namespace, ServiceAccount, Secrets, a ConfigMap, a DaemonSet, and a
**RoleBinding to `system:openshift:scc:privileged`** — usually restricted to admins.

Check early for a signal:
```
oc auth can-i create rolebindings -n edrf-tests
```
If this returns `no`, ask your OpenShift platform/cluster-admin team to grant the permission,
or to run the `helm install` command in Part G on your behalf.

### B3 (any machine with a browser). Artifactory credentials

You need an **API key or identity token**, not your regular login password — Artifactory does
not accept SSO passwords for CLI/API pulls.

1. Log in at https://artifactory.trellix.com.
2. Click your username/avatar (top-right) → **Edit Profile**.
3. Find **Generate an Identity Token** (newer JFrog UI) or **Generate API Key** (older UI).
4. Generate one and save it to a password manager immediately — most Artifactory versions
   only display it once.

If your UI differs, check your Artifactory/DevOps support channel — the menu wording varies
by version, but "Identity Token" / "API Key" under your profile is the standard JFrog
location.

**The identity token screen shows several fields — only one is the credential you use:**

| Field shown on screen | Example shape | Use it? |
|---|---|---|
| **Reference token** | long string, e.g. `cmVmdGtuOjAxOjE4MjQ4...` | Yes — this is the credential. Use it as `<artifactory_api_key>` throughout this guide (`--docker-password` in Part E, `curl -u` in Part C). |
| Username | your email | This is `<artifactory_user>` |
| Scope | e.g. `applied-permissions/user` | Informational only |
| Audience | e.g. `*@*` | Informational only |
| Expires In | e.g. `11 month(s) 30 day(s)` | Informational — tells you when to regenerate |
| **Token ID** | short UUID, e.g. `6a66d39a-7450-489b-...` | No — not a credential. An internal label for locating/revoking the token. Using it as a password fails with `401 Unauthorized`. |

The Reference Token and Token ID look similarly random at a glance — if a pull or lookup
fails with `401`, confirm you used the Reference Token, not the Token ID.

### B4 (any machine — information only). ePO server details

The agent needs:
- ePO IP/hostname
- ePO username
- ePO password

Request these from the team managing ePO for your environment. A dedicated service account
is preferred over a personal login.

### B5 (jumpbox). Extract the Helm chart

Extract the chart on the jumpbox — Parts D onward run `helm` commands from this folder. If
the archive is only on your workstation, transfer it to the jumpbox first.

This guide assumes `C:\edrf-deploy` (Windows) or `~/edrf-deploy` (Linux/Mac); adjust the path
if you use a different location.

**PowerShell:**
```powershell
mkdir C:\edrf-deploy
Expand-Archive -Path C:\path\to\edrf-xclient.zip -DestinationPath C:\edrf-deploy
```
**cmd.exe:**
```cmd
mkdir C:\edrf-deploy
tar -xf C:\path\to\edrf-xclient.zip -C C:\edrf-deploy
```
**Linux/Mac:**
```bash
mkdir -p ~/edrf-deploy
unzip ~/Downloads/edrf-xclient.zip -d ~/edrf-deploy
```
(If `unzip` is missing: `sudo apt install unzip` or `sudo yum install unzip`. Mac includes
`unzip` by default.)

Resulting layout:
```text
edrf-xclient/
├── Chart.yaml                   # chart identity: name, version, description
├── values.yaml                  # every configurable value (ePO, image, resources, deployment identity)
└── templates/
    ├── 01-serviceaccount.yaml   # ServiceAccount + RoleBinding for the privileged SCC
    ├── 02-epo-details.yaml      # ConfigMap + Secret holding ePO credentials
    └── 03-deploy-edrf.yaml      # the DaemonSet that runs XClient on every node
```
Values are supplied via `--set` flags in Part G, not by editing `values.yaml` or the
templates directly.

**Background:** the container image bundles DXL (Data Exchange Layer). The DaemonSet injects
identity environment variables for Kubernetes-side telemetry — `CLUSTER_NAME` /
`DEPLOYMENT_MODE` (from `deploymentDetails`, set in Part G), `POD_NAME`, and
`TRELLIX_KUBERNETES_ENVIRONMENT=1` — and mounts the host's `/etc/os-release` so the agent
reports the host OS rather than the container image's OS.

Run all subsequent commands from this folder:
```
cd C:\edrf-deploy        # Windows
cd ~/edrf-deploy          # Linux/Mac
```

**Sanity-check the chart**, from the parent of `edrf-xclient` (identical across shells):
```
helm lint ./edrf-xclient
```

**Expected:** `level=WARN msg="missing required values"` lines are normal at this stage (values
haven't been supplied yet). The line that matters is:
```
1 chart(s) linted, 0 chart(s) failed
```
`0 chart(s) failed` confirms the chart is structurally valid.

> **If you instead see:**
> ```
> Error unable to check Chart.yaml file in chart: GetFileAttributesEx edrf-xclient\Chart.yaml: The system cannot find the path specified.
> ```
> (Windows) or
> ```
> Error unable to check Chart.yaml file in chart: stat edrf-xclient/Chart.yaml: no such file or directory
> ```
> (Linux/Mac) — the command is being run from the wrong directory, commonly because you're
> already inside `edrf-xclient` itself. Check your location:
> ```powershell
> Get-Location                              # PowerShell
> ```
> ```cmd
> cd                                        :: cmd.exe, no arguments
> ```
> ```bash
> pwd                                       # Linux/Mac
> ```
> then move up one level and retry:
> ```
> cd C:\edrf-deploy     (Windows)     /     cd ~/edrf-deploy     (Linux/Mac)
> helm lint ./edrf-xclient
> ```
> Every `helm` command in this guide assumes you're one level above the `edrf-xclient`
> folder. A path error is almost always a wrong-directory problem — check your location
> first.

---

## Part C (Artifactory-reachable machine — see Part 0) — Resolve the image repository and digest

If Part 0 showed the jumpbox has no internet access, run this part on your workstation
instead. Only `curl` and your Artifactory credentials are needed — not `oc`, `helm`, or the
chart. Carry the resulting `image.repository` and `image.digest` values back to the jumpbox
for Part G.

A typical Artifactory container path —
`https://artifactory.trellix.com/artifactory/edrf-xclient-container-security/trellix-edrf/114/`
— is the web UI browse page for a given build (the trailing `/` makes it a folder listing,
not a file). `values.yaml` requires two separate fields:

```yaml
image:
  repository: ""   # e.g. artifactory.trellix.com/edrf-xclient-container-security/trellix-edrf
  digest: ""        # e.g. sha256:3f9a1c...
```

**PowerShell note, applies to every command below:** PowerShell aliases `curl` to
`Invoke-WebRequest`. Use `curl.exe` explicitly. `cmd.exe` and Linux/Mac use plain `curl` as
written.

### Method 1 (preferred) — Artifactory Storage API

A Docker image digest is the SHA256 checksum of its manifest file. Artifactory's Storage API
returns that checksum directly, using the same path from the browse UI (`api/storage/`
inserted after `/artifactory/`, with the manifest filename appended):

```
curl -s -u "<artifactory_user>:<artifactory_api_key>" "https://artifactory.trellix.com/artifactory/api/storage/edrf-xclient-container-security/trellix-edrf/114/manifest.json"
```
(PowerShell: `curl.exe`.)

A working response:
```json
{
  "repo": "edrf-xclient-container-security",
  "path": "/trellix-edrf/114/manifest.json",
  "checksums": {
    "sha1": "...",
    "md5": "...",
    "sha256": "3f9a1c0b7e..."
  }
}
```
Prepend `sha256:` to the `sha256` value for `image.digest`. `image.repository` is:
```
artifactory.trellix.com/edrf-xclient-container-security/trellix-edrf
```

**If the response isn't clean JSON**, re-run with `-v` to see the full response:
```
curl -v -u "<artifactory_user>:<artifactory_api_key>" "https://artifactory.trellix.com/artifactory/api/storage/edrf-xclient-container-security/trellix-edrf/114/manifest.json"
```
(PowerShell: `curl.exe`.)

Check the HTTP status:
- **`200 OK` with `"children": [...]` instead of `"checksums"`** → `114` is a folder and
  `manifest.json` isn't the actual filename. List the folder first:
  ```
  curl -s -u "<artifactory_user>:<artifactory_api_key>" "https://artifactory.trellix.com/artifactory/api/storage/edrf-xclient-container-security/trellix-edrf/114/"
  ```
  then re-run with the real filename.
- **`401 Unauthorized` / `403 Forbidden`** → the username or identity token is wrong, expired,
  or lacks read access — see Part B3, confirm you're using the Reference Token.
- **`404 Not Found`** → `edrf-xclient-container-security` may be a display name rather than
  the actual repo key, or build 114 doesn't exist at this path. Confirm the repo key and path
  with whoever provided the Artifactory link or your CI/build team.
- **`302` redirect or an HTML login page instead of JSON** → the instance is rejecting this
  auth method for API calls (common with SSO-integrated setups). Check with your Artifactory
  admins whether a different token type or header is required.

### Method 2 (fallback) — Docker Registry API

If the actual Docker repository name differs from the browse-UI name (some orgs use a
`-docker` suffix), query the registry directly:

**PowerShell:**
```powershell
curl.exe -sI -u "<artifactory_user>:<artifactory_api_key>" -H "Accept: application/vnd.docker.distribution.manifest.v2+json" "https://artifactory.trellix.com/artifactory/api/docker/<docker-repo-name>/v2/trellix-edrf/manifests/114" | Select-String -Pattern "docker-content-digest"
```
**cmd.exe:**
```cmd
curl -sI -u "<artifactory_user>:<artifactory_api_key>" -H "Accept: application/vnd.docker.distribution.manifest.v2+json" "https://artifactory.trellix.com/artifactory/api/docker/<docker-repo-name>/v2/trellix-edrf/manifests/114" | findstr /i "docker-content-digest"
```
**Linux/Mac:**
```bash
curl -sI -u "<artifactory_user>:<artifactory_api_key>" -H "Accept: application/vnd.docker.distribution.manifest.v2+json" "https://artifactory.trellix.com/artifactory/api/docker/<docker-repo-name>/v2/trellix-edrf/manifests/114" | grep -i docker-content-digest
```
Replace `<docker-repo-name>` with the actual repository name. A working response prints:
```
Docker-Content-Digest: sha256:3f9a1c0b7e...
```
`image.repository` is `artifactory.trellix.com/<docker-repo-name>/trellix-edrf`;
`image.digest` is the printed value.

---

Keep the resolved repository + digest for Part G. Do not write the digest into
`values.yaml` directly — it changes with every build.

---

## Part D (jumpbox) — Create the namespace

```
oc new-project edrf-tests
```
If it already exists:
```
oc project edrf-tests
```
**Verify:**
```
oc project
```

**Expected:** `Using project "edrf-tests" on server ...`.

---

## Part E (jumpbox) — Create the Artifactory image pull secret

OpenShift needs its own copy of your Artifactory credentials to pull the private image. This
command builds the secret correctly, identical across all three shells:

```
oc create secret docker-registry artifactory-pull-secret --docker-server=artifactory.trellix.com --docker-username="<artifactory_user>" --docker-password="<artifactory_api_key>" -n edrf-tests
```

**If the token contains special characters** like `&`, `|`, `^`, or `!`, `cmd.exe` can
misinterpret them even inside quotes — run this command from PowerShell instead, or
regenerate a token without those characters.

**Verify:**
```
oc get secret artifactory-pull-secret -n edrf-tests
```

**Expected:** one row, `TYPE kubernetes.io/dockerconfigjson`.

Note the secret name — `artifactory-pull-secret` — it's used as `imagePullSecret` in Part G.

**What this creates:** a `kubernetes.io/dockerconfigjson` secret, the same `config.json`
format Docker uses for registry logins:
```json
{
  "auths": {
    "artifactory.trellix.com": {
      "username": "<artifactory_user>",
      "password": "<artifactory_api_key>",
      "auth": "<base64 of artifactory_user:artifactory_api_key>",
      "email": "<artifactory_user>"
    }
  }
}
```
The `auth` field is `username:password`, base64-encoded. Do not construct this file by hand —
the command above handles the encoding correctly. To inspect the decoded contents:
```
oc get secret artifactory-pull-secret -n edrf-tests -o jsonpath='{.data.\.dockerconfigjson}'
```
Pipe the (base64) output through `| base64 -d` on Linux/Mac, or
`| % {[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($_))}` in PowerShell, to
read it in plain text. Treat that output as a live credential.

---

## Part F (jumpbox) — Pre-create the ePO password secret (recommended)

Using an existing secret keeps the ePO password out of shell history and `helm get values`
output.

```
oc create secret generic epo-existing-secret --from-literal=epo-passwd="<epo_password>" -n edrf-tests
```
(Same special-character caveat as Part E — switch to PowerShell or bash if `cmd.exe`
misinterprets the password.)

**Verify:**
```
oc get secret epo-existing-secret -n edrf-tests
```

Reference it by name (`epo.existingSecret=epo-existing-secret`) in Part G. Passing the
password inline instead is also supported — see the note there.

---

## Part G (jumpbox) — Dry-run, then install

Confirm you're in `C:\edrf-deploy` (or `~/edrf-deploy`), with `edrf-xclient` present.

**Step 1 — dry run.** Renders and validates against the cluster without creating anything:

```
helm upgrade --install edrf-xclient ./edrf-xclient -n edrf-tests --set image.repository="artifactory.trellix.com/<docker-repo-name>/trellix-edrf" --set image.digest="sha256:<digest-from-part-c>" --set imagePullSecret="artifactory-pull-secret" --set epo.ip="<epo_ip>" --set epo.user="<epo_user>" --set epo.existingSecret="epo-existing-secret" --set deploymentDetails.clusterName="<your-cluster-name>" --dry-run=client
```

Replace every `<...>` placeholder with values collected in Parts B and C.

- If you skipped Part F, remove `--set epo.existingSecret="epo-existing-secret"` and add
  `--set epo.password="<epo_password>"` instead.
- `<your-cluster-name>` is a label identifying this cluster in reporting. OpenShift's
  installer names nodes `<cluster-name>-<random-infra-id>-<role>-<index>` — the part before
  the random suffix is usually the right value. Confirm with:
  ```
  oc get infrastructure cluster -o jsonpath='{.status.infrastructureName}'
  ```
  If your ePO console already has agents reporting from other clusters, match their naming
  convention — this is ultimately a reporting-identity decision for whoever manages ePO, not
  a technical constraint of the chart.

An `execution error ... is required` means a `--set` value is missing or misspelled — fix and
re-run before continuing.

**Optional — render without touching the cluster at all.** `helm template` renders entirely
locally (no `oc login` required), useful for reviewing every manifest line before applying
anything:
```
helm template edrf-xclient ./edrf-xclient --set image.repository="artifactory.trellix.com/<docker-repo-name>/trellix-edrf" --set image.digest="sha256:<digest-from-part-c>" --set imagePullSecret="artifactory-pull-secret" --set epo.ip="<epo_ip>" --set epo.user="<epo_user>" --set epo.existingSecret="epo-existing-secret" --set deploymentDetails.clusterName="<your-cluster-name>" > rendered.yaml
```

**Step 2 — real install.** Same command with `--dry-run=client` removed:

```
helm upgrade --install edrf-xclient ./edrf-xclient -n edrf-tests --set image.repository="artifactory.trellix.com/<docker-repo-name>/trellix-edrf" --set image.digest="sha256:<digest-from-part-c>" --set imagePullSecret="artifactory-pull-secret" --set epo.ip="<epo_ip>" --set epo.user="<epo_user>" --set epo.existingSecret="epo-existing-secret" --set deploymentDetails.clusterName="<your-cluster-name>"
```

**Expected:** ends with `STATUS: deployed`.

This creates, in order: an `edrf-admin` ServiceAccount with the privileged-SCC RoleBinding,
an `epo-details` ConfigMap (and Secret, if not using an existing one), and a DaemonSet
(`edrf-xclient-container-security`) running one pod per Linux node. The pod runs with host
networking, host PID namespace, and privileged mode — required for the agent to observe
kernel/container activity on the node.

A `Forbidden` error mentioning `rolebindings` or `clusterroles` means insufficient
permissions — return to Part B2.

---

## Part H (jumpbox) — Link the pull secret to the service account

`edrf-admin` is created during the Part G install, so this step must run after, not before:

```
oc secrets link edrf-admin artifactory-pull-secret --for=pull -n edrf-tests
```

The DaemonSet also references the pull secret directly in its pod spec, so pulls should
already succeed without this — this step covers anything else running as this
ServiceAccount later.

---

## Part I (jumpbox) — Verify the deployment

```
# One pod per Linux node — all should be Running, not Pending/ImagePullBackOff/CrashLoopBackOff
oc get pods -n edrf-tests -o wide

# Overall Helm release status
helm status edrf-xclient -n edrf-tests

# Confirm applied values (--all includes values.yaml defaults, not just --set overrides)
helm get values edrf-xclient -n edrf-tests --all

# Tail logs to confirm the agent connected to ePO
oc logs -n edrf-tests -l name=edrf-xclient-container-security --tail=100

# Every Helm release in this namespace
helm list -n edrf-tests

# Every release across all namespaces you can see
helm list --all-namespaces
```

**Expected:** one `Running` pod per Linux worker node, `helm status` showing
`STATUS: deployed`, and pod logs showing a successful ePO connection — a `Running` status
alone only confirms the container started, not that the agent is reporting.

---

## Part J (jumpbox) — Troubleshooting

| Symptom | Likely cause | Action |
|---|---|---|
| `execution error ... required` during `helm upgrade --install` | A required `--set` value is missing or misspelled | Re-check every value against Parts B/C/D/E/F |
| `Forbidden` / RBAC error during install | Insufficient permission to create the RoleBinding | Part B2 — request access from your platform/cluster-admin team |
| `namespace "<ns>" not found` when creating a secret | The namespace doesn't exist yet, or was created after an earlier failed attempt | Run Part D, then re-run the secret command and confirm it succeeded with `oc get secret -n <ns>` — don't assume a prior command worked |
| `secret "<name>" not found` when running `oc secrets link` | The secret was never actually created (a common follow-on from the row above) | Verify with `oc get secret -n <ns>`, then re-run Part E |
| Pod shows `ImagePullBackOff`, event `Authentication is required` / `Unable to retrieve some image pull secrets` | The pull secret didn't exist (or wasn't in the right namespace) when the pod was scheduled | Fix the secret (Part E), then force an immediate retry — don't wait for kubelet's backoff timer: `oc delete pod -n edrf-tests -l name=edrf-xclient-container-security` |
| Pod stuck in `Pending` | The ServiceAccount didn't get the privileged SCC binding | Confirm with `oc get rolebinding -n edrf-tests`; if missing, re-run Part G with sufficient permissions |
| Pod `Running`, but not visible in the ePO console | Wrong `epo.user`/`epo.password` (including leftover placeholder values), `epo.ip` unreachable from the worker node network, or the agent is pending manual approval in ePO | Check `oc logs <pod-name> -n edrf-tests` for connection/auth errors; confirm `epo.ip` is reachable from the worker node itself (pods run `hostNetwork: true`, so test from the node, not the pod network, via `oc debug node/<node-name>`); ask your ePO admin to check the pending/unmanaged systems list |
| `CrashLoopBackOff`, logs mention ePO connection failures | Wrong ePO IP/user/password, or the cluster can't reach the ePO server | Re-check Part B4 values; confirm network reachability with your network team |
| Agent reports the container's OS instead of the host's | Should not occur — the chart mounts `/etc/os-release` automatically | `oc describe pod <pod-name> -n edrf-tests` and confirm the `host-os-release` volume is mounted; if missing, verify the chart version |

For anything not covered above, `oc describe pod <pod-name> -n edrf-tests` and
`oc logs <pod-name> -n edrf-tests` resolve most issues directly.

---

## Part K (jumpbox, plus Part C's machine) — Upgrading to a new build

When a new build ships with a new digest, repeat only Part C for the new digest, then:

```
helm upgrade edrf-xclient ./edrf-xclient -n edrf-tests --reuse-values --set image.digest="sha256:<new-digest>"
```

`--reuse-values` preserves everything else (ePO settings, pull secret, cluster name).

---

## Part L (jumpbox) — Rolling back

```
helm history edrf-xclient -n edrf-tests
```
Identify the last good revision, then:
```
helm rollback edrf-xclient <revision-number> -n edrf-tests
```

---

## Part M (jumpbox) — Uninstalling

```
helm uninstall edrf-xclient -n edrf-tests
```

This removes the DaemonSet, ServiceAccount, RoleBinding, ConfigMap, and Secret that Helm
created. It does **not** remove resources created manually outside Helm:

```
# The namespace itself
oc delete namespace edrf-tests

# The Artifactory pull secret from Part E
oc delete secret artifactory-pull-secret -n edrf-tests

# The ePO secret from Part F, if created
oc delete secret epo-existing-secret -n edrf-tests
```
