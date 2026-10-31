# SSH Access to Containers via a Forced-Command Wrapper

This example demonstrates how to give users **direct SSH access into a running
container** — without giving them access to the host system, the host shell,
or the container runtime (docker/podman).

It is the missing piece for setups like the [`php8-sshd-nodejs-container`](../../README.md)
or any other containerized service: instead of publishing one SSH port per
container, a **single SSH server on the host** authenticates users and forwards
them straight into the container assigned to them.

> **See also:** [`../custom-caddyfile`](../custom-caddyfile) — serve your PHP
> application with a custom Caddyfile instead of the image's stock one.

---

## Why this approach?

The naive way to run "SSH into a container" is to expose the container's SSH
port. That works, but it has drawbacks:

* You need to **run a full sshd inside every container** (or use dropbear),
  manage its host keys, and harden each of them.
* You need **one published port per user/container**.
* Users still need an account and authorized keys inside each container image.

With this wrapper you get:

* **No host shell access** — the user's key is locked to a single command
  (`command=` option in `authorized_keys`), so even if it is used
  interactively, the user never lands on the host.
* **No container-runtime privileges** — the user never needs to be in the
  `docker` group or have podman access.
* **Centralized authentication** — one host sshd, one `authorized_keys`
  database, standard key management.
* **Full feature support** — interactive shells, `scp`, `sftp`, `rsync` and
  plain remote commands all work, because SSH clients signal these through
  `SSH_ORIGINAL_COMMAND`.
* **Auditable sessions** — every invocation is logged with user, container,
  client IP and requested command.

---

## How it works

OpenSSH exposes the command requested by the client in the special environment
variable `SSH_ORIGINAL_COMMAND`. When you create an `authorized_keys` entry
with a `command="..."` restriction, that forced command runs instead of the
user's shell/login request, and `SSH_ORIGINAL_COMMAND` tells it what the
client actually asked for.

The `ssh-container-shell` script in this directory exploits that:

| Client request                        | Detected via `SSH_ORIGINAL_COMMAND`                | Wrapper action                                            |
| ------------------------------------- | -------------------------------------------------- | --------------------------------------------------------- |
| Interactive login (`ssh host`)        | empty                                              | `podman exec -it <container> /bin/bash`                   |
| `scp file host:/path`                 | starts with `scp `                                 | `podman exec -i <container> <command>` (legacy SCP)       |
| `sftp` / rsync over ssh / file managers| starts with `/usr/lib/openssh/sftp-server` or `rsync` | `podman exec -i <container> <command>`                  |
| Remote command (`ssh host cmd`)       | anything else                                      | `podman exec <container> /bin/bash -c "<command>"`        |

Everything after the container name comes from the *host's* SSH server, so the
container doesn't need sshd at all — only `bash` and the usual tools.

---

## Files

* [`ssh-container-shell`](ssh-container-shell) — the forced-command wrapper script.
* [`init`](init) — custom container init script, same as the stock
  [`build/init`](../../build/init) but **without the dropbear SSH server**,
  since no sshd is needed inside the container.
* [`compose.yml`](compose.yml) — sample compose file using that custom init
  script.

---

## Requirements

* A standard host `sshd` (OpenSSH) where users connect.
* `podman` (or `docker` — just replace `podman` with `docker` in the script)
  available to the user running the wrapper.
* At least one running container, referenced by name.
* The wrapper script installed somewhere the forced command can execute it,
  e.g. `~/.local/bin/ssh-container-shell`. (If it must work for multiple
  system users, install it in a shared path such as `/usr/local/bin/`.)

---

## Setup

### 1. Run the container without its own sshd

Use the sample compose file from this directory. It mounts the custom `init`
script (dropbear removed) over `/sbin/init` and — since users connect through
the host sshd instead — does **not** publish port 22. The web services still
publish their usual ports (adjust as needed):

