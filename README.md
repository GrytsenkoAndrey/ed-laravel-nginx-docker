# ed-laravel-nginx-docker
https://medium.com/@faidfadjri/how-to-setup-laravel-nginx-using-docker-2023-ba9de4b60d04

First let’s talk about docker, what Docker actually is.

according to the (Docker official sites.)[https://docs.docker.com/get-started/overview/]

Docker is a FOSS ( Free Open Source Software ) which enable us to separated our application using container.

In some cases you may handle a few project which run on different technologies on a different version. For example you may use nodejs to your project and some project may need nodejs 17 and another one need node js 18 and if you just install nodejs 17 then the nodejs 18 application will broken and say error like “deprecated packages” or so on.

Then we need to using the docker. it’s a life saver tho :)
cause we don’t need to install any version of it. just configure it.

Before jump to the next part i’ll need to introduce you some of basic knownledge about docker & nginx :

Images : is a executable package that included everything need to run in application. for example like windows you’ve been used, php language, node js and so on.

Container : Instance of images. in simple term when you need to modify something about the image you need to create a container based on image. For example you may need to use ubuntu OS for your application and you may need to install some package or change folder permission. then you need to create a container. Container also needed to wrap your whole project and it called a Docker Container.

NGINX : is a web server which can serve a webpages to the client. Nginx also can act as reverse proxy so he manage all the request then forwarding to the spesific port using little configuration script.

Then i borrow some illustration below.

![image](https://github.com/GrytsenkoAndrey/ed-laravel-nginx-docker/assets/63291871/66086126-83de-4d54-ab64-64d460decd4f)


According to sample illustration Nginx will receive the request and forward it into spesific application.

So , let’s jump to the tutorial.

First you need to install some package :
1. Docker : Docker Download
2. Composer : Composer
3. Docker Compose : How to install Docker Compose

Then, create a laravel project using this command

> composer create-project laravel/laravel laravel-nginx-docker

Open folder using visual studio code or your current text editor.

create a new file call Dockerfile on a root of folder
and add this script :

```
# use PHP 8.2
FROM php:8.2-fpm

# Install common php extension dependencies
RUN apt-get update && apt-get install -y \
    libfreetype-dev \
    libjpeg62-turbo-dev \
    libpng-dev \
    zlib1g-dev \
    libzip-dev \
    unzip \
    && docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install -j$(nproc) gd \
    && docker-php-ext-install zip

# Set the working directory
COPY . /var/www/app
WORKDIR /var/www/app

RUN chown -R www-data:www-data /var/www/app \
    && chmod -R 775 /var/www/app/storage


# install composer
COPY --from=composer:2.6.5 /usr/bin/composer /usr/local/bin/composer

# copy composer.json to workdir & install dependencies
COPY composer.json ./
RUN composer install

# Set the default command to run php-fpm
CMD ["php-fpm"]
```

We are using php:8.2-fpm images as a base images then install some php-extension package.

```
COPY . /var/www/app
WORKDIR /var/www/app
```

That mean we will copy . ( whole application file & directory )
to /var/www/app (which is our container path).

```
RUN chown -R www-data:www-data /var/www/app \
    && chmod -R 775 /var/www/app/storage
```

We’ll change the folder owner to www-data which is the default nginx user and change the folder permission which can be read and write.

```
# install composer
COPY --from=composer:2.6.5 /usr/bin/composer /usr/local/bin/composer

# copy composer.json to workdir & install dependencies
COPY composer.json ./
RUN composer install
```

Then we install composer package and run “composer install” which is the command to install all needed dependencies

Then we can create file inside a folder nginx/default.conf and add this script

```
server {
    listen 80;
    server_name localhost;
    root /var/www/app/public;

    index index.php index.html index.htm;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_pass php:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }

    error_log  /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;
}
```

We will tell the nginx to listen to the port 80 which is the default nginx port and set the root folder on /var/www/app/public which is our public laravel directory inside of our container.

Last but not least we can create docker-compose.yaml inside of root of our application, and add this script :

```
version: "3"

networks:
  laravel:
    driver: bridge

services:
  nginx:
    image: nginx:alpine
    container_name: nginx
    restart: unless-stopped
    tty: true
    ports:
      - "7000:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
      - .:/var/www/app:delegated
    depends_on:
      - php
    networks:
      - laravel

  php:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: php
    restart: unless-stopped
    tty: true
    expose:
      - "9000"
    volumes:
      - .:/var/www/app:delegated
    networks:
      - laravel
```

We’ll create network call laravel to be a bridge for each container can be communicate.

Then create a nginx services and copy the nginx.conf and set the volume. if you notice the volume on nginx and laravel are same it’s mean they can share data on to another.

Then you can just :

```
docker-compose up -d
```

You’ll see the container are running and you can access through your browser just hit :

```
http://localhost:7000
```

![image](https://github.com/GrytsenkoAndrey/ed-laravel-nginx-docker/assets/63291871/37ffba6d-933d-4d38-96f5-269a0eaa6b8e)






