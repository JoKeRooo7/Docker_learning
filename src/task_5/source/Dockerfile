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
