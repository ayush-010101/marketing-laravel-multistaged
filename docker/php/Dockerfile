# ============================================================
# STAGE 1 — BUILDER
# This stage installs all dependencies and compiles assets.
# Think of this as the "kitchen" where we prepare everything.
# It will NOT exist in the final image.
# ============================================================
FROM php:8.3-fpm-alpine AS builder

# Why Alpine? It's a tiny Linux distro — only 5MB base size.
# The regular PHP image is 400MB+. Alpine version is ~80MB.

# --------------------------------
# Install system dependencies
# These are Linux packages needed to compile PHP extensions
# --------------------------------
RUN apk add --no-cache \
    # zip/unzip — needed by Composer to handle packages
    zip \
    unzip \
    # git — Composer needs this to pull some packages from GitHub
    git \
    # curl — for downloading things
    curl \
    # oniguruma — needed for PHP's mbstring extension (handles multi-byte text like Arabic, Chinese)
    oniguruma-dev \
    # libxml2 — needed for PHP's xml extension
    libxml2-dev \
    # libpng, libjpeg, freetype — needed if your app handles images
    libpng-dev \
    libjpeg-turbo-dev \
    freetype-dev \
    # nodejs and npm — needed to compile frontend assets (CSS, JS)
    nodejs \
    npm

# --------------------------------
# Install PHP extensions
# Laravel requires specific PHP extensions to work
# --------------------------------
RUN docker-php-ext-configure gd \
        --with-freetype \
        --with-jpeg \
    && docker-php-ext-install \
        # pdo_mysql — lets PHP talk to MySQL database
        pdo_mysql \
        # mbstring — handles multi-byte string operations
        mbstring \
        # xml — XML parsing
        xml \
        # bcmath — precise math calculations (important for financial/marketing data)
        bcmath \
        # gd — image processing
        gd \
        # opcache — caches compiled PHP code in memory, makes app faster
        opcache \
        # pcntl — process control, needed for Laravel queue workers
        pcntl

# --------------------------------
# Install the Redis PHP extension
# This one isn't built into PHP so we use PECL (PHP's extension installer)
# --------------------------------
RUN apk add --no-cache $PHPIZE_DEPS \
    && pecl install redis \
    && docker-php-ext-enable redis

# --------------------------------
# Install Composer
# Composer is PHP's package manager (like npm for Node)
# We copy it from its official Docker image — clean and always up to date
# --------------------------------
COPY --from=composer:2.7 /usr/bin/composer /usr/bin/composer

# --------------------------------
# Set working directory
# This is where all our app code will live inside the container
# --------------------------------
WORKDIR /var/www/html

# --------------------------------
# Copy composer files FIRST (before copying all app code)
# Why? Docker caches layers. If we copy composer.json first and
# run composer install, Docker caches that layer. Next time you
# build, if composer.json hasn't changed, it skips this slow step.
# This makes rebuilds much faster.
# --------------------------------
COPY src/composer.json src/composer.lock ./

# --------------------------------
# Install PHP dependencies
# --no-dev = skip development-only packages (testing tools etc)
# --optimize-autoloader = generates a faster class map for production
# --no-interaction = don't ask any questions during install
# --------------------------------
RUN composer install \
    --no-dev \
    --optimize-autoloader \
    --no-interaction \
    --no-scripts

# --------------------------------
# Copy package.json for Node dependencies (frontend assets)
# Same caching trick as composer — copy manifest first, install, then copy rest
# --------------------------------
COPY src/package*.json ./

# Install Node dependencies
RUN npm ci --ignore-scripts

# --------------------------------
# Now copy ALL the application source code
# We do this AFTER installing dependencies so Docker's cache works efficiently
# --------------------------------
COPY src/ .

# --------------------------------
# Compile frontend assets (CSS, JS) for production
# This runs Laravel Mix or Vite — whatever your package.json defines
# --------------------------------
RUN npm run build

# --------------------------------
# Run Laravel optimizations
# These commands cache routes, config, and views for faster performance
# --------------------------------
RUN php artisan config:cache \
    && php artisan route:cache \
    && php artisan view:cache

# ============================================================
# STAGE 2 — PRODUCTION
# This is the FINAL image. It starts completely fresh.
# We only copy the finished, compiled app from Stage 1.
# No build tools, no Node, no Composer, no npm — just the app.
# ============================================================
FROM php:8.3-fpm-alpine AS production

# --------------------------------
# Install ONLY runtime dependencies
# (not build tools — just what PHP needs to actually RUN)
# --------------------------------
RUN apk add --no-cache \
    libpng \
    libpng-dev \
    libjpeg-turbo \
    libjpeg-turbo-dev \
    freetype \
    freetype-dev \
    oniguruma \
    oniguruma-dev \
    libxml2 \
    libxml2-dev \
    zlib \
    zlib-dev \
    libzip \
    libzip-dev

# --------------------------------
# Install PHP extensions in production image too
# (same extensions, but no dev tools needed to build them)
# --------------------------------
RUN docker-php-ext-configure gd \
        --with-freetype \
        --with-jpeg \
    && docker-php-ext-install \
        pdo_mysql \
        mbstring \
        xml \
        bcmath \
        gd \
        opcache \
        pcntl

# Install Redis extension
RUN apk add --no-cache $PHPIZE_DEPS \
    && pecl install redis \
    && docker-php-ext-enable redis \
    && apk del $PHPIZE_DEPS

# --------------------------------
# Create a non-root user
# By default Docker runs as root — dangerous.
# If an attacker exploits your app they'd have root access.
# We create a normal user called "laravel" with limited permissions.
# --------------------------------
RUN addgroup -g 1000 laravel \
    && adduser -u 1000 -G laravel -s /bin/sh -D laravel

# Set working directory
WORKDIR /var/www/html

# --------------------------------
# Copy the finished application from Stage 1 (the builder)
# --from=builder means "take this from the first stage, not the host machine"
# We copy only what's needed to run — not build tools
# --------------------------------
COPY --from=builder --chown=laravel:laravel /var/www/html .

# --------------------------------
# Copy our custom PHP settings
# --------------------------------
COPY docker/php/php.ini /usr/local/etc/php/conf.d/custom.ini

# --------------------------------
# Set correct permissions on storage and cache folders
# Laravel needs to write logs and cache files to these folders
# --------------------------------
RUN mkdir -p storage/logs storage/framework/cache storage/framework/sessions \
        storage/framework/views bootstrap/cache \
    && chown -R laravel:laravel storage bootstrap/cache \
    && chmod -R 775 storage bootstrap/cache

# Switch to the non-root user
# Everything from this line onwards runs as "laravel", not "root"
USER laravel

# --------------------------------
# Expose port 9000
# PHP-FPM listens on port 9000. Nginx will forward requests here.
# --------------------------------
EXPOSE 9000

# --------------------------------
# Start PHP-FPM
# This is the process that actually runs your Laravel application
# --------------------------------
CMD ["php-fpm"]