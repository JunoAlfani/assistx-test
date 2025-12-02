# =========================
# 1. Node build
# =========================
FROM node:20-alpine AS node_builder

WORKDIR /app
COPY package*.json ./
COPY vite.config.js ./
RUN npm ci
COPY . .
RUN npm run build

# =========================
# 2. Composer build
# =========================
FROM --platform=linux/amd64 composer:2.6 AS composer_builder

WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install --no-dev --no-interaction --no-scripts --no-progress --prefer-dist
COPY . .
RUN composer dump-autoload --optimize

# =========================
# 3. PHP-FPM + NGINX base
# =========================
FROM php:8.2-fpm-alpine

# Install packages
RUN apk add --no-cache bash curl autoconf g++ make shadow \
    libpng-dev libjpeg-turbo-dev freetype-dev libzip-dev oniguruma-dev linux-headers \
    nginx supervisor

# Install PHP Extensions
RUN docker-php-ext-configure gd --with-freetype --with-jpeg \
 && docker-php-ext-install pdo pdo_mysql mbstring exif pcntl bcmath zip gd opcache

# Create user
RUN addgroup -g 1000 appuser && adduser -D -u 1000 -G appuser appuser

WORKDIR /var/www/html

# Copy PHP + Build assets
COPY --from=composer_builder /app /var/www/html
COPY --from=node_builder /app/public/build /var/www/html/public/build

# =========================
# FIX NGINX PERMISSIONS
# =========================
RUN mkdir -p /var/www/html/nginx_temp \
    && mkdir -p /var/www/html/nginx_logs \
    && chown -R appuser:appuser /var/www/html/nginx_temp \
    && chown -R appuser:appuser /var/www/html/nginx_logs

# Copy configs
COPY docker/nginx.conf /etc/nginx/nginx.conf
COPY docker/supervisord.conf /etc/supervisord.conf

# Fix permissions for Laravel storage
RUN chown -R appuser:appuser /var/www/html \
 && chmod -R 775 /var/www/html/storage /var/www/html/bootstrap/cache

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --retries=3 CMD curl -f http://localhost:8080/ || exit 1

USER appuser

CMD ["/usr/bin/supervisord", "-c", "/etc/supervisord.conf"]
