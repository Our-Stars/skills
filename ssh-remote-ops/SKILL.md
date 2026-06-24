---
name: ssh-remote-ops
description: Use this skill when the user wants Codex to connect to one or more remote servers over SSH from the local machine, upload or download files, inspect remote paths, run remote commands, tail logs, or operate remote servers through local SSH/rsync. The skill must not assume or hardcode the remote host, username, SSH port, or local identity-file path. Use connection details already provided in the current conversation directly; ask only for missing required details.
---

# SSH Remote Operations

Use this workflow to control one or more remote servers from the local Codex environment through OpenSSH, `rsync`, and shell commands.

## Required Connection Inputs

Before connecting, confirm these inputs are known from the user request or current conversation:

- Remote host or IP.
- Remote username.
- SSH port, defaulting to `22` only if the user does not specify one.
- Local identity file path, for example `/Users/name/.ssh/key_name`.

Do not hardcode connection details in this skill. Never include real hosts, usernames, ports, or key paths in this file.

If all required values are available from the current turn or earlier conversation, do not ask again; connect directly. If only some values are missing, ask only for the missing values before running SSH or rsync.

If the user gives a public key path ending in `.pub`, do not use it with `ssh -i`. Check whether the sibling private key path without `.pub` exists and has safe permissions. If it exists, use that private key; otherwise ask the user for the private key path.

If the username contains `@`, prefer `ssh -l "<user>" "<host>"` or carefully quote `"<user>@<host>"` so OpenSSH does not split the username incorrectly.

## Safety Rules

- Prefer read-only checks first: `hostname`, `whoami`, `pwd`, `ls`, `stat`, `df`, `du`, `ps`, `tail`.
- Do not delete, overwrite, stop services, edit server config, or run destructive commands unless the user explicitly asks.
- For upload/download, use `rsync -avP` by default so progress is visible and interrupted transfers can resume.
- Use SSH keepalive options for longer work:
  `-o ServerAliveInterval=30 -o ServerAliveCountMax=3`.
- Use `BatchMode=yes` for non-interactive checks when the key is expected to work without passphrase prompts.
- Use `IdentitiesOnly=yes` when an identity file is specified so OpenSSH tries the intended key.
- Use `-F /dev/null` for diagnostic connection checks when local SSH config might interfere; omit it when the user's SSH config is intentionally required.
- If a command can hang, wrap it with `timeout` when available or run it in the background with logs and a PID file.
- If tool execution requires network or filesystem access outside the workspace sandbox, request scoped approval before connecting.

## Initial Connection Check

When connection details are complete, start with a read-only check:

```bash
ssh -F /dev/null \
  -i "<identity_file>" \
  -p "<port>" \
  -l "<user>" \
  -o IdentitiesOnly=yes \
  -o BatchMode=yes \
  -o ConnectTimeout=12 \
  -o ServerAliveInterval=30 \
  -o ServerAliveCountMax=3 \
  "<host>" 'hostname; whoami; pwd'
```

If this succeeds, report the remote hostname, user, and working directory. Continue with the user's requested task.

If the user asks to work on multiple servers and provides complete connection details for each one, run independent read-only checks in parallel when the tools allow it. Report per-server success/failure separately.

Use placeholders only in examples. Do not record real connection details in the skill.

## Common Tasks

List a remote directory:

```bash
ssh -i "<identity_file>" -p "<port>" -l "<user>" "<host>" 'ls -la "<remote_path>"'
```

Upload a file or directory:

```bash
rsync -avP \
  -e "ssh -i <identity_file> -p <port> -l <user> -o IdentitiesOnly=yes -o ServerAliveInterval=30 -o ServerAliveCountMax=3" \
  "<local_path>" "<host>:<remote_path>"
```

Download a file or directory:

```bash
rsync -avP \
  -e "ssh -i <identity_file> -p <port> -l <user> -o IdentitiesOnly=yes -o ServerAliveInterval=30 -o ServerAliveCountMax=3" \
  "<host>:<remote_path>" "<local_path>"
```

Run a short remote command:

```bash
ssh -i "<identity_file>" -p "<port>" -l "<user>" "<host>" 'cd "<remote_dir>" && <command>'
```

Run a long remote task with logs:

```bash
ssh -i "<identity_file>" -p "<port>" -l "<user>" "<host>" \
  'cd "<remote_dir>" && nohup <command> > run.log 2>&1 & echo $! > run.pid'
```

Check long-task status:

```bash
ssh -i "<identity_file>" -p "<port>" -l "<user>" "<host>" \
  'cd "<remote_dir>" && ps -p "$(cat run.pid)" -o pid,etime,cmd; tail -n 80 run.log'
```

## Failure Handling

- If the TCP connection succeeds but SSH closes before authentication, retry once with verbose SSH (`-vv`) and/or `ssh-keyscan` to distinguish banner/KEX issues from key rejection.
- If SSH says `Permission denied`, verify username, identity file, server `authorized_keys`, and file permissions.
- If SSH asks for a passphrase in a non-interactive Codex run, ask the user to add the key to `ssh-agent`/keychain or provide a no-passphrase key.
- If the host is unreachable, check IP, port, LAN/VPN connectivity, firewall, and whether `sshd` is running.
- If a proxy/VPN blocks the server but the agent needs proxy for other services, suggest rule-based routing: keep the server host/IP direct and let other traffic follow the user's proxy rules. Do not edit proxy rules unless the user explicitly asks.
