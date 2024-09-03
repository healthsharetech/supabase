FROM dunglas/frankenphp

RUN install-php-extensions \
    pdo_pgsql

ARG USER=www-data

RUN \
	# Use "adduser -D ${USER}" for alpine based distros
	useradd -D ${USER}; \
	# Remove default capability
	setcap -r /usr/local/bin/frankenphp; \
	# Give write access to /data/caddy and /config/caddy
	chown -R ${USER}:${USER} /data/caddy && chown -R ${USER}:${USER} /config/caddy

# copy custom caddyfile
COPY Caddyfile /etc/caddy/Caddyfile

RUN cp $PHP_INI_DIR/php.ini-production $PHP_INI_DIR/php.ini

USER ${USER}

