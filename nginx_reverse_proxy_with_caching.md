В 2016 году Sanasol предложил <a href="https://goo.gl/DGxujN">кешировать видео с помощью cloudflare</a> (бесплатный тариф).<br/>
<a href="https://img.poiuty.com/img/41/6fc3f0c7447fce2f96ebe8300af4c341.png">Статистика cloudflare</a> за последний месяц (август 2017).
```
Total Bandwidth 205.71 TB
Cached Bandwidth 170.13 TB
Uncached Bandwidth 35.58 TB
```

Халява закончилась в октябре. Cloudflare <a href="https://img.poiuty.com/img/75/bf96a5525bba1fbbc32e53117c580e75.png">отключил cache</a>.<br/>
Есть свободные мощности и желание помочь проекту anilibria.tv?<br/>
Запустите на своем сервере cache proxy. Минимальные требования.<br/>
```
RAM: 16GB
Disk space: 500GB
Network: 250Mbit/s
```

<hr/>

Создаем поддомен.

```
xakep1 A 109.248.206.13 # YALOCO  [PROXY]
x      A 5.9.82.141     # HETZNER [MAIN]
```

Устанавливаем пакеты.
```
apt-get update && apt-get upgrade -y
apt-get install nginx munin munin-node certbot spawn-fcgi libcgi-fast-perl htop bwm-ng strace lsof libfile-readbackwards-perl
```

Устанавливаем munin плагин <a href="https://github.com/munin-monitoring/contrib/blob/master/plugins/nginx/nginx-cache-hit-rate">nginx-cache-hit-rate</a>
```
# wget -O /usr/share/munin/plugins/nginx-cache-hit-rate https://raw.githubusercontent.com/munin-monitoring/contrib/master/plugins/nginx/nginx-cache-hit-rate
# chmod 755 /usr/share/munin/plugins/nginx-cache-hit-rate
# ln -s /usr/share/munin/plugins/nginx-cache-hit-rate /etc/munin/plugins/

# nano /etc/munin/plugin-conf.d/munin-node
...
[nginx-cache-hit-rate]
user www-data

# /etc/init.d/munin-node restart
```

Генерируем dhparam. Получаем letsencrypt сертификат.
```
mkdir /etc/nginx/ssl/
openssl dhparam -out /etc/nginx/ssl/dhparam.pem 2048
chown www-data:www-data /etc/nginx/ssl/dhparam.pem
chmod 400 /etc/nginx/ssl/dhparam.pem
certbot certonly --webroot -w /var/www/html -d xakep1.anilibria.tv -m admin@anilibria.tv --agree-tos
```

Настраиваем автопродление сертификата, добавляем renew-hook, перезагружаем cron.
```
# nano /etc/cron.d/certbot
...
0 */12 * * * root test -x /usr/bin/certbot -a \! -d /run/systemd/system && perl -e 'sleep int(rand(3600))' && certbot -q renew --renew-hook "/etc/init.d/nginx restart"

# /etc/init.d/cron restart
```
