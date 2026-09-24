FROM php:8.3-fpm

RUN apt-get update && apt-get install -y \
    git \
    unzip \
    zip \
    curl \
    wget \
    libzip-dev \
    libpng-dev \
    libjpeg62-turbo-dev \
    libfreetype6-dev \
    libxslt1-dev \
    libicu-dev \
    libxml2-dev \
    libonig-dev \
    libmagickwand-dev \
    libcurl4-openssl-dev \
    pkg-config \
    default-mysql-client \
    cron \
    sudo \
    vim \
    && rm -rf /var/lib/apt/lists/*

RUN docker-php-ext-configure gd --with-jpeg --with-freetype

RUN docker-php-ext-install -j$(nproc) \
    bcmath \
    ctype \
    curl \
    dom \
    gd \
    intl \
    mbstring \
    opcache \
    pdo_mysql \
    soap \
    sockets \
    xsl \
    zip \
    exif \
    pcntl

RUN pecl install redis imagick && \
    docker-php-ext-enable redis imagick

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

RUN pecl install xdebug && docker-php-ext-enable xdebug

WORKDIR /var/www/html

COPY php.ini /usr/local/etc/php/conf.d/magento.ini

EXPOSE 9000

CMD ["php-fpm"]
