FROM dunglas/frankenphp

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
  gd \
  intl \
  zip \
  opcache


RUN curl -o- https://fnm.vercel.app/install | bash; \
  bash -c 'source /root/.bashrc; fnm install 24'


  # sysvinit-core \

EXPOSE 22 80 443

ENTRYPOINT [ "/bin/bash", "/sbin/init" ]
