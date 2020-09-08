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

## Setting up WordPress in ECS EC2 mode

```
ecs-cli configure --cluster wp-ec2 --default-launch-type EC2 --config-name wp-ec2 --region ap-northeast-1
ecs-cli configure profile --profile-name training --access-key $AWS_ACCESS_KEY_ID --secret-key $AWS_SECRET_ACCESS_KEY
ecs-cli up --instance-role my-ec2-role-for-ecs --keypair my-key --size 1 --security-group sg-04d366d09698b0636,sg-02dca44b8bd6f0e88 --subnets subnet-02bc52743e4cd6571,subnet-0f1ab6ad48c03cf1a --vpc vpc-030cb9b38bbf5d7ba --instance-type t3.micro --tags Name=wp-ecs --cluster wp-ec2
ecs-cli scale --capability-iam --size 0 --cluster wp-ec2
ecs-cli scale --capability-iam --size 1 --cluster wp-ec2
ecs-cli down --cluster wp-ec2
ecs-cli push wordpress-ecs_web:1.18.0
ecs-cli push wordpress-ecs_app:7.4.0
ecs-cli compose --file ecs-compose.yml --project-name wp-ec2 --ecs-params ecs-params.yml --cluster-config wp-ec2 create
ecs-cli compose --file ecs-compose.yml --project-name wp-ec2 --ecs-params ecs-params.yml --cluster-config wp-ec2 service up
ecs-cli compose --file ecs-compose.yml --project-name wp-ec2 --ecs-params ecs-params.yml --cluster-config wp-ec2 service stop
ecs-cli compose --file ecs-compose.yml --project-name wp-ec2 --ecs-params ecs-params.yml --cluster-config wp-ec2 service down
ecs-cli compose --file ecs-compose.yml --project-name wp-ec2 --ecs-params ecs-params.yml --cluster-config wp-ec2 service up --vpc vpc-030cb9b38bbf5d7ba --target-group-arn arn:aws:elasticloadbalancing:ap-northeast-1:274682760725:targetgroup/wp-ecs-ec2/e60e39ae9a63e604 --container-name web --container-port 80
ecs-cli compose --file ecs-compose.yml --project-name wp-ec2 --cluster-config wp-ec2 service scale 0

ecs-cli compose --file ecs-compose.yml --project-name wp-ec2 --ecs-params ecs-params.yml create launch-type EC2
ecs-cli compose --file ecs-compose.yml --project-name wp-ec2 --ecs-params ecs-params.yml service --cluster wp-ecs-ec2-spot-ECSCluster-5iAeoHg7upMl down
ecs-cli compose --file ecs-compose.yml --project-name wp-ec2 --ecs-params ecs-params.yml service --cluster wp-ecs-ec2-spot-ECSCluster-5iAeoHg7upMl up --target-groups "targetGroupArn=arn:aws:elasticloadbalancing:ap-northeast-1:274682760725:targetgroup/wp-ec-ALBTa-FSSIMGZHVTNX/16243b19898250f9,containerName=web,containerPort=80"
ecs-cli compose --file ecs-compose.yml --project-name wp-ec2 --cluster-config wp-ec2 service --cluster wp-ecs-ec2-spot-ECSCluster-5iAeoHg7upMl scale 1
```
