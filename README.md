# docker-laravel-nginx


## Configurações adicionais
- criar volume referenciando arquivo: `<path-config>/host.conf:/etc/nginx/conf.d/default.conf:ro`

## Exemplo arquivo `<path-config>/host.conf`

````
server {
    listen 80;

    #listen 442 ssl;

    #ssl_certificate /etc/letsencrypt/live/<dominio>/fullchain.pem;
    #ssl_certificate_key /etc/letsencrypt/live/<dominio>/privkey.pem;

    index index.php index.html;
    root /var/www/public;


    client_max_body_size 200M;

    location / {
        try_files $uri /index.php?$args;
    }

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass php:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_read_timeout 900;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }

    location /socket.io {
	    proxy_pass http://echo:6001; #could be localhost if Echo and NginX are on the same box
	    proxy_http_version 1.1;
	    proxy_set_header Upgrade $http_upgrade;
	    proxy_set_header Connection "Upgrade";
	}

}
````

## Configuração exemplo do nginx no host que faz o proxy
````
server {
    listen 443 ssl;
    root /var/www/app1/laravelapp/public;
    server_name <dominio>;
    index index.php index.html;

    ssl_certificate /etc/letsencrypt/live/<dominio>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<dominio>/privkey.pem;

    location / {
	proxy_pass https://localhost:8000; # definir para porta do container
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    	proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /socket.io {
        proxy_pass https://localhost:8000/socket.io; # definir para porta do container
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
    }
}
````

## Observações
- Certificados devem ser configurados nos dois nginx, do container e do proxy
- No nginx proxy apontar a porta para a porta que foi definada ao criar o container
- `http://echo` para grupos de containers criado com docker-compose echo é um aliás para o host servidor de socket

## Exemplo completo do docker-compose.yml para o projeto laravel
Lista de serviços para o projeto laravel:
- nginx
- php
- queue (fila do laravel)
- echo (servidor de websocket)
- jasperserver

````
version: '3'
services:
    nginx:
        image: juareze2ti/laravel-nginx
        container_name: <sistema>_<cidade>_nginx
        restart: always
        volumes:
            - ./:/var/www
            - /etc/localtime:/etc/localtime:ro
            - ./docker/nginx/host.conf:/etc/nginx/conf.d/default.conf:ro
            - /etc/letsencrypt/archive/<dominio>:/etc/letsencrypt/archive/<dominio>
            - /etc/letsencrypt/live/<dominio>:/etc/letsencrypt/live/<dominio>
        ports:
            - "<porta_nginx>:443"
        depends_on:
            - php
            - redis
            - echo

    php:
        image: juareze2ti/laravel-php
        container_name: <sistema>_<cidade>_php
        restart: always
        mac_address: 00:e0:91:5f:34:ae
        volumes:
            - ./:/var/www
            - /tmp:/tmp
            - /etc/localtime:/etc/localtime:ro
            - ./docker/php/config.ini:/usr/local/etc/php/conf.d/config.ini:ro
            - ./storage/licencas/sintetizador-de-voz.txt:/usr/vt/helena/P16/data-common/verify/verification.txt:ro
        depends_on:
            - redis

    queue:
        image: juareze2ti/laravel-queue
        container_name: <sistema>_<cidade>_queue
        restart: always
        environment:
           - QUEUE=sms
           - SLEEP=1
           - TRIES=24
           - TIMEOUT=120
           - DELAY=600
           - MEMORY=512
        volumes:
            - ./:/var/www
            - /tmp:/tmp
            - /etc/localtime:/etc/localtime:ro
            - ./docker/php/config.ini:/usr/local/etc/php/conf.d/config.ini:ro
        depends_on:
            - redis

    echo:
        image: juareze2ti/laravel-echo
        container_name: <sistema>_<cidade>_echo
        restart: always
        volumes:
            - ./docker/echo/config.json:/app/laravel-echo-server.json:ro
        depends_on:
            - redis

    redis:
        image: redis
        container_name: <sistema>_<cidade>_redis
        restart: always

    jasperserver:
        image: juareze2ti/jasperserver-ce
        container_name: <sistema>_<cidade>_jasperserver
        restart: always
        ports:
            - "8080:8080"
        environment:
            - DB_HOST=postgres
            - DB_USER=postgres
            - DB_PASSWORD=postgres
            - DB_PORT=5432
            - DB_NAME=jasperserver
            - POSTGRES_USER=postgres
            - POSTGRES_PASSWORD=postgres
            - TZ=America/Manaus
        volumes:
            - ./storage/jasper_keystore:/usr/local/share/jasperserver/keystore
            - /etc/localtime:/etc/localtime:ro
````
