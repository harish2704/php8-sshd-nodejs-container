# php8-sshd-nodejs-container
php server container along with dropbear sshd and nodejs installed


## Examples

The [`examples/`](examples) directory contains ready-to-run variants of this
setup:

* [`examples/ssh-via-wrapper`](examples/ssh-via-wrapper) — give users direct
  SSH access into a running container via the host's sshd and a
  forced-command wrapper, without giving them access to the host system or
  the container runtime. Also shows how to run the container with a custom
  init script that does *not* start the built-in dropbear sshd.

* [`examples/custom-caddyfile`](examples/custom-caddyfile) — host a PHP app
  with your own Caddyfile, mounted as a directory over `/etc/frankenphp`
  (instead of configuring Caddy through environment-variable placeholders).
  Includes a file-based access log with rotation, a plain-PHP hosting demo
  and a reverse proxy demo.


## Usage

**compose.yml**
```yaml
services:
  www:
    image: "ghcr.io/harish2704/php8-sshd-nodejs-container:latest"
    tty: true
    volumes:
      # - ./build/caddy.initd:/etc/init.d/caddy
      # - ./build/config/php.ini-production:/usr/local/etc/php/php.ini
      # - ./build/init:/sbin/init
      - ./srvroot:/public
      - ./authorized_keys:/root/.ssh/authorized_keys
    # entrypoint: ["/bin/bash", "-c", "trap \"echo Shutting down; exit 0\" SIGTERM SIGINT SIGKILL; /bin/sleep infinity & wait"]
    ports:
      - "3022:22"
      - "3080:80"
      - "3443:443"
    environment:
      CADDY_GLOBAL_OPTIONS: ""
      FRANKENPHP_CONFIG: ""
      CADDY_EXTRA_CONFIG: ""
      SERVER_NAME: ":80"
      SERVER_ROOT: /public
      MERCURE_TRUSTED_ISSUERS: "https://localhost"
      MERCURE_EXTRA_DIRECTIVES: ""
      CADDY_SERVER_EXTRA_DIRECTIVES: ""

```
