# php8-sshd-nodejs-container
php server container along with dropbear sshd and nodejs installed


## Usage

**compose.yml**
```yaml
services:
  www:
    image: "ghcr.io/harish2704/php8-sshd-nodejs-container:latest"
    tty: true
    volumes:
      - ./build/caddy.initd:/etc/init.d/caddy
      - ./config/php.ini-production:/usr/local/etc/php/php.ini
      - ./srvroot:/public
      - ./init:/sbin/init
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
