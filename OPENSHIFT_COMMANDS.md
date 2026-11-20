# eDRF XClient on OpenShift: Useful Commands for Testing and Troubleshooting

These are commands for working with a deployed `edrf-xclient` agent on OpenShift. They cover
nodes and agent pods, the agent CLI inside the pod, node shells, test pods, network checks,
adding worker nodes, exclusion tests, and evidence collection. For installing, upgrading, or
uninstalling the agent itself, see `DEPLOYMENT_GUIDE.md`.

All commands were used on build 50.3.6.x from a Windows jumpbox (`cmd.exe`) with `oc` logged in
as cluster-admin. They work the same in a Linux shell, except where a section says otherwise.

**Replace every `<...>` placeholder before you run a command:**

| Placeholder | Meaning | How to find it |
|---|---|---|
| `<namespace>` | Namespace the agent DaemonSet runs in | `helm list -A \| findstr edrf-xclient` |
| `<node-name>` | Worker node name | `oc get nodes` |
| `<agent-pod>` | Agent pod on that node | Section 1 |
| `<test-namespace>` | Namespace for your own test pods, for example `edrf-qa-custom` | You choose it |
| `<machineset>` | Worker MachineSet name | Section 6 |

**Quoting rules in `cmd.exe`:**

- Put the whole `sh -c` / `bash -c` script in **double quotes**.
- Use **single quotes** inside it.
- `$VAR`, `$(...)` and `|` inside the double quotes are passed to the Linux shell unchanged.

---

## 1. Nodes and agent pods

```
oc whoami
oc whoami --show-server
oc get nodes -o wide
oc get pods -n <namespace> -o wide
```

**Find the agent pod on a specific node:**
```
oc get pods -n <namespace> -o wide --field-selector spec.nodeName=<node-name>
```

**Node details** (OS image, kernel, Ready state):
```
oc get node <node-name> -o jsonpath="{.status.nodeInfo.osImage} {.status.nodeInfo.kernelVersion}{'\n'}"
oc get node <node-name> -o jsonpath="{.status.conditions[?(@.type=='Ready')].status}{'\n'}"
```

**Pod health:**
```
oc describe pod -n <namespace> <agent-pod>
oc logs -n <namespace> <agent-pod> --previous
oc get events -n <namespace> --sort-by=.lastTimestamp
```

**Restart one agent pod.** The DaemonSet recreates it on the same node.
```
oc delete pod -n <namespace> <agent-pod>
```

---

## 2. Agent CLI inside the pod

The Trellix Agent (MA) is at `/opt/McAfee/agent/bin/cmdagent`, and XClient is at
`/opt/Trellix/XClient/bin/xclient`.

**Agent info** (agent GUID, ePO server, TA version, XClient version):
```
oc exec -n <namespace> <agent-pod> -- /opt/McAfee/agent/bin/cmdagent -i
```

**Pull and enforce policy now,** instead of waiting for the next ASCI. Run both after every
policy change in ePO:
```
oc exec -n <namespace> <agent-pod> -- /opt/McAfee/agent/bin/cmdagent -c
oc exec -n <namespace> <agent-pod> -- /opt/McAfee/agent/bin/cmdagent -e
```

**Dump the agent's effective configuration** (every policy key the agent received), then save
it on the jumpbox:
```
oc exec -n <namespace> <agent-pod> -- sh -c "/opt/Trellix/XClient/bin/xclient -x /tmp/cfg.txt >/dev/null 2>&1; cat /tmp/cfg.txt; rm -f /tmp/cfg.txt" > agent_config.txt
```

**Check specific keys.** Change the `grep` pattern to the keys you need:
```
oc exec -n <namespace> <agent-pod> -- sh -c "/opt/Trellix/XClient/bin/xclient -x /tmp/cfg.txt >/dev/null 2>&1; grep -iE 'NetworkFlow/enabled|UnifiedEdr/enabled|HostRemediation/enabled|unquarantine_password_enabled|Allowed_IP' /tmp/cfg.txt; rm -f /tmp/cfg.txt"
```

