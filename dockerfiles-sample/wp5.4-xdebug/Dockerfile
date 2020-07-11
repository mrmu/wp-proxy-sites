FROM wordpress:5.4.2-php7.4

# install the PHP extensions we need
# RUN set -ex; \
# 	\
# 	apt-get update; \
# 	apt-get install -y --no-install-recommends \
# 		libxslt-dev \
# 	; \
# 	\
# 	docker-php-ext-install -j "$(nproc)" \
# 		xsl \
# 	; \
# 	ls

# Xdebug
ENV XDEBUG_PORT 9000

RUN yes | pecl install xdebug \
    && echo "zend_extension=$(find /usr/local/lib/php/extensions/ -name xdebug.so)" > /usr/local/etc/php/conf.d/xdebug.ini \
    && echo "xdebug.remote_enable=1" >> /usr/local/etc/php/conf.d/xdebug.ini \
    && echo "xdebug.remote_autostart=1" >> /usr/local/etc/php/conf.d/xdebug.ini \
    && echo "xdebug.profiler_enable=1" >> /usr/local/etc/php/conf.d/xdebug.ini \
    && echo "xdebug.profiler_output_name=cachegrind.out.%t" >> /usr/local/etc/php/conf.d/xdebug.ini \
    && echo "xdebug.profiler_output_dir=/tmp" >> /usr/local/etc/php/conf.d/xdebug.ini \
    && echo "max_input_vars=2000" >> /usr/local/etc/php/conf.d/custom.ini \
    && rm -rf /usr/local/etc/php/conf.d/opcache-recommended.ini

EXPOSE 9000