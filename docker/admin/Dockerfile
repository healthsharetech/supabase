FROM dunglas/frankenphp

RUN install-php-extensions \
    pdo_pgsql \
    gd

# QuickJS runtime (needed for todo-cli.cjs reconciliation bundle)
# Build from source for latest stable version
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        git \
        make \
        gcc \
        libc6-dev && \
    cd /tmp && \
    git clone https://github.com/bellard/quickjs.git && \
    cd quickjs && \
    git checkout b5e62895c619d4ffc75c9d822c8d85f1ece77e5b && \
    make && \
    make install && \
    cd / && \
    rm -rf /tmp/quickjs && \
    apt-get purge -y git make gcc libc6-dev && \
    apt-get autoremove -y && \
    rm -rf /var/lib/apt/lists/*

ARG USER=www-data

RUN \
	# Use "adduser -D ${USER}" for alpine based distros
	useradd -D ${USER}; \
	# Remove default capability
	setcap -r /usr/local/bin/frankenphp; \
	# Give write access to /data/caddy and /config/caddy
	chown -R ${USER}:${USER} /data/caddy && chown -R ${USER}:${USER} /config/caddy

# copy custom caddyfile
COPY Caddyfile /etc/frankenphp/Caddyfile

# RUN cp $PHP_INI_DIR/php.ini-production $PHP_INI_DIR/php.ini

USER ${USER}