Keys used during testing:

| Key | What it controls |
|---|---|
| `UEDR/NetworkFlow/enabled` | Network flow feature (containment depends on it) |
| `mxa/container/UnifiedEdr/enabled` | Unified EDR in the container agent |
| `mxa/container/HostRemediation/enabled` | Host remediation (containment) |
| `mxa/container/Eventor/enabled` | Eventor (event collection) |
| `UEDR/NetworkFlow/unquarantine_password_enabled` | Unlock code for `uncontain` |
| `mxa/eventor/excludedProcessNames` | Eventor process exclusions |
| `mxa/eventor/excludedPaths` | Eventor file and path exclusions |
| `UEDR/FileHashing/ignored_paths_linux` | Paths never hashed (default includes `/tmp`, `/proc`, `/var/log`) |
| `UEDR/FileHashing/ignored_extensions_linux` | Extensions never hashed (includes `.txt`, `.log`) |
| `mxa/log/level` | XClient log level |

**Read the XClient log for the last N seconds, filtered.** The example covers 2 hours:
```
oc exec -n <namespace> <agent-pod> -- sh -c "/opt/Trellix/XClient/bin/xclient -G -T 7200 2>&1 | grep -iE 'contain|quarantin|networkflow|config_manager' | tail -n 400" > xclient_log.txt
```
- `config_manager` lines show every config value the agent applies, for example
  `m=mxa/eventor/excludedPaths=...`. They are the fastest proof that a policy change arrived.
- The XClient log does **not** list individual process, file, or network events, even at
  `DEBUG`. Don't use it to decide whether an event was collected (see Section 7).

**Unlock a contained node from inside the pod,** using the unlock code configured in the policy:
```
oc exec -n <namespace> <agent-pod> -- /opt/Trellix/XClient/bin/uncontain <unlock-code>
```

**Processes in the pod:**
```
oc exec -n <namespace> <agent-pod> -- sh -c "ps -ef | grep -iE '[x]client|[e]ventor|[m]asvc'"
```

---

## 3. Shell on the node (host)

`oc debug node` starts a temporary debug pod on the node. `chroot /host` switches into the
node's own filesystem.

**One command** (the debug pod is removed afterwards):
```
oc debug node/<node-name> -- chroot /host uname -r
oc debug node/<node-name> -- chroot /host bash -c "hostname; ip -4 addr show | grep inet"
```

**Interactive shell:**
```
oc debug node/<node-name>
chroot /host
```

**Things to know on RHCOS nodes:**
- `/usr` is read-only.
- Use `/var/tmp` for test files.
- `/tmp` works, but the agent ignores it for file hashing.

**Firewall state on the node,** for example to check whether containment added rules:
```
oc debug node/<node-name> -- chroot /host bash -c "nft list tables"
oc debug node/<node-name> -- chroot /host bash -c "nft list ruleset | grep -inE -B3 'trellix|xclient|quarant|contain' | head -n 200"
oc debug node/<node-name> -- chroot /host bash -c "iptables-save | grep -iE 'trellix|xclient|quarant' | head -n 100"
```
A normal OVN-Kubernetes node shows tables such as `ovn-kubernetes`, `nat`, `filter`, and
`mangle`. Trellix-specific rules appear only while containment is enforced.

**Persisted agent data** (survives pod restarts; cleared once by a fresh install from build
50.3.6.35):
```
oc debug node/<node-name> -- chroot /host ls -lrth /mnt/XClientLogs/
```

---

## 4. Custom test pods pinned to a node

Use the cluster's own `openshift/tools` image. It includes `bash`, `ping`, and `curl`, needs no
external registry access, and lets an unprivileged pod send ping.

**1. Get the image reference:**
```
oc get istag -n openshift tools:latest -o jsonpath="{.image.dockerImageReference}"
```

