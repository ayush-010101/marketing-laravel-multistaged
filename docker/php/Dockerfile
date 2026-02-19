# ============================================================
# STAGE 1 — BUILDER
# ============================================================
FROM php:8.3-fpm-alpine AS builder

RUN apk add --no-cache \
    zip \
    unzip \
    git \
    curl \
    oniguruma-dev \
    libxml2-dev \
    zlib-dev \
    libzip-dev

RUN docker-php-ext-install \
    pdo_mysql \
    mbstring \
    xml \
    bcmath \
    opcache \
    pcntl \
    zip

RUN apk add --no-cache $PHPIZE_DEPS \
    && pecl install redis \
    && docker-php-ext-enable redis \
    && apk del $PHPIZE_DEPS

COPY --from=composer:2.7 /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/html

COPY src/composer.json src/composer.lock ./

RUN composer install \
    --no-dev \
    --optimize-autoloader \
    --no-interaction \
    --no-scripts

COPY src/ .

RUN php artisan config:cache \
    && php artisan route:cache \
    && php artisan view:cache

# ============================================================
# STAGE 2 — PRODUCTION
# ============================================================
FROM php:8.3-fpm-alpine AS production

RUN apk add --no-cache \
    oniguruma \
    oniguruma-dev \
    libxml2 \
    libxml2-dev \
    zlib \
    zlib-dev \
    libzip \
    libzip-dev

RUN docker-php-ext-install \
    pdo_mysql \
    mbstring \
    xml \
    bcmath \
    opcache \
    pcntl \
    zip

RUN apk add --no-cache $PHPIZE_DEPS \
    && pecl install redis \
    && docker-php-ext-enable redis \
    && apk del $PHPIZE_DEPS

RUN addgroup -g 1000 laravel \
    && adduser -u 1000 -G laravel -s /bin/sh -D laravel

WORKDIR /var/www/html

COPY --from=builder --chown=laravel:laravel /var/www/html .

COPY docker/php/php.ini /usr/local/etc/php/conf.d/custom.ini

RUN mkdir -p storage/logs storage/framework/cache \
        storage/framework/sessions storage/framework/views \
        bootstrap/cache \
    && chown -R laravel:laravel storage bootstrap/cache \
    && chmod -R 775 storage bootstrap/cache

USER laravel

EXPOSE 9000

CMD ["php-fpm"]