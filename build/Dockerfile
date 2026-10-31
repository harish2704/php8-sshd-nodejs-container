FROM dunglas/frankenphp:1-php8-trixie

RUN apt-get update ;\
  env DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
  insserv \
  procps \
  neovim \
  dropbear \
  git \
  openssh-client \
  unzip \
  curl \
  iproute2 \
  wget; \
  install-php-extensions \
  pdo_mysql \
  pdo_pgsql \
  pdo_sqlite \
  xlswriter \
  redis \
  gd \
  intl \
  zip \
  opcache


RUN curl -o- https://fnm.vercel.app/install | bash; \
  bash -c 'source /root/.bashrc; fnm install 24; node -v'


RUN apt-get update ; apt-get install -y --no-install-recommends \
  openssh-sftp-server \
  rsync \
  python3 \
  python3-pip \
  python3-uvicorn

RUN install-php-extensions \
  mysqli

  # sysvinit-core \

EXPOSE 22 80 443

 
COPY caddy.initd /etc/init.d/caddy
COPY config/php.ini-production /usr/local/etc/php/php.ini
COPY init /sbin/init

ENTRYPOINT [ "/bin/bash", "/sbin/init" ]
