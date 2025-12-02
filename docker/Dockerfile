# ============================
# 1) Node builder (Vite)
# ============================
FROM node:20-alpine AS node_builder
WORKDIR /app

# Copy file yang diperlukan saja
COPY ../../package*.json ./
COPY ../../vite.config.js ./

RUN npm ci

# Copy seluruh source code
COPY ../../ .

RUN npm run build


# ============================
# 2) Composer builder
# ============================
FROM composer:2.6 AS composer_builder
WORKDIR /app

COPY ../../composer.json ../../composer.lock ./
RUN composer install --no-dev --no-interaction --no-scripts --no-progress --prefer-dist

COPY ../../ .
RUN composer dump-autoload --optimize


# ============================
# 3) PHP-FPM ONLY
# ============================
FROM php:8.2-fpm-alpine

RUN apk add --no-cache \
    bash curl \
    libpng-dev libjpeg-turbo-dev freetype-dev \
    libzip-dev oniguruma-dev linux-headers

RUN docker-php-ext-configure gd --with-freetype --with-jpeg && \
    docker-php-ext-install pdo_mysql mbstring exif pcntl bcmath zip gd opcache

WORKDIR /var/www/html

# Copy hasil builder
COPY --from=composer_builder /app /var/www/html
COPY --from=node_builder /app/public/build /var/www/html/public/build

RUN chmod -R 775 storage bootstrap/cache

EXPOSE 8080

CMD ["php", "-S", "0.0.0.0:8080", "-t", "public"]
