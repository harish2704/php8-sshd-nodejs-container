# Hosting a PHP Application with a Custom Caddyfile

This example demonstrates how to host a PHP application (and even proxy to a
secondary backend) by supplying your **own Caddyfile** instead of relying on
the environment-variable placeholders (`CADDY_GLOBAL_OPTIONS`,
`CADDY_EXTRA_CONFIG`, `CADDY_SERVER_EXTRA_DIRECTIVES`, ...) that the stock
image's Caddyfile understands.

Controlling the full Caddyfile is the cleanest way to manage the web server
when your configuration grows beyond a few environment snippets. It builds on
the base image described in the [main README](../../README.md).

---

## How it works

The FrankenPHP binary reads its configuration from `/etc/frankenphp/Caddyfile`
(see [`build/caddy.initd`](../../build/caddy.initd): `CONFIG=/etc/frankenphp/Caddyfile`).
By bind-mounting a directory containing our own `Caddyfile` over
`/etc/frankenphp`, the stock configuration is replaced entirely — every
address, route and log setting is defined by us.

### Why mount a directory instead of the single Caddyfile?

It is tempting to mount just the file (`./Caddyfile:/etc/frankenphp/Caddyfile`),
but file bind mounts can exhibit **synchronization problems**: editors that
replace the file by renaming a new inode onto the destination do **not** update
the inode the container watches, so the changes go unnoticed or cause odd
stale-content symptoms. Mounting the parent **directory**
(`./etc-frankenphp:/etc/frankenphp`) avoids these issues — new files/inodes
appear reliably, and you keep room for extra files (e.g. certificates or
fragments) if you need them later.

---

## Files

* [`etc-frankenphp/Caddyfile`](etc-frankenphp/Caddyfile) — the custom Caddy configuration.
* [`compose.yml`](compose.yml) — sample compose file wiring everything up.

---

## The Caddyfile, explained

The shipped [`etc-frankenphp/Caddyfile`](etc-frankenphp/Caddyfile) is minimal
and self-explanatory:

```
{
	# Global options block, kept empty on purpose.
}

:80 {
	log {
		output file /var/log/caddy/caddy.log {
			roll_size 10MB
			roll_keep 5
			roll_keep_for 30d
		}
		format json
	}

	handle /api/* {
		reverse_proxy 127.0.0.1:3000
	}

	handle {
		root * {$SERVER_ROOT:-/public}
		php_server
	}
}
```

* **Global options block `{...}`** — empty: `php_server` is a standard
  pre-registered Caddy directive, so no `order` directive (or any other global
  option) is needed for this minimal setup.
* **`:80` site block** — the server listens **only on HTTP port 80**. No 443
  listener is created and no automatic HTTPS/certificates are configured —
  i.e. ideal behind an existing reverse proxy / load balancer, or for local
  development.
* **`log` block** — access logs go to `/var/log/caddy/caddy.log` with sensible
  rotation: the file rolls as soon as it reaches **10 MB**, keeps the **last
  5** rolled files, and each for at least **30 days**. `format json` writes
  one structured JSON line per request (great when parsed by log tools);
  switch to `format console` for human-oriented output. Remove or trim this
  block if you prefer Caddy's default logging.
* **`handle /api/*` reverse proxy demo** — requests under `/api/` are forwarded
  to `127.0.0.1:3000`, e.g. a Node.js `node server.js` running inside the same
  container (the image ships Node.js 24 via fnm), or replace the address with
  another service.
* **`handle` fallback with `php_server`** — everything else is served as a
  classic PHP application from the web root: static files, `index.php` as the
  directory index, and PHP execution via FrankenPHP. `SERVER_ROOT` is still
  honoured through a Caddyfile environment substitution (fallback `/public`).

Note the two `handle` blocks: `/api/*` first, everything else second. Adjust
paths or add more `handle` blocks as your application grows. The stock
Caddyfile ends with `import Caddyfile.d/*.caddyfile` (per-site fragments);
since we replace the whole `/etc/frankenphp` directory, that import is gone —
add your own site blocks or extra configuration directly to this file instead.

---

## Usage

1. Have some PHP application in the web root (the repo's demo
   [`../../srvroot`](../../srvroot) with a `phpinfo()` page works).

2. Start the container:

   ```bash
   cd examples/custom-caddyfile
   podman compose up -d   # or: docker compose up -d
   ```

3. Open the app on the published port:

   ```bash
   curl http://localhost:3080/
   ```

---

## Adapting it

* **Different document root** — change `root * {$SERVER_ROOT:-/public}` (or set
  `SERVER_ROOT` in the compose file's `environment:`).
* **Real hostname** — replace `:80` with e.g. `example.com` (then Caddy adds
  automatic HTTPS — keep in mind this needs a 443 listener and certificates);
  the `log {...}` and both `handle` blocks stay inside that site block.
* **Disable the demo `/api` proxy** — delete the `handle /api/*` block once you
  have your own routing.
* **SSH into the container as well** — combine with the
  [`ssh-via-wrapper`](../ssh-via-wrapper) example (which also documents
  starting the container without its built-in sshd).
* **Reload without restart** — the container ships
  [`/etc/init.d/caddy`](../../build/caddy.initd), so after editing the
  mounted Caddyfile:

  ```bash
  podman exec -i www /etc/init.d/caddy reload
  ```
