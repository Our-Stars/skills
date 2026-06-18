---
name: ssh-remote-ops
description: Use this skill when the user wants Codex to connect to a remote server over SSH from the local machine, upload or download files, inspect remote paths, run remote commands, tail logs, or operate a remote server through local SSH/rsync. The skill must not assume or hardcode the remote IP, username, SSH port, or local identity-file path; ask for any missing connection detail before connecting.
---

# SSH Remote Operations

Use this workflow to control a remote server from the local Codex environment through OpenSSH, `rsync`, and shell commands.

## Required Connection Inputs

Before connecting, confirm these inputs are known from the user request or current conversation:

- Remote host or IP.
- Remote username.
- SSH port, defaulting to `22` only if the user does not specify one.
- Local identity file path, for example `/Users/name/.ssh/key_name`.

Do not hardcode connection details in this skill. If host/IP, username, or identity file path is missing, ask the user for the missing value before running SSH or rsync. If the user provides all required values, connect directly.

## Safety Rules

- Prefer read-only checks first: `hostname`, `whoami`, `pwd`, `ls`, `stat`, `df`, `du`, `ps`, `tail`.
- Do not delete, overwrite, stop services, edit server config, or run destructive commands unless the user explicitly asks.
- For upload/download, use `rsync -avP` by default so progress is visible and interrupted transfers can resume.
- Use SSH keepalive options for longer work:
  `-o ServerAliveInterval=30 -o ServerAliveCountMax=3`.
- Use `BatchMode=yes` for non-interactive checks when the key is expected to work without passphrase prompts.
- If a command can hang, wrap it with `timeout` when available or run it in the background with logs and a PID file.

## Connection Template

Set shell variables mentally or in the command from user-provided values:

```bash
ssh -i "<identity_file>" \
  -p "<port>" \
  -o BatchMode=yes \
  -o ConnectTimeout=8 \
  -o ServerAliveInterval=30 \
  -o ServerAliveCountMax=3 \
  "<user>@<host>" 'hostname; whoami; pwd'
```

If this succeeds, report the host, user, and remote working directory.

## Common Tasks

List a remote directory:

```bash
ssh -i "<identity_file>" -p "<port>" "<user>@<host>" 'ls -la "<remote_path>"'
```

Upload a file or directory:

```bash
rsync -avP \
  -e "ssh -i <identity_file> -p <port> -o ServerAliveInterval=30 -o ServerAliveCountMax=3" \
  "<local_path>" "<user>@<host>:<remote_path>"
```

Download a file or directory:

```bash
rsync -avP \
  -e "ssh -i <identity_file> -p <port> -o ServerAliveInterval=30 -o ServerAliveCountMax=3" \
  "<user>@<host>:<remote_path>" "<local_path>"
```

Run a short remote command:

```bash
ssh -i "<identity_file>" -p "<port>" "<user>@<host>" 'cd "<remote_dir>" && <command>'
```

Run a long remote task with logs:

```bash
ssh -i "<identity_file>" -p "<port>" "<user>@<host>" \
  'cd "<remote_dir>" && nohup <command> > run.log 2>&1 & echo $! > run.pid'
```

Check long-task status:

```bash
ssh -i "<identity_file>" -p "<port>" "<user>@<host>" \
  'cd "<remote_dir>" && ps -p "$(cat run.pid)" -o pid,etime,cmd; tail -n 80 run.log'
```

## Failure Handling

- If SSH says `Permission denied`, verify username, identity file, server `authorized_keys`, and file permissions.
- If SSH asks for a passphrase in a non-interactive Codex run, ask the user to add the key to `ssh-agent`/keychain or provide a no-passphrase key.
- If the host is unreachable, check IP, port, LAN/VPN connectivity, firewall, and whether `sshd` is running.
- If Codex tool execution needs approval to access SSH or rsync outside the workspace sandbox, request scoped approval before connecting.
