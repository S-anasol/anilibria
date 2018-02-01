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

1. Откройте <a href="https://github.com/poiuty/anilibria/issues">issues</a>, напишите IP адрес и конфиг вашего сервера.<br/> 
2. Создаем поддомен.<br/>
3. Настройте cache proxy, сгенерируйте secret cookie (proxy_cache_bypass), отправьте на почту lupin@anilibria.tv<br/>
4. Проверяем, если ок - добавляем ваш сервер.

<hr/>

Создаем поддомен.

```
xakep1 A 109.248.206.13 # YALOCO  [PROXY]
x      A 5.9.82.141     # HETZNER [MAIN]
```

Действия выполняются на Debian 9, устанавливаем пакеты.
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

Скачиваем конфиг <a href="https://github.com/poiuty/anilibria/blob/master/conf/nginx_caching_proxy.conf">/etc/nginx/nginx.conf</a>, перезагружаем nginx.
```
wget https://raw.githubusercontent.com/poiuty/anilibria/master/conf/nginx_caching_proxy.conf -O /etc/nginx/nginx.conf
/etc/init.d/nginx restart
```

Проверим что cache работает.
```
# после трех запросов => файл попадает в cache
# curl https://xakep1.anilibria.tv/videos/ts/5223/0001/fff31.ts -s -I | grep x-cache-status
x-cache-status: MISS

# HIT => кеш работает.
# curl https://xakep1.anilibria.tv/videos/ts/5223/0001/fff31.ts -s -I | grep x-cache-status
x-cache-status: HIT
```

За удаление файла из кеша - отвечает настройка proxy_cache_bypass в конфиге nginx.<br/>
Чтобы удалить - отправим запрос => `ded334209c901fe8c90c9ca08c8aa86c` secret cookie.
```
curl https://xakep1.anilibria.tv/videos/ts/4576/0001/fff1.ts -s -I -H "ded334209c901fe8c90c9ca08c8aa86c:true"
```

<hr/>

`proxy_cache_min_uses` задаёт число запросов, после которого ответ будет закэширован. <a href="https://stackoverflow.com/questions/26399776/proxy-cache-min-uses-time-window">Временное окно</a> зависит от настройки proxy_cache_path `keys_zone`, `max_size` и `inactive`. Вытесняются из кеша, если нет обращений > `inactive time` или когда размер кеша превышает максимальное значение (алгоритм LRU).

Позволяет значительно снизить нагрузку на диск.


```
proxy_cache_min_uses 1;
iostat -xk -t 10
Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
nvme0n1           0.00    99.20  351.80   96.90 36560.40  8809.20   202.23    40.22   89.64    8.12  385.62   0.46  20.64
nvme0n1           0.00    53.20  341.60   15.60 35042.40  8408.00   243.28     5.71   15.97    7.35  204.74   0.54  19.32
nvme0n1           0.00    42.90  318.80   17.50 32736.80 10322.40   256.08     6.68   19.88    9.21  214.17   0.67  22.64
nvme0n1           0.00    90.70  388.30  100.20 40371.20 11722.00   213.28    41.87   85.70   15.70  357.00   0.53  26.04
nvme0n1           0.00    51.90  297.40   15.60 30661.60  9576.00   257.11     7.03   22.48   10.67  247.59   0.66  20.64

AVG %util (20.64+19.32+22.64+26.04+20.64)/5 = 21.856
```
```
proxy_cache_min_uses 3;
iostat -xk -t 10
Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
nvme0n1           0.00    20.00  316.80    3.20 32999.20  2244.80   220.28     0.38    1.20    0.88   33.00   0.35  11.08
nvme0n1           0.00    47.30  340.80   52.40 35730.80  2792.40   195.95     1.42    3.61    0.90   21.22   0.31  12.28
nvme0n1           0.00    22.50  298.20    5.00 31284.40  3541.20   229.72     0.92    3.04    1.96   68.00   0.38  11.64
nvme0n1           0.00    25.40  337.60    3.70 35014.00  2290.00   218.60     0.42    1.24    0.88   33.84   0.29   9.96
nvme0n1           0.00    41.20  322.90   42.20 33559.60  3074.80   200.68     2.77    7.59    1.16   56.80   0.33  12.08

AVG %util (11.08+12.28+11.64+9.96+12.08)/5 = 11.408
```
<hr/>

Если `proxy_temp_path` и `proxy_cache_path` располагаться на разных файловых системах, то вместо дешёвой операции переименовывания в пределах одной файловой системы файл копируется с одной файловой системы на другую. Поэтому лучше, если кэш будет находиться на той же файловой системе, что и каталог с временными файлами.<br/>

```
# df -h
/dev/md0                 92G   15G   73G  17% /
/dev/nvme0n1p1          459G  406G   30G  94% /var/www/cache
```
```
http {
	...
	proxy_cache_path /var/www/cache levels=1:2 keys_zone=STATIC:125m inactive=1d max_size=430g;
	...
location ~*  \.(m3u8|ts)$ {
	...
	proxy_temp_path /var/www/cache/proxy_temp 1 2;
	...
```

<hr/>

Nginx: <a href="https://nginx.ru/ru/docs/">документация</a>.<br/>
<a href="https://www.nginx.com/resources/wiki/start/topics/examples/reverseproxycachingexample/">Reverse Proxy with Caching</a>.<br/>
<a href="https://www.nginx.com/blog/nginx-caching-guide/">A Guide to Caching with NGINX and NGINX Plus</a>.<br/>
<a href="https://habrahabr.ru/post/260669/">Пулы потоков</a>: ускоряем NGINX в 9 и более раз.<br/>
Увеличиваем производительность с помощью <a href="https://habrahabr.ru/post/259403/">SO_REUSEPORT в NGINX 1.9.1</a><br/>

