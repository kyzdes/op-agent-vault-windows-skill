---
name: op-agent-vault-windows
description: Windows only. Set up a Windows machine so every AI agent (Claude Code, Codex, Hermes, any shell) reads, creates, edits and deletes items in a dedicated 1Password vault through a service account, with no password, Windows Hello or approval prompts. Use when the user wants agents to use 1Password "without asking", when `op` pops up the 1Password desktop unlock or Windows Hello, when Claude Code blocks `op item create/edit/delete` as a secret-store write, or says "настрой 1Password для агентов на Windows", "чтобы агенты писали в Development vault без подтверждений", "убери окно пароля 1Password", "сервисный аккаунт 1Password для агентов". Also for auditing the current agent ↔ 1Password setup on Windows.
---

# 1Password vault for agents on Windows, without prompts

Goal: one 1Password vault (here called `<VAULT>`, e.g. `Development`) belongs to the agents. Every agent on this Windows user account can read and write it with no prompts. Other vaults stay protected.

The method has three parts, and prompts come from whichever part is missing:

| Prompt the user sees | Cause | Fix |
|---|---|---|
| 1Password unlock / Windows Hello window | `op` falls back to the desktop app because no service-account token is set, or an agent cleared it or passed `--account` | Step 2: persistent user env `OP_SERVICE_ACCOUNT_TOKEN`; Step 4: rule "never clear it" |
| Claude Code "permission denied … Secret-Store Writes" or an approval prompt | Auto-mode classifier and permission rules treat 1Password writes as risky | Step 3: `permissions.allow` + `autoMode.allow` |
| `op` error "not authorized" / "permission denied" from 1Password | The service account has read-only access to `<VAULT>` | Step 1: a service account with read and write |

Never print a token or secret value. Read values into variables, report only lengths, prefixes such as `ops_ey…`, or yes/no comparisons.

## 0. Inspect the current state (read-only)

```powershell
Get-Command op -ErrorAction SilentlyContinue | Select-Object Source; op --version
$t = [Environment]::GetEnvironmentVariable('OP_SERVICE_ACCOUNT_TOKEN','User'); "user env token: " + $(if ($t) { $t.Substring(0,6) + "… ($($t.Length) chars)" } else { 'missing' })
op whoami                 # User Type must be SERVICE_ACCOUNT
op user get --me          # service account name, state, last auth
op vault list             # vaults it can see
op service-account ratelimit
Get-Content "$env:USERPROFILE\.claude\settings.json"
Test-Path "$env:USERPROFILE\.claude\CLAUDE.md", "$env:USERPROFILE\.codex\AGENTS.md"
```

If `op` is missing: `winget install AgileBits.1Password.CLI`, then open a new shell.

`op` cannot show a service account's per-vault permissions. Write access is proven only by a test write (step 5) or by the 1Password web console (Developer → Service Accounts).

## 1. Service account (the user does this in the browser)

Agents cannot create a service account or its token; that needs the user's own 1Password session. Ask the user to:

1. Open 1Password.com → Developer → Service Accounts → New service account.
2. Grant `<VAULT>` with **read and write** (add other vaults read-only if agents need them).
3. Save the token as an item in `<VAULT>` (e.g. `op-service-account-<machine>`, field `credential`). It is shown only once.

Vault permissions are chosen when the account is created; if an existing account lacks write, create a new one rather than hunting for a toggle.

## 2. Make the token persistent for this Windows user

The token goes into the **user** environment (`HKCU\Environment`), so every process the user starts later inherits it: Claude desktop, Codex, terminals, Hermes.

If no token is set yet, reading the token item once goes through the desktop app (one Windows Hello prompt, the last one):

```powershell
$tok = op read "op://<VAULT>/<token item>/credential"
[Environment]::SetEnvironmentVariable('OP_SERVICE_ACCOUNT_TOKEN', $tok, 'User')
$env:OP_SERVICE_ACCOUNT_TOKEN = $tok; Remove-Variable tok
```