**2. Create the namespace:**
```
oc create namespace <test-namespace>
```

**3. Save the manifest as `custom-pod.yaml`.**
- Replace `<tools-image>` with the output of step 1.
- Repeat the document (separated by `---`) once per node, each with its own `name` and
  `nodeName`.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-pod-1
  namespace: <test-namespace>
  labels:
    app: edrf-qa-custom
spec:
  nodeName: <node-name>
  restartPolicy: Never
  terminationGracePeriodSeconds: 5
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: test
    image: <tools-image>
    command: ["sleep", "infinity"]
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
```
`nodeName` places the pod on that node directly, without the scheduler.

**4. Apply and wait:**
```
oc apply -f custom-pod.yaml
oc wait --for=condition=Ready pod/custom-pod-1 -n <test-namespace> --timeout=300s
oc get pods -n <test-namespace> -o wide
```
**Expected:** `1/1 Running` on the node you chose. A `Warning: would violate PodSecurity`
line, if one appears, is only a warning.

**5. Use the pod:**
```
oc exec -it -n <test-namespace> custom-pod-1 -- bash
oc exec -n <test-namespace> custom-pod-1 -- ping -c 3 -W 2 <ip>
```

**6. Remove all test pods:**
```
oc delete namespace <test-namespace>
```

---

## 5. Network reachability checks (node and pod)

Run each check from the node (as a host process) and from a pod on the same node (in the pod
network). Containment and whitelist behaviour can differ between the two.

| Check | From the node | From a test pod |
|---|---|---|
| ICMP | `oc debug node/<node-name> -- chroot /host ping -c 3 -W 2 <ip>` | `oc exec -n <test-namespace> custom-pod-1 -- ping -c 3 -W 2 <ip>` |
| TCP connect | `oc debug node/<node-name> -- chroot /host bash -c "timeout 5 bash -c 'exec 3<>/dev/tcp/<ip>/<port>' && echo OPEN \|\| echo CLOSED"` | `oc exec -n <test-namespace> custom-pod-1 -- bash -c "timeout 5 bash -c 'exec 3<>/dev/tcp/<ip>/<port>' && echo OPEN \|\| echo CLOSED"` |
| HTTPS | `oc debug node/<node-name> -- chroot /host curl -s -o /dev/null -w "%{http_code}\n" --max-time 8 https://<host>` | `oc exec -n <test-namespace> custom-pod-1 -- curl -s -o /dev/null -w "%{http_code}\n" --max-time 8 https://<host>` |
| One UDP packet | `oc debug node/<node-name> -- chroot /host bash -c "echo test > /dev/udp/<ip>/<port>"` | `oc exec -n <test-namespace> custom-pod-1 -- bash -c "echo test > /dev/udp/<ip>/<port>"` |

Notes:
- **Ping:** `0% packet loss` means reachable. `100% packet loss` means blocked or no route.
- **UDP:** the command has no reply to check. It's for generating a UDP network event, for
  example to trigger an HX IOC on `ipv4NetworkEvent/protocol` = `udp`.
- **Contained node:** while the node is contained, `oc debug` and `oc exec` to it can fail.
  That is a side effect of containment, not a test result. Run long-running probes (a loop
  that logs with timestamps) that you start before containment, and read their logs after
  release.
- **Control node:** compare against a node that isn't contained. It shows that a block on the
  test node comes from containment and not from the network.

---

## 6. Add or remove a worker node (Machine API)

Use this on IPI clusters (GCP, AWS, Azure, vSphere) where workers come from a MachineSet. The
new node gets an agent pod automatically from the DaemonSet.

```
oc get machinesets -n openshift-machine-api
oc get machines -n openshift-machine-api -o wide
```

