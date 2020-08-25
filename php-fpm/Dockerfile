FROM php:7.4-fpm-alpine

# persistent dependencies
RUN apk add --no-cache \
# Alpine package for "imagemagick" contains ~120 .so files, see: https://github.com/docker-library/wordpress/pull/497
		imagemagick

# install the PHP extensions we need (https://make.wordpress.org/hosting/handbook/handbook/server-environment/#php-extensions)
RUN set -ex; \
	\
	apk add --no-cache --virtual .build-deps \
		$PHPIZE_DEPS \
		freetype-dev \
		imagemagick-dev \
		libjpeg-turbo-dev \
		libpng-dev \
		libzip-dev \
	; \
	\
	docker-php-ext-configure gd --with-freetype --with-jpeg; \
	docker-php-ext-install -j "$(nproc)" \
		bcmath \
		exif \
		gd \
		mysqli \
		zip \
	; \
	pecl install imagick-3.4.4; \
	docker-php-ext-enable imagick; \
	\
	runDeps="$( \
		scanelf --needed --nobanner --format '%n#p' --recursive /usr/local/lib/php/extensions \
			| tr ',' '\n' \
			| sort -u \
			| awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
	)"; \
	apk add --virtual .wordpress-phpexts-rundeps $runDeps; \
	apk del .build-deps

# set recommended PHP.ini settings
# see https://secure.php.net/manual/en/opcache.installation.php
RUN set -eux; \
	docker-php-ext-enable opcache; \
	{ \
		echo 'opcache.memory_consumption=128'; \
		echo 'opcache.interned_strings_buffer=8'; \
		echo 'opcache.max_accelerated_files=4000'; \
		echo 'opcache.revalidate_freq=86400'; \
		echo 'opcache.fast_shutdown=1'; \
	} > /usr/local/etc/php/conf.d/opcache-recommended.ini
# https://wordpress.org/support/article/editing-wp-config-php/#configure-error-logging
RUN { \
# https://www.php.net/manual/en/errorfunc.constants.php
# https://github.com/docker-library/wordpress/issues/420#issuecomment-517839670
		echo 'error_reporting = E_ERROR | E_WARNING | E_PARSE | E_CORE_ERROR | E_CORE_WARNING | E_COMPILE_ERROR | E_COMPILE_WARNING | E_RECOVERABLE_ERROR'; \
		echo 'display_errors = Off'; \
		echo 'display_startup_errors = Off'; \
		echo 'log_errors = On'; \
		echo 'error_log = /dev/stderr'; \
		echo 'log_errors_max_len = 1024'; \
		echo 'ignore_repeated_errors = On'; \
		echo 'ignore_repeated_source = Off'; \
		echo 'html_errors = Off'; \
	} > /usr/local/etc/php/conf.d/error-logging.ini

# custmize
RUN set -eux ; \
    addgroup -g 101 -S nginx; \
    adduser -S -D -H -u 101 -h /var/cache/nginx -s /sbin/nologin -G nginx -g nginx nginx; \
    cp /usr/local/etc/php/php.ini-production /usr/local/etc/php/php.ini; \
    sed -i -r -e 's/^;?date.timezone =.*$/date.timezone = "UTC"/g' /usr/local/etc/php/php.ini; \
    sed -i -r -e 's/^;?expose_php =.*$/expose_php = Off/g' /usr/local/etc/php/php.ini; \
    sed -i -r -e 's/^;?max_execution_time =.*$/max_execution_time = 120/g' /usr/local/etc/php/php.ini; \
    sed -i -r -e 's/^;?post_max_size =.*$/post_max_size = 16M/g' /usr/local/etc/php/php.ini; \
    sed -i -r -e 's/^;?upload_max_filesize =.*$/upload_max_filesize = 16M/g' /usr/local/etc/php/php.ini; \
    sed -i -r -e 's/^;?emergency_restart_threshold =.*$/emergency_restart_threshold = 10/g' /usr/local/etc/php-fpm.conf; \
    sed -i -r -e 's/^;?emergency_restart_interval =.*$/emergency_restart_interval = 1m/g' /usr/local/etc/php-fpm.conf; \
    sed -i -r -e 's/^;?process_control_timeout =.*$/process_control_timeout = 10/g' /usr/local/etc/php-fpm.conf; \
    sed -i -r -e 's/^;?user =.*$/user = nginx/g' /usr/local/etc/php-fpm.d/www.conf; \
    sed -i -r -e 's/^;?group =.*$/group = nginx/g' /usr/local/etc/php-fpm.d/www.conf; \
    sed -i -r -e 's/^;?pm.max_requests =.*$/pm.max_requests = 500/g' /usr/local/etc/php-fpm.d/www.conf; \
    sed -i -r -e 's/^;?pm.max_children =.*$/pm.max_children = 5/g' /usr/local/etc/php-fpm.d/www.conf; \
    sed -i -r -e 's/^;?pm.start_servers =.*.*$/pm.start_servers = 2/g' /usr/local/etc/php-fpm.d/www.conf; \
    sed -i -r -e 's/^;?pm.min_spare_servers =.*.*$/pm.min_spare_servers = 1/g' /usr/local/etc/php-fpm.d/www.conf; \
    sed -i -r -e 's/^;?pm.max_spare_servers =.*.*$/pm.max_spare_servers = 3/g' /usr/local/etc/php-fpm.d/www.conf; \
    sed -i -r -e 's/^;?request_terminate_timeout =.*.*$/request_terminate_timeout = 120/g' /usr/local/etc/php-fpm.d/www.conf; \
    curl -fsL -o /usr/local/bin/wp https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar; \
    chmod +x /usr/local/bin/wp

VOLUME /var/www/html

CMD ["php-fpm"]
