FROM php:8.3-fpm-alpine

# Combined into one layer to keep image size tiny
RUN apk add --no-cache \
    freetype \
    libjpeg-turbo \
    libpng \
    libzip \
    oniguruma \
    imagemagick \
    # Build dependencies (deleted later)
    freetype-dev \
    libjpeg-turbo-dev \
    libpng-dev \
    libzip-dev \
    oniguruma-dev \
    imagemagick-dev \
    $PHPIZE_DEPS \
    && docker-php-ext-install mysqli pdo pdo_mysql opcache mbstring zip \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install gd \
    && pecl install redis imagick \
    && docker-php-ext-enable redis imagick \
    # Remove build deps and temporary files immediately
    && apk del $PHPIZE_DEPS *-dev \
    && rm -rf /tmp/* /usr/share/php7 /usr/share/man /var/cache/apk/*
