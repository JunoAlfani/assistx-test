# node build
FROM node:20-alpine AS node_builder

WORKDIR /app

COPY package*.json ./

COPY vite.config.js ./

RUN npm ci

COPY resources resources

RUN npm run build

# composer install
FROM --platform=linux/amd64 composer:2.6 AS composer_builder

WORKDIR /app

COPY composer.json composer.lock ./

RUN composer install --no-dev --no-interaction --no-scripts --no-progress --prefer-dist

COPY . .

RUN composer dump-autoload --optimize

# adding image with php-fpm + nginx on alpine
FROM php:8.2-fpm-alpine AS php_base

RUN apk add --no-cache nginx supervisor curl bash

# install extensions (pdo, pdo_mysql, mbstring, xml, tokenizer, json)
RUN docker-php-ext-install pdo pdo_mysql mbstring exif pcntl bcmath opcache

RUN addgroup -g 1000 appuser && adduser -D -u 1000 -G appuser appuser

WORKDIR /var/www/html

COPY --from=composer_builder /app /var/www/html

COPY --from=node_builder /app/resources/js/dist /var/www/html/public/build

# nginx config and supervisor
COPY docker/nginx.conf /etc/nginx/nginx.conf

COPY docker/supervisord.conf /etc/supervisord.conf

RUN chown -R appuser:appuser /var/www/html/storage /var/www/html/bootstrap/cache

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --retries=3 CMD curl -f http://localhost:8080/ || exit 1

USER appuser

CMD ["/usr/bin/supervisord", "-c", "/etc/supervisord.conf"]





