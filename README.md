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
