FROM dunglas/frankenphp:1-php8-trixie

# NOTE on layer layout (performance / disk usage):
#   * The number of RUN layers does not matter at runtime (overlayfs), but it
#     affects build time and image size through the build cache. Order layers
#     from rarely-changing to frequently-changing, and keep the total low.
#   * Everything downloaded in a layer must be cleaned IN THE SAME RUN —
#     cleaning in a later layer does not shrink earlier ones.

# All OS packages + all PHP extensions + system caches cleanup in ONE layer.
# Earlier there were 3 such runs (plus one more apt-get update hidden inside
# every install-php-extensions call), re-downloading apt lists over and over
# again and leaving a stale set of files in each layer.
RUN apt-get update ; \
  env DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
  insserv \
  procps \
  neovim \
  dropbear \
  git \
  openssh-client \
  openssh-sftp-server \
  unzip \
  curl \
  iproute2 \
  wget \
  rsync \
  sqlite3 \
  tmux \
  python3 \
  python3-pip \
  python3-uvicorn; \
  install-php-extensions \
  pdo_mysql \
  pdo_pgsql \
  pdo_sqlite \
  mysqli \
  xlswriter \
  redis \
  gd \
  intl \
  zip \
  opcache ; \
  apt-get clean ; \
  rm -rf /var/lib/apt/lists/* /var/tmp/* /tmp/*

# Node.js via fnm, in its own layer (node pulls move often). The installer and
# fnm download packages into /root/.cache and fnm's cache dir — drop them, or
# the layer keeps tens of MB of node tarballs/zips forever.
RUN curl -o- https://fnm.vercel.app/install | bash ; \
  bash -c 'source /root/.bashrc; fnm install 24; node -v' ; \
  rm -rf /root/.cache/*

EXPOSE 22 80 443

 
COPY caddy.initd /etc/init.d/caddy
COPY config/php.ini-production /usr/local/etc/php/php.ini
COPY init /sbin/init

ENTRYPOINT [ "/bin/bash", "/sbin/init" ]
