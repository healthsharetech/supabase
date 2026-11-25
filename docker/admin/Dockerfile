# Pin the PHP minor: bin/deploy builds with --pull, so an unpinned tag jumps to
# whatever PHP `latest` ships. PHP 8.6 dropped mbstring from the base image and
# Laravel 12 isn't ready for its deprecations. --pull still picks up 8.4 patches.
FROM dunglas/frankenphp:1-php8.4

RUN install-php-extensions \
    pdo_pgsql \
    gd \
    # Str::studly() etc. need mb_split(); not guaranteed in the base image.
    mbstring \
    # queue:work only installs its SIGTERM handler when pcntl is loaded. Without it
    # PHP runs as PID 1 with no handler, the kernel drops SIGTERM, and
    # `docker-compose stop admin-worker` waits the full stop_grace_period (660s).
    pcntl

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

