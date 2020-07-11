FROM wordpress:5.3.2-php7.4

# install the PHP extensions we need
RUN set -ex; \
	\
	apt-get update; \
	apt-get install -y --no-install-recommends \
		libxslt-dev \
	; \
	\
	docker-php-ext-install -j "$(nproc)" \
		xsl \
	; \
	ls
