# wordpress-ecs

## Setting up WordPress on your machine

Enter the container

```
docker-compose run --rm app ash
```

Download WordPress in the container

```
wp core download --allow-root --locale=ja
apk add mysql-client
wp config create --allow-root --dbname=exampledb --dbuser=exampleuser --dbpass=examplepass --dbhost=db --dbprefix=wp_ --dbcharset=utf8mb4 --dbcollate=utf8mb4_ja_0900_as_cs_ks --extra-php <<'EOS'
define('DISABLE_WP_CRON', true);
define('WP_POST_REVISIONS', false);
if (isset($_SERVER['HTTP_CLOUDFRONT_FORWARDED_PROTO']) && $_SERVER['HTTP_CLOUDFRONT_FORWARDED_PROTO'] == 'https') {
    $_SERVER['HTTPS'] = 'on';
} elseif (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] == 'https') {
    $_SERVER['HTTPS'] = 'on';
}
EOS
rm wp-config-sample.php
rm readme.html
```

Run containers

```
docker-compose up -d
```

Configure Wordpress

http://localhost:8080