**Add one node.** Set `--replicas` to the current count + 1:
```
oc scale machineset <machineset> -n openshift-machine-api --replicas=<current+1>
oc get machines -n openshift-machine-api -w
oc get nodes -w
```
**Expected:**
- The new Machine goes `Provisioning` → `Provisioned` → `Running`.
- A new node turns `Ready` in about 5–10 minutes.
- An agent pod starts on it (`oc get pods -n <namespace> -o wide`).

**Remove that node again.** Mark the machine first. Otherwise the MachineSet deletes a random
one:
```
oc annotate machine <new-machine-name> -n openshift-machine-api machine.openshift.io/delete-machine=true
oc scale machineset <machineset> -n openshift-machine-api --replicas=<previous count>
```

A new node creates a new system in ePO and a new host in HX. Removing the node leaves the
stale entries behind. Delete them so that later policy assignment and HX actions don't hit
the wrong entry.

---

## 7. Process and file exclusion tests (host processes)

**Goal:** prove that an excluded process path and an excluded file path stop producing events,
while identical controls that aren't excluded still produce them.

Don't use a copy of `bash` as the unique process:
- Shells and their child processes can be monitored as a group, so the test wouldn't be
  isolated.
- Use a renamed copy of `cp`, started directly by `chroot` with no shell, so its parent is
  `chroot`.

| Item | Absolute path | Excluded |
|---|---|---|
| Unique process | `/var/tmp/esexcl/esexcl_copier` | Yes (process exclusion) |
| Control process | `/var/tmp/esexcl/esexcl_ctrlcopier` | No |
| Unique file | `/var/tmp/esexcl/excluded/esexcl_file.dat` | Yes (file exclusion) |
| Control file | `/var/tmp/esexcl/notexcluded/esexcl_file.dat` | No |

The files use `.dat` and live in `/var/tmp`, because `.txt`, `.log`, and `/tmp` are already in
the agent's ignore lists.

**Setup (once):**
```
oc debug node/<node-name> -- chroot /host mkdir -p /var/tmp/esexcl/excluded /var/tmp/esexcl/notexcluded /var/tmp/esexcl/out
oc debug node/<node-name> -- chroot /host cp /usr/bin/cp /var/tmp/esexcl/esexcl_copier
oc debug node/<node-name> -- chroot /host cp /usr/bin/cp /var/tmp/esexcl/esexcl_ctrlcopier
oc debug node/<node-name> -- chroot /host ls -lR /var/tmp/esexcl
```

**Trigger.** Run the same five commands in every phase, and note the time the first one prints:
```
oc debug node/<node-name> -- chroot /host date -u
oc debug node/<node-name> -- chroot /host /var/tmp/esexcl/esexcl_copier /etc/os-release /var/tmp/esexcl/out/copier_out.dat
oc debug node/<node-name> -- chroot /host /var/tmp/esexcl/esexcl_ctrlcopier /etc/os-release /var/tmp/esexcl/out/ctrlcopier_out.dat
oc debug node/<node-name> -- chroot /host cp /etc/os-release /var/tmp/esexcl/excluded/esexcl_file.dat
oc debug node/<node-name> -- chroot /host cp /etc/os-release /var/tmp/esexcl/notexcluded/esexcl_file.dat
```

**Policy values to enter in ePO.** They arrive in the agent as `mxa/eventor/excludedProcessNames`
and `mxa/eventor/excludedPaths`:
```
/var/tmp/esexcl/esexcl_copier
/var/tmp/esexcl/excluded/esexcl_file.dat
```

**Confirm the agent received them.** After `cmdagent -c` and `-e` (Section 2):
```
oc exec -n <namespace> <agent-pod> -- sh -c "/opt/Trellix/XClient/bin/xclient -x /tmp/c.txt >/dev/null 2>&1; grep -i eventor/excluded /tmp/c.txt; rm -f /tmp/c.txt"
```
The XClient log also shows them, as `config_manager.cc ... m=mxa/eventor/excludedPaths=...`.

**Check whether the events were collected.** The XClient log isn't a valid check (Section 2).
Use one of these:

