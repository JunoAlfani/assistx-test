# -------------------------------------------------------------
# STAGE 1 — Node build (vite)
# -------------------------------------------------------------
FROM node:20-alpine AS node_builder

WORKDIR /app
COPY package*.json ./
COPY vite.config.js ./
RUN npm ci

COPY . .
RUN npm run build

# -------------------------------------------------------------
# STAGE 2 — Composer dependencies
# -------------------------------------------------------------
FROM --platform=linux/amd64 composer:2.6 AS composer_builder

WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install --no-dev --no-interaction --no-scripts --no-progress --prefer-dist

COPY . .
RUN composer dump-autoload --optimize


# =======================
# 3. PHP + NGINX RUNTIME
# =======================
FROM php:8.2-fpm-alpine

# install dependencies
RUN apk add --no-cache \
    bash curl nginx supervisor wget autoconf g++ make shadow \
    libpng-dev libjpeg-turbo-dev freetype-dev libzip-dev oniguruma-dev linux-headers

# php extensions
RUN docker-php-ext-configure gd --with-freetype --with-jpeg && \
    docker-php-ext-install pdo pdo_mysql mbstring exif pcntl bcmath zip gd opcache

# --- IMPORTANT FIX ---
# nginx MUST run as root user
# php-fpm can run as www-data automatically

# prepare folders for nginx (permissions fix)
RUN mkdir -p /var/lib/nginx/tmp \
    && mkdir -p /var/log/nginx \
    && mkdir -p /run/nginx \
    && chmod -R 777 /var/lib/nginx \
    && chmod -R 777 /var/log/nginx \
    && chmod -R 777 /run/nginx

# project files
WORKDIR /var/www/html

COPY --from=composer_builder /app /var/www/html
COPY --from=node_builder /app/public/build /var/www/html/public/build

# nginx + supervisor configs
COPY docker/nginx.conf /etc/nginx/nginx.conf
COPY docker/supervisord.conf /etc/supervisord.conf

# fix permissions for storage
RUN chmod -R 775 storage bootstrap/cache || true

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -qO- http://localhost:8080/ || exit 1

# DO NOT DROP PRIVILEGE! nginx needs root.
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisord.conf"]