```yaml
services:
  www:
    image: "ghcr.io/harish2704/php8-sshd-nodejs-container:latest"
    tty: true
    volumes:
      # custom init script without the dropbear sshd server
      - ./init:/sbin/init
      # web root (paths are relative to this compose file)
      - ../../srvroot:/public
    ports:
      - "3080:80"
      - "3443:443"
    environment:
      SERVER_NAME: ":80"
      SERVER_ROOT: /public
      # ... remaining Caddy/Mercure environment variables omitted here —
      # see compose.yml for the full set

```

With this, the image's stock sshd (dropbear) never starts, which removes one
daemon, its host keys and its attack surface from every container.

### 2. Start the container

```bash
cd examples/ssh-via-wrapper
podman compose up -d   # or: docker compose up -d
```

### 3. Install the wrapper
install -m 0755 ssh-container-shell ~/.local/bin/ssh-container-shell
```

### 4. Add a forced-command entry to `authorized_keys`

Each key gets its own forced command, pinning that key to exactly one
container:

```bash
command="~/.local/bin/ssh-container-shell tmp-os-1" ssh-ed25519 AAAAC3Nza... user@host
```

* `tmp-os-1` is the **container name** the key is allowed to enter.
  Use a different entry (or a different line) per user/container.
* Because of the `command=` restriction, this key can **only** enter
  `tmp-os-1` — it cannot get a host shell, change the container, or run
  arbitrary host commands.

Example multi-user layout in `authorized_keys`:

```
command="~/.local/bin/ssh-container-shell tmp-os-1"   ssh-ed25519 AAAA... alice@dev
command="~/.local/bin/ssh-container-shell tmp-os-2"   ssh-ed25519 AAAA... bob@dev
```

> **Tip:** For a shared account, you can additionally restrict each entry with
> `restrict,no-port-forwarding,no-X11-forwarding,no-agent-forwarding`, e.g.:
>
> ```
> command="~/.local/bin/ssh-container-shell tmp-os-1",no-port-forwarding ssh-ed25519 AAAA... alice@dev
> ```

### 5. Connect

```bash
# Interactive shell inside the container
ssh alice@host

# Copy files in / out (scp)
scp -O ./index.php alice@host:/public/

# SFTP / rsync / GUI file managers (GVFS, WinSCP, FileZilla, ...)
rsync -av ./project/ alice@host:/public/project/

# Run a one-off command
ssh alice@host 'php -v'
```

Everything executes **inside the container**, as the container's user...

---

## Session logging

The wrapper appends one entry per connection to
`/tmp/ssh-container-wrapper-<container>-<user>.log`, containing:

* timestamp and PID,
* invoking OS user,
* container name,
* client connection info (`SSH_CONNECTION`),
* the requested command (`SSH_ORIGINAL_COMMAND`).

Handy for auditing who did what on a shared host:

```console
===== Invocation: 2026-10-31 09:15:02+0530 =====
PID: 4213
User: uid=1002(alice) gid=1002(alice) groups=1002(alice)
Container: tmp-os-1
SSH_CONNECTION: 192.168.1.20 51022 192.168.1.5 22
SSH_ORIGINAL_COMMAND: 'rsync --server -ve.LsfxC . /public/project/'
```

---

## Limitations & notes

* **PTY for remote commands** (`ssh -t host 'cmd'`) is not supported by this
  simple wrapper (`exec` without `-t`); interactive sessions and `scp`/`sftp`
  are covered.
* The container must be **running** — the wrapper runs commands with a plain
  `exec`, so a stopped container makes the connection fail with a clear
  podman/docker error.
* `scp` with modern clients uses the **SFTP protocol** by default; the legacy
  `scp` path here is for older clients / `-O` mode. SFTP access works via the
  `sftp-server` branch, provided the container image bundles an sftp-server
  binary at that path.
* The log file is written under `/tmp` — move it somewhere persistent and
  rotate it if you rely on it for auditing.
* The user (account the forced command runs as) needs permission to `exec`
  into the target container — e.g. **root**, or a **rootless podman** session
  owned by that user. This is the only host-level privilege required.
* Container **name vs. user mapping** is done purely through `authorized_keys`
  entries. Keep one SSH account (or per-line entries) per container to avoid
  confusing ownership.
