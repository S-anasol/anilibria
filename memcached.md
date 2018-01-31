Конфиг <a href="https://github.com/poiuty/anilibria/blob/master/conf/memcached.conf">memcached.conf</a>

```
apt-get install memcached php5-memcache
wget https://raw.githubusercontent.com/poiuty/anilibria/master/conf/memcached.conf -O /etc/memcached.conf
/etc/init.d/memcached restart
```

<hr>

Bitrix. <a href="https://dev.1c-bitrix.ru/learning/course/index.php?COURSE_ID=32&LESSON_ID=5505">Включаем memcached</a>.
```
# nano /var/www/anilibria/root/bitrix/php_interface/dbconn.php
define("BX_CACHE_TYPE", "memcache");
define("BX_CACHE_SID", $_SERVER["DOCUMENT_ROOT"]."#01");
define("BX_MEMCACHE_HOST", "unix:///tmp/memcached.sock");
define("BX_MEMCACHE_PORT", "0");
```
```
# nano /var/www/anilibria/root/bitrix/.settings_extra.php
<?php
return array(
  'cache' => array(
    'value' => array(
      'type' => 'memcache',
      'memcache' => array(
        'host' => 'unix:///tmp/memcached.sock',
        'port' => '0',
      ),
      'sid' => $_SERVER["DOCUMENT_ROOT"]."#01"
    ),
  ),
);
?>
```
```
chown anilibria:www-data /var/www/anilibria/root/bitrix/php_interface/dbconn.php
chown anilibria:www-data /var/www/anilibria/root/bitrix/.settings_extra.php
```
<hr>

Сохраняем <a href="http://php.net/manual/ru/memcached.sessions.php">php сессии в memcached</a>.
```
# nano /etc/php5/fpm/pool.d/anilibria.conf
php_admin_value[session.save_handler] = memcache
php_admin_value[session.save_path] = "unix:///tmp/memcached.socket:0"
```
