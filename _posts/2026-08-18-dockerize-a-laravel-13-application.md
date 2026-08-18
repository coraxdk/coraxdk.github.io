---
layout: post
title: "Dockerize a Laravel 13 application"
categories: laravel
tags: laravel docker hosting development tutorial
image:
  path: https://cdn.coraxnet.dk/bCdOni-rAolzuJYcp2eLFymd1YOktjhtzl_ChR6Burs/rs:fill/bG9jYWw6Ly8vZG9j/a2VyLWxhcmF2ZWwt/YmFubmVyLnBuZw
---
So I was working on a Laravel application and got the idea of hosting it as a docker container on my server instead of uploading it to a hosting provider as usual. It took me a fair bit of time digging though Googles search results to find out the best method of doing so which none of them lead me to a final usable result, but they did give me hints here and there which helped me building my image in the end - but it took longer than I thought it would to be honest.<br>
There are many tutorials on how to dockerize Laravel, but they are aither written for older versions of Laravel, or they were not as optimized as I wanted it to be.
Trust me, you will want to optimize it. When I made my first Dockerfile for this, I just build everthing in the same image, which resultet in a whopping 2.1 Gb image, while in a multi-stage like this, we have everything we need in just around 800 Mb - so a huge difference, not to mention the amount f vulnerabilities we have limited.

## Skip Wayfinder
The first thing we need to do is to prevent wayfinder to run during NPM build. To do this, change the <i>vite.config.ts</i> in the root of your project, so it looks like this:

```js
import inertia from '@inertiajs/vite';
import { wayfinder } from '@laravel/vite-plugin-wayfinder';
import react from '@vitejs/plugin-react';
import laravel from 'laravel-vite-plugin';
import { bunny } from 'laravel-vite-plugin/fonts';
import { defineConfig } from 'vite';

export default defineConfig({
    plugins: [
        laravel({
            input: ['resources/css/app.css', 'resources/js/app.tsx'],
            refresh: true,
            fonts: [
                bunny('Instrument Sans', {
                    weights: [400, 500, 600],
                }),
            ],
        }),
        inertia(),
        react({
            babel: {
                plugins: ['babel-plugin-react-compiler'],
            },
        }),
        process.env.SKIP_WAYFINDER !== 'true' ?
            wayfinder({
                formVariants: true,
            }) :
            null,
    ],
});
```
If you dont do this, then it will trigger an php artisan command when building the JavaScript files, which means we would need to install PHP on the node:image, and we dont want that.

## .dockerignore
Next lets add a few lines to .dockerignore file in the root of the project.
I have chosen to leave out the entire storage folder, the build and cache folders. If you are not building your image from your local development environment this wont do much difference, but since I am also building the docker image locally as well, I dont want any clutter in these folder.
```
.git
.gitignore
.github
node_modules
vendor
storage/*
public/build
bootstrap/cache
```


## Dockerfile
Now that the project is prepared, let's build us an image.
There ara millions of ways of assembling this. A common method is to use php:fpm as image and then stack it together with nginx for serving the site, but I have chosen to use php:apache, so I dont need that extra container and configuration afterwards.

```Dockerfile
# ============================================
# COMPSER LIBRARIES BUILD
# ============================================
FROM composer:latest AS libraries
WORKDIR /app
COPY . .

RUN mkdir -p /app/bootstrap/cache /app/storage/framework/views
RUN composer install --no-dev --optimize-autoloader
RUN php artisan wayfinder:generate --with-form

# ============================================
# NPM LIBRARIES BUILD
# ============================================
FROM node:latest AS frontend
WORKDIR /app
COPY . .
COPY --from=libraries /app/storage /app/storage
ENV SKIP_WAYFINDER=true
RUN npm ci
RUN npm run build

# ============================================
# ASSEMBLING THE IMAGE
# ============================================
FROM php:apache

# Enabling rewrite apache mod
RUN a2enmod rewrite

# Enable php mod pdo_mysql (can be omitted if your are not using MySQL for database. Sqlite is already included in the image)
RUN docker-php-ext-install pdo_mysql

# Changing the document root of Apache as it will have to point to the public folder of our Laravel application
ENV APACHE_DOCUMENT_ROOT=/var/www/html/public
RUN sed -ri -e 's!/var/www/html!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/sites-available/*.conf
RUN sed -ri -e 's!/var/www/!${APACHE_DOCUMENT_ROOT}!g' /etc/apache2/apache2.conf /etc/apache2/conf-available/*.conf

# Adding the project files to the image
WORKDIR /var/www/html
COPY . .
COPY --from=libraries /app/vendor /var/www/html/vendor
COPY --from=libraries /app/bootstrap/cache /var/www/html/bootstrap/cache
COPY --from=libraries /app/storage /var/www/html/storage
COPY --from=frontend /app/public/build /var/www/html/public/build

# We left out the storage dir, so lets rebuild it and set permissions
RUN mkdir -p /var/www/html/storage/private /var/www/html/storage/public /var/www/html/storage/framework/cache /var/www/html/storage/framework/sessions
RUN chown -R www-data:www-data /var/www/html/storage

# Executing the entrypoint script and starts apache
ENTRYPOINT ["docker/entrypoint.sh"]
CMD ["apachectl", "-D", "FOREGROUND"]
```

## Entrypoint
The last step for building our image is to create a new folder (if its not already there) called <i>docker</i>.<br>
In this folder we will create a new file called <i>entrypoint.sh</i>
Everything in this file will be executed every time the container starts.

```bash
#!/bin/bash
php artisan optimize
php artisan migrate --force

if [ -f "/var/www/html/database/database.sqlite" ]; then
  chown www-data:www-data /var/www/html/database/database.sqlite
fi

exec docker-php-entrypoint "$@"
```
Its a pretty simple bash script running some artisan commands, so I think it is pretty much self explainatory.<br>
Thats it... we are now ready for building our image!

## Building the image
You can build the image just by running
```console
docker build . -t <imagename>
```
from a console while standing in the root of your Laravel project (thats where the Dockerfile is).<br>
A better way, but also more complicated is to use Github actions to build it and store it online on Github, but thats for another time.

## Running the image
All that is left now is to spin it up and see your awesome application run from within a Docker container.
```console
docker run -D -p 80:80 <imagename>
```

## Configuration
Notice that in this case we have'nt set any ENV configuration, so it will fall back to the default database, using sqlite, but you can just add environment vars to override Laravel default configuration values. If you have anything specific to configure, like for example changing the database to MySQL, then you can just pass that as env when running the container.

## Example docker-compose using MariaDB
Here is an example on how to run your application directly from the local project folder in a docker-compose file.
Look at the environment section - it is exaclty like you would have written it inside an .env instead
```yaml
services: 
  web:
    build:
      context: .
      dockerfile: Dockerfile
    restart: unless-stopped
    ports:
      - 80:80/tcp
    depends_on:
      - database
    environment: 
      APP_ENV: production
      APP_KEY: <your app key>
      DB_CONNECTION: mysql
      DB_HOST: database
      DB_DATABASE: myapp
      DB_USERNAME: mysql
      DB_PASSWORD: mysql
  database:
    image: mariadb:latest
    restart: unless-stopped
    environment: 
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: myapp
      MYSQL_USER: mysql
      MYSQL_PASSWORD: mysql
    volumes:
      - database:/var/lib/mysql

volumes:
  database:
```
## Conclusion
So after some hours testing back and forth, I think I found a pretty OK solution for running Laravel in a container. The image is kept small and secure, and it is hosted out of its own container.<br>
I hope you could use this toturial, if not just some of it.<br>