- **HX:** run a triage collection on the host, then search its event timeline for `esexcl`
  after the trigger time. Or create HX IOCs on `processEvent/processPath` and
  `fileWriteEvent/fullPath`.
- **The local Eventor database** (`mxa/eventor/db_in_memory:"false"`, SQLite in WAL mode).
  This command lists every `.db` / `.db-wal` file in the agent pod and counts each test string.
  Compare the counts before and after each trigger. Not yet validated on every build.
  ```
  oc exec -n <namespace> <agent-pod> -- sh -c "for f in $(find / \( -path /proc -o -path /sys -o -path /dev -o -path /host \) -prune -o -type f \( -name '*.db' -o -name '*.db-wal' \) -print 2>/dev/null); do echo == $f $(du -k $f | cut -f1)KB; for s in esexcl_copier esexcl_ctrlcopier /esexcl/excluded/esexcl_file.dat /esexcl/notexcluded/esexcl_file.dat; do echo $s $(grep -a -o $s $f | wc -l); done; done"
  ```

**Phases and expected results:**

| Phase | esexcl_copier | esexcl_ctrlcopier | excluded/esexcl_file.dat | notexcluded/esexcl_file.dat |
|---|---|---|---|---|
| 1. No exclusions | events | events | event | event |
| 2. Process exclusion | **none** | events | event | event |
| 3. Process + file exclusion | **none** | events | **none** | event |

How to read the results:
- Events in a **none** cell mean the exclusion isn't honoured.
- If the controls show no events, the phase is invalid. The problem is event collection, not
  the exclusion.
- If the absolute path doesn't exclude the process, also try just the name (`esexcl_copier`),
  because the agent key is called `excludedProcessNames`.

**Cleanup:**
1. Remove the exclusions from the policy.
2. Delete the test files:
   ```
   oc debug node/<node-name> -- chroot /host rm -rf /var/tmp/esexcl
   ```

---

## 8. Evidence collection

**Copy a file out of a pod.** Use a relative destination: on Windows, a destination that starts
with a drive letter fails with `one of src or dest must be a local file specification`.
```
oc cp -n <namespace> <agent-pod>:/tmp/<file> ./<file>
```

**Save command output straight to a file on the jumpbox:**
```
oc exec -n <namespace> <agent-pod> -- /opt/McAfee/agent/bin/cmdagent -i > agent_info.txt
oc get pods -n <namespace> -o wide > pods.txt
oc describe pod -n <namespace> <agent-pod> > pod_describe.txt
```

**Full MER log:** use the `mer-capture` skill (https://github.com/harshongh4t/mer-capture). It
runs `Trellix_EDRF_Container_LogCollector.sh` in the pod and copies the `.tar.gz` back.

---

## 9. Windows jumpbox notes

- **Running `.ps1` scripts from `cmd.exe`:** start them with
  `powershell -NoProfile -ExecutionPolicy Bypass -File .\script.ps1 ...`. Typing
  `.\script.ps1` in `cmd.exe` does nothing, or opens the file in Notepad.
- **Arrays with `-File`:** values like `-Keys a,b` arrive as one string (`"a,b"`), not as an
  array. Run the script from a PowerShell prompt if it needs an array parameter.
- **PowerShell 5.1 and `oc exec ... sh -c`:** PowerShell 5.1 strips embedded double quotes from
  arguments passed to native programs. Inside the `sh -c` script, use single quotes for
  `grep` patterns. Otherwise, a `|` inside the pattern becomes a shell pipe, and you get
  errors such as `quarantin: command not found`.
- **Console freezes:** clicking inside a `cmd.exe` or PowerShell window (QuickEdit mode) pauses
  the running command until you press a key. Don't click into a window that runs a long test.
- **Passwords:** set them in the same window (`set EPO_PASSWORD=...`) or type them at a
  prompt. Never write them into scripts, logs, or files in git.
