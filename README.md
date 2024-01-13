# Working with docker on windows

* [Download, restart and basic commands](#download-restart-and-basic-commands)
* [Configuring nginx.conf](#configuring-nginxconf)
* [Serven to fastcgi in c](#serven-to-fastcgi-in-c)
* [Writing a Dockerfile to execute all positions above](#writing-a-dockerfile-to-execute-all-positions-aboveserven-to-fastcgi-in-c)
* [Using Dockle](#using-dockle)
* [Creating Docker-compose.yml](#creating-docker-composeyml)

Due to security reasons, some of the screenshots of the work performed have been severely curtailed and deleted


## Download, restart and basic commands

> downloading the image of the `docker pull nginx` container

> checking for the presence of the `docker images ls` image + container size
![docker images](task_1/image/docker_image_ls.png)

> running the `docker run -d nginx` image

> checking the launch of the `docker ps` image

> part of the information about the container `docker inspect [container_id|container_name]`

> stopping the container image `docker stop [container_id|container_name]` and `docker ps -a`

> running the image on ports 80 and 443 `docker run -d -p 80:80 -p 443:443 nginx`
> if you enter it in the browser http://localhost:80 get rid of the official nginx page

> restart and check `docker restart [container_id|container_name]`


## Configuring nginx.conf

> contents of the original nginx file `docker exec [container_id|container_name] cat /etc/nginx/nginx.conf`

> the contents of the new nginx.conf file are located in the folder ./task_2/source
![new nginx](task_2/image/nginx_conf_source.png)

> co-opting my file to the container image `docker cp nginx.conf [container_id|container_name]:/etc/nginx/nginx.conf`

> restart via `exec` `docker exec [container_id|container_name] nginx -s reload`

> container export `docker export [container_id|container_name] -o container.tar`

> stopping the image `docker stop [container_id|container_name]`

> deleting the `docker rmi -f [image_id]` image

> deleting a stopped container `docker rm -f [container_id|container_name]`

> import `dcoker import ./container.tar`

> add the tag `docker tag [image_id] my_import_image:nginx` to the local file

> running the image on ports 80 and 443 `docker run -d -p 80:80 -p 443:443 nginx`
> check the status on the page http://localhost:80/status


## Serven to fastcgi in c

> I wrote a simple server with the response `hello world` in c, for further launching it on port 81
> the server is located in the folder ./task_3/source/main.c

> wrote nginx configuration to listen on port 81 using FastCGI to process requests in docker (port 8080)
> the configuration is located in ./task_3/source/nginx.conf

> since I'm working inside the image, I decided to download all the packages and the server itself into the image.
> updating the `apt-get` program:
* `docker exec [container_id|container_name] apt-get update`
> downloading packages:
* `docker exec [container_id|container_name] apt-get install -y libfcgi-dev`
* `docker exec [container_id|container_name] apt-get install spawn-fcgi`
> download the compiler:
* `docker exec [container_id|container_name] apt-get install -y install gcc`

> file co-op in docker
* `docker cp main.c [container_id|container_name]:task_3/source/main.c`
* `docker cp nginx.conf [container_id|container_name]:/etc/nginx/nginx.conf`

> restarting nginx `docker exec [container_id|container_name] nginx -s reload`

> compilation of the server file into the used server file `server`
* `docker exec [container_id|container_name] gcc -o server main.c -lfcgi`

> starting the server `docker exec spawn-fcgi -p 8080 ./server`

> checking the server:
![port_81](task_3/image/port_81.png)


## Writing a Dockerfile to execute all positions [above](#serven-to-fastcgi-in-c)

Dockerfile (task_4/source/Dockerfile):

``` Dockerfile
FROM nginx

RUN apt-get update && apt-get install -y \
libfcgi-dev \
spawn-fcgi \
gcc

COPY main.c .
COPY nginx.conf /etc/nginx/nginx.conf

RUN gcc -o server main.c -lfcgi
EXPOSE 8080
CMD spawn-fcgi -p 8080 server && nginx -g 'daemon off;'
```

> build the image using the `build` command. To build, you need to be in the same directory
> `docker build -t name/part:tag`

> start with mapping port 81 to port 80 `docker run -d -p 80:81 [image_id]`
> verification http://localhost:80

> fix nginx.conf
![new nginx](task_4/image/tpshtch_sht.png)

> restarting the container
![status new](task_4/image/corrected %20nginx.png)


## Using Dockle

Since I work on windows, I used a dockle image, so running it is a bit specific.

> download dockle `docker pull goodwithtech/dockle`

> launched it according to the official repository `docker run --rm -v /var/run/docker.sock:/var/run/docker.sock goodwithtech/dockle [image_id]`

> received error reports
* FATAL CIS-DI-0010
* FATAL DKL-DI-0005
* WARN CIS-DI-0001
* WARN DKL-DI-0006
* other INFO

> To fix serious errors, I had to rewrite the Dockerfile (task_5/source/Dockerfile)
* added a user to it
* folders inside nginx
* permissions for nginx
* deleted apt-get stories

``` Dockerfile
FROM nginx:latest
RUN apt-get update && apt-get install -y \
libfcgi-dev \
spawn-fcgi \
gcc \
&& rm -rf /var/lib/apt/lists/*
RUN groupadd -r appgroup && useradd -r -g appgroup new_user
USER new_user
COPY main.c /tmp/
COPY nginx.conf /etc/nginx/nginx.conf
RUN gcc -o /tmp/server /tmp/main.c -lfcgi
USER root
RUN mkdir -p /var/cache/nginx/client_temp \
&& chown -R new_user:appgroup /var/cache/nginx/client_temp \
&& chmod -R 777 /var/cache/nginx/client_temp
EXPOSE 8080
CMD spawn-fcgi -p 8080 /tmp/server && nginx -g 'daemon off;'
```

> to accept suspicious (FATAL CIS-DI-0010), because I do not use these environment variables (this is inside nginx), I used the flag `--ak NGINX_GPGKEY_PATH --ak NGINX_GPGKE`

> running `docker run --rm -v /var/run/docker.sock:/var/run/docker.sock goodwithtech/dockle --ak NGINX_GPGKEY_PATH --ak NGINX_GPGKEY [image_id|image_name:tag]`


## Creating Docker-compose.yml

> New docker-compose files.yml, nginx.conf, add to ./task_6/source/

In docker-compose.yml I use two containers, one that is created from the dockerfile folder ./task_5/source/, the first container uses its nginx.conf from ./task_5/source/.

My main nginx container runs on port 80 and transmits all changes.
Its nginx.conf is listening on port 8080. Port 80 proxies all requests to port 81.

In order to create a web server, I use the commands `docker-compose build` and to run `docker-compose run`
![port80](task_6/image/port80.png)