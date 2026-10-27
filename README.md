# op-agent-vault-windows

**Windows only.** A Claude Code / Codex skill that sets up a Windows machine so every AI agent on it reads, creates, edits and deletes items in one dedicated 1Password vault through a service account, with no password, Windows Hello or approval prompts. Other vaults stay protected.

What the skill does:

1. Inspects the current setup: `op`, the token, the service account, its vaults, and the agent configs.
2. Has the user create a service account with read and write on the agents' vault (browser, one time).
3. Stores the token as a persistent user environment variable, so every agent process inherits it.
4. Adds `op` permission and auto-mode rules to `~/.claude/settings.json`, so Claude Code stops asking and stops blocking writes.
5. Writes shared rules to `~/.claude/CLAUDE.md` and `~/.codex/AGENTS.md`: use the token, never clear it, never print secrets.
6. Verifies with a fresh process and a create → edit → delete test item.

Secret values are never printed.

## Install

Claude Code:

```text
/plugin marketplace add kyzdes/claude-skills
/plugin install op-agent-vault-windows@claude-skills
```

Codex:

```bash
codex plugin marketplace add https://github.com/kyzdes/claude-skills.git
codex plugin add op-agent-vault-windows@claude-skills
```

Then ask, for example: "Set up 1Password on this Windows machine so agents can write to the Development vault without prompts."

## Requirements

- Windows 10/11, the 1Password CLI (`winget install AgileBits.1Password.CLI`).
- A 1Password account where you can create service accounts.

macOS and Linux are not covered.

## License

MIT
