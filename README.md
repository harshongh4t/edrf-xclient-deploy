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
