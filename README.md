# eDRF XClient Container Security — Deployment Resources

## References

1. [Helm Chart — eDRF XClient Container Security](https://github.trellix.com/unified-agent/XClientInstaller/tree/a6549dd1dfff3dd4afefec2957445b3aee1cd970/container-security/helm#chart-structure) — Sailesh
2. [Container Security — eDRF Deployment Guide](https://github.trellix.com/unified-agent/XClientInstaller/blob/624a8911399b436f777ba5ce8ca0d29f968feee6/container-security/README.md#deployment-steps) — Vikas M

---

## Get guided deployment help from Claude Code

This repository includes a Claude Code skill (`.claude/skills/edrf-xclient-deploy/`) built
from `DEPLOYMENT_GUIDE.md`. It lets Claude walk you through installing or troubleshooting
`edrf-xclient` interactively — enforcing the correct step order and recognizing the specific
failure modes documented in the guide — instead of you reading the guide end to end.

### Option 1 — Clone this repo and open it in Claude Code (recommended)

1. Clone the repository:
   ```bash
   git clone https://github.com/harshongh4t/edrf-xclient-deploy.git
   ```
2. Open the cloned folder in Claude Code (the `claude` CLI, or the VS Code/JetBrains
   extension's **Open Folder**).
3. The skill loads automatically for any session started in this folder. Ask Claude to help
   deploy or troubleshoot `edrf-xclient`, or invoke it directly by name:
   ```
   /edrf-xclient-deploy
   ```

### Option 2 — Install it as a personal skill (available in every project)

Use this if you want the skill available outside this repo, in any folder you open with
Claude Code.

1. Copy the skill folder and the deployment guide into your user-level Claude Code skills
   directory, keeping the same relative layout:

   **Windows (PowerShell or cmd.exe):**
   ```
   mkdir "%USERPROFILE%\.claude\skills\edrf-xclient-deploy"
   copy DEPLOYMENT_GUIDE.md "%USERPROFILE%\.claude\skills\edrf-xclient-deploy\"
   copy .claude\skills\edrf-xclient-deploy\SKILL.md "%USERPROFILE%\.claude\skills\edrf-xclient-deploy\"
   ```
   **Linux/Mac:**
   ```bash
   mkdir -p ~/.claude/skills/edrf-xclient-deploy
   cp DEPLOYMENT_GUIDE.md ~/.claude/skills/edrf-xclient-deploy/
   cp .claude/skills/edrf-xclient-deploy/SKILL.md ~/.claude/skills/edrf-xclient-deploy/
   ```
2. Start a new Claude Code session (in any project). The skill is now available everywhere,
   not just inside this repo.

Either way, no further configuration is required — Claude Code discovers skills under
`.claude/skills/` (project-level) or `~/.claude/skills/` (personal) on its own.

---

## Updating the skill

The skill and `DEPLOYMENT_GUIDE.md` are updated whenever the deployment process changes (for
example, the move to the bundle and `deploy.sh` flow). Claude Code doesn't update a skill on
its own, so pull the latest version using the method that matches how you installed it.

**Note:** this repository's commit history was rewritten once to remove sensitive values. If
you cloned it **before** that, a normal `git pull` fails or reports diverged branches. Use the
one-time reset in step 1 below instead.

### If you used Option 1 (cloned repo)

1. Go to your clone and get the latest version:
   ```bash
   cd edrf-xclient-deploy
   git pull
   ```
   **If you cloned before the history rewrite**, run this once instead. It discards any local
   changes in the clone:
   ```bash
   git fetch origin
   git reset --hard origin/main
   ```
   Or delete the folder and clone again:
   ```bash
   git clone https://github.com/harshongh4t/edrf-xclient-deploy.git
   ```
2. Start a **new** Claude Code session in the folder. A session that was already open keeps
   the old version of the skill.

### If you used Option 2 (personal skill)

1. Get the latest repository files (step 1 above, or download the repository as a ZIP from
   GitHub and extract it).
2. From the repository folder, copy the updated files over your existing copies:

   **Windows (cmd.exe):**
   ```
   copy /Y DEPLOYMENT_GUIDE.md "%USERPROFILE%\.claude\skills\edrf-xclient-deploy\"
   copy /Y .claude\skills\edrf-xclient-deploy\SKILL.md "%USERPROFILE%\.claude\skills\edrf-xclient-deploy\"
   ```
   **Windows (PowerShell):**
   ```powershell
   Copy-Item DEPLOYMENT_GUIDE.md "$env:USERPROFILE\.claude\skills\edrf-xclient-deploy\" -Force
   Copy-Item .claude\skills\edrf-xclient-deploy\SKILL.md "$env:USERPROFILE\.claude\skills\edrf-xclient-deploy\" -Force
   ```
   **Linux/Mac:**
   ```bash
   cp DEPLOYMENT_GUIDE.md ~/.claude/skills/edrf-xclient-deploy/
   cp .claude/skills/edrf-xclient-deploy/SKILL.md ~/.claude/skills/edrf-xclient-deploy/
   ```
3. Start a **new** Claude Code session.

### Confirm you have the latest version

Open `SKILL.md` (in your clone, or in your personal skills folder) and check that it mentions
**`extract-coninfo.sh`** and the **`epo-coninfo`** secret. If it doesn't, or if it still
describes `--epo-ip` / `--epo-password` flags or a manual `helm install`, you have an old
version. Repeat the steps above.

You can also ask Claude directly: invoke `/edrf-xclient-deploy` and ask how to deploy. The
answer should start with installing the tools (including Git for Windows on Windows), include
creating the `epo-coninfo` secret from the ePO SmartInstaller, and use `deploy.sh` without any
ePO flags.
