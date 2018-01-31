Подключаем <a href="https://www.torproject.org/docs/debian.html.en">репозиторий torproject</a>.

```
# nano /etc/apt/sources.list
deb http://deb.torproject.org/torproject.org jessie main
deb-src http://deb.torproject.org/torproject.org jessie main
```

Добавляем ключи.
```
gpg --keyserver keys.gnupg.net --recv A3C4F0F979CAA22CDBA8F512EE8CBC9E886DDD89
gpg --export A3C4F0F979CAA22CDBA8F512EE8CBC9E886DDD89 | apt-key add -
```

Устанавливаем tor.
```
apt update
apt install tor deb.torproject.org-keyring
```

Создаем HiddenService.
```
# nano /etc/tor/torrc
HiddenServiceDir /var/lib/tor/hidden_service/
HiddenServicePort 80 127.0.0.1:8081

# /etc/init.d/tor restart
```

Tor сгенерирует приватный ключ и домен (hostname).
Сделайте резервную копию ключа, храните в надеждном месте.
```
# ls /var/lib/tor/hidden_service/
hostname  private_key
```
С помощью <a href="https://github.com/lachesis/scallion">scallion</a> - вы можете сгенерировать красивый домен.<br/>
Используя две карты GTX 1070, за 90 минут - `anilibriaq22mxcb.onion`

<hr/>

Настроим виртуальный хост, кэширование страниц, запретим POST запросы.
```
# mkdir /var/cache/nginx
# chown www-data:www-data /var/cache/nginx
```

```
# nano /etc/nginx/nginx.conf
http {
	...
	# Cache
	fastcgi_cache_path /var/cache/nginx/data levels=1:2 keys_zone=onion:10m;
```
```
# nano /etc/nginx/conf.d/onion.conf
server {
	listen 		 127.0.0.1:8081;
	server_name   anilibriaq22mxcb.onion;
	
	access_log /var/www/anilibria/logs/nginx_onion_access.log;
	error_log  /var/www/anilibria/logs/nginx_onion_error.log;
	index index.php index.html index.htm;
	root /var/www/anilibria/root/;
	index  index.php index.html index.htm;
	
	# Disable POST
	if ($request_method ~ "POST"){
		return 403;
	}

	rewrite /release/(.+).html$ /tracker/?ELEMENT_CODE=$1 last; # чпу для релизов

	location ~ (/\.ht|/bitrix/modules|/upload/support/not_image|/bitrix/php_interface){ # запрещаем доступ к include скриптам движка
		deny all;
	}

	location /upload/ { # запрещаем выполнять PHP в директории
		location ~* ^.+\.php {
			deny all;
		}
	}

	location ~* ^.+\.(jpg|jpeg|gif|png|svg|js|css|ico)$ {
		expires 7d;
		access_log off;
	}

	location ~* ^.+\.(mp3|ogg|mpe?g|avi|zip|gz|bz2?|rar|torrent|json)$ { # не логируем обращение к статике.
		access_log off;
	}

	location / {
		try_files $uri $uri/ @bitrix;
	}

	location ~ \.php$ { # Обрабатываем PHP скрипты
		try_files $uri $uri/ =404;
		include fastcgi_params;
		fastcgi_param  SCRIPT_FILENAME    $document_root$fastcgi_script_name;
		fastcgi_param  PATH_TRANSLATED	  $document_root$fastcgi_script_name;
		fastcgi_pass unix:/var/run/anilibria.sock;
		fastcgi_connect_timeout 60;
		fastcgi_send_timeout 60;
		fastcgi_read_timeout 60;
		fastcgi_buffer_size  128k;
		fastcgi_buffers  4 256k;
		fastcgi_busy_buffers_size  256k;
		
		# NGINX CACHE
		fastcgi_ignore_headers "Cache-Control" "Expires" "Set-Cookie";
		fastcgi_cache_key "$host|$request_uri";
		fastcgi_cache onion;
		fastcgi_cache_use_stale error timeout invalid_header http_500;
		fastcgi_cache_valid 404 502 30s;
		fastcgi_cache_valid any 5m;
		fastcgi_cache_min_uses 3;
		add_header X-Cache-Status $upstream_cache_status; # show cache status
	}

	location @bitrix { # Для ЧПУ bitrix, запросы идут на urlrewrite.php
		include fastcgi_params;
		fastcgi_param  SCRIPT_FILENAME    $document_root/bitrix/urlrewrite.php;
		fastcgi_param  PATH_TRANSLATED	  $document_root/bitrix/urlrewrite.php;
		fastcgi_pass unix:/var/run/anilibria.sock;
		fastcgi_connect_timeout 60;
		fastcgi_send_timeout 60;
		fastcgi_read_timeout 60;
		fastcgi_buffer_size  128k;
		fastcgi_buffers  4 256k;
		fastcgi_busy_buffers_size  256k;
		
		# NGINX CACHE
		fastcgi_ignore_headers "Cache-Control" "Expires" "Set-Cookie";
		fastcgi_cache_key "$host|$request_uri";
		fastcgi_cache onion;
		fastcgi_cache_use_stale error timeout invalid_header http_500;
		fastcgi_cache_valid 404 502 30s;
		fastcgi_cache_valid any 5m;
		fastcgi_cache_min_uses 3;
		add_header X-Cache-Status $upstream_cache_status; # show cache status
	}
}
```

Перезагружаем nginx и tor.
<pre>/etc/init.d/nginx restart
/etc/init.d/tor restart</pre>

Проверяем.
<img src="https://img.poiuty.com/img/bd/4e7372341d9fe6705ba71dd2225965bd.png">
