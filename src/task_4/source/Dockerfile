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