If the user pastes the token instead, run the same `SetEnvironmentVariable` with it; never echo it back.

Processes started before this keep the old environment. Tell the user to fully restart the Claude desktop app, Codex, terminals and agent daemons (e.g. Hermes gateway).

Do not put the token in `~/.claude/settings.json` `env`, `.env` files or repos: the user env var already reaches every agent, and a second copy is one more place to leak and to forget at rotation.

## 3. Claude Code: no permission prompts, no classifier blocks

Merge into `~/.claude/settings.json` (read it first; keep existing keys and array items):

```json
{
  "permissions": {
    "allow": ["PowerShell(op *)", "Bash(op *)"]
  },
  "autoMode": {
    "allow": [
      "$defaults",
      "1Password CLI (`op`) on this machine runs as the service account <SA NAME> (user env OP_SERVICE_ACCOUNT_TOKEN). The user has explicitly authorized agents to read, create, edit, archive and delete items in the 1Password vault \"<VAULT>\" (made for agents) without asking, including reading secret values into variables or files the task needs."
    ]
  }
}
```

- `permissions.allow` covers a single `op …` command. A compound command (several statements, pipes into other tools) does not match the prefix rule and goes to the auto-mode classifier, so the `autoMode.allow` sentence is what lets those through.
- `"$defaults"` keeps the built-in rules. Without it you replace them.
- Writes to vaults not named in the sentence stay blocked. That is intended; widen the sentence only if the user asks.
- Validate the JSON after editing: `Get-Content $f -Raw | ConvertFrom-Json | Out-Null`. A broken file silently disables all its settings. The change applies to running sessions without a restart.

## 4. Rules every agent reads

Write the same block to `~/.claude/CLAUDE.md` (Claude Code) and `~/.codex/AGENTS.md` (Codex). Append to existing files, do not overwrite them. Other agents (Hermes and the like) get it in their own global instructions file.

```markdown
## 1Password
- All secrets live in 1Password. `op` on this machine runs as the service account `<SA NAME>` (user env `OP_SERVICE_ACCOUNT_TOKEN`), with no Windows Hello prompts.
- Vault `<VAULT>` is made for agents: read, create, edit and delete items there freely, without asking the user.
- Never clear or override `OP_SERVICE_ACCOUNT_TOKEN` and never pass `--account`: that switches `op` to the desktop app and makes the user type a password.
- Read values into a variable (`op read "op://<VAULT>/<item>/<field>"`), never print them in chat or logs.
- Other vaults: read only; ask before writing.
```

Remove any older rule telling agents to clear the token for writes. That rule is what produced the password window.

Codex: `approval_policy = "never"` in `~/.codex/config.toml` means no approval prompts; `op` also needs `[sandbox_workspace_write] network_access = true`. Test `op whoami` from a Codex task before relying on it.

## 5. Verify

Run each as a separate command, so a failure points at one step:

```powershell
powershell -NoProfile -Command "`$env:OP_SERVICE_ACCOUNT_TOKEN=[Environment]::GetEnvironmentVariable('OP_SERVICE_ACCOUNT_TOKEN','User'); op whoami"
op item create --vault <VAULT> --category "Secure Note" --title agent-write-test "notesPlain=safe to delete" --format json
op item edit agent-write-test --vault <VAULT> "notesPlain=edited"
op item delete agent-write-test --vault <VAULT>
```

Pass criteria: `User Type: SERVICE_ACCOUNT`, create/edit/delete succeed, no window appears, Claude Code does not ask. Confirm the test item is gone. Report what was verified and what was not (e.g. Codex or Hermes not tested).

## Notes

- Rate limits per service account: about 1000 reads and 100 writes per hour, plus a daily account quota. Check with `op service-account ratelimit` when agents batch many writes.
- Rotation: create a new service account, store its token in `<VAULT>`, repeat step 2, restart agents, then delete the old account in the web console. Update the account name in the step 3 and 4 texts.
- A leaked token gives exactly the vault access granted in step 1. Keep the agents' vault free of secrets that agents must not have.
