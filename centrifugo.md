```
apt-get install centrifugo
```
или https://packagecloud.io/FZambia/centrifugo

```
centrifugo genconfig
```

/etc/centrifugo/config.json
```
{
  "secret": "сгенерируется выше",
  "address": "127.0.0.1",
  "insecure": true,
  "admin": true,
  "web": true,
  "ssl": false,
  "presence": true,
  "join_leave": true,
  "admin_password": "g34ewfqwsa",
  "admin_secret": "46hwter"
}

```

supervisor

```
[program:ws]
command=centrifugo --config=/etc/centrifugo/config.json
directory=/home/centrifugo
user=centrifugo
autostart=true
autorestart=true
stderr_logfile=/var/log/supervisor/ws.err.log
stdout_logfile=/var/log/supervisor/ws.out.log
```

nginx

```
upstream centrifugo {
    #sticky;
    ip_hash;
    server 127.0.0.1:8000;
    keepalive 512;
}

server {
	listen 		 <IP>:80;
	server_name   <WS_DOMAIN>;

	root /var/www/libriaws;
	index  index.php index.html index.htm;
}

# https://github.com/centrifugal/documentation/blob/master/deploy/nginx.md
# https://github.com/centrifugal/documentation/blob/master/deploy/tuning.md
server {
	listen 		 <IP>:443 ssl http2;
	server_name   <WS_DOMAIN>;
	keepalive_timeout 10;
	ssl_certificate /etc/letsencrypt/live/<WS_DOMAIN>/fullchain.pem;
	ssl_certificate_key /etc/letsencrypt/live/<WS_DOMAIN>/privkey.pem;

	default_type application/octet-stream;

	sendfile on;
	tcp_nopush on;
	tcp_nodelay on;
	gzip on;
	gzip_min_length 1000;
	gzip_proxied any;

	# Only retry if there was a communication error, not a timeout
	# on the Tornado server (to avoid propagating "queries of death"
	# to all frontends)
	proxy_next_upstream error;

	proxy_set_header X-Real-IP $remote_addr;
	proxy_set_header X-Scheme $scheme;
	proxy_set_header Host $http_host;

	location /connection {
		proxy_pass http://centrifugo;
		proxy_buffering off;
		proxy_read_timeout 60s;
		proxy_http_version 1.1;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Scheme $scheme;
		proxy_set_header Host $http_host;
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection $connection_upgrade;
	}

	location /socket {
		proxy_pass http://centrifugo;
		proxy_buffering off;
		proxy_read_timeout 60s;
		proxy_http_version 1.1;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Scheme $scheme;
		proxy_set_header Host $http_host;
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection $connection_upgrade;
	}

	location / {
	#	return 403;
		proxy_pass http://centrifugo;
	}
}
```
