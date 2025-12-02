# node build
FROM node:20-alpine AS node_builder
WORKDIR /app
COPY package*.json ./
COPY vite.config.js ./
RUN npm ci
COPY . .
RUN npm run build

# composer install
FROM --platform=linux/amd64 composer:2.6 AS composer_builder
WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install --no-dev --no-interaction --no-scripts --no-progress --prefer-dist
COPY . .
RUN composer dump-autoload --optimize

# base image php + nginx
FROM php:8.2-fpm-alpine AS php_base

RUN apk add --no-cache bash curl autoconf g++ make shadow \
    libpng-dev libjpeg-turbo-dev freetype-dev libzip-dev oniguruma-dev linux-headers \
    nginx supervisor

RUN docker-php-ext-configure gd --with-freetype --with-jpeg && \
    docker-php-ext-install pdo pdo_mysql mbstring exif pcntl bcmath zip gd opcache

# user for PHP-FPM only (NOT for nginx)
RUN addgroup -g 1000 appuser && adduser -D -u 1000 -G appuser appuser

WORKDIR /var/www/html
COPY --from=composer_builder /app /var/www/html
COPY --from=node_builder /app/public/build /var/www/html/public/build

COPY docker/nginx.conf /etc/nginx/nginx.conf
COPY docker/supervisord.conf /etc/supervisord.conf

# fix nginx permissions
RUN mkdir -p /var/lib/nginx/tmp \
    && mkdir -p /var/log/nginx \
    && chmod -R 755 /var/lib/nginx /var/log/nginx \
    && chown -R root:root /var/lib/nginx /var/log/nginx

# laravel storage permissions
RUN chown -R appuser:appuser /var/www/html \
    && chmod -R 775 /var/www/html/storage /var/www/html/bootstrap/cache

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --retries=3 CMD wget -qO- http://localhost:8080/ || exit 1

# IMPORTANT: nginx MUST run as root
USER root

CMD ["/usr/bin/supervisord", "-c", "/etc/supervisord.conf"]
