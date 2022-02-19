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
wp config create --allow-root --dbname=exampledb --dbuser=exampleuser --dbpass=examplepass --dbhost=db --dbprefix=wp_ --dbcharset=utf8mb4 --dbcollate=utf8mb4_0900_as_cs --extra-php <<'EOS'
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

## Put parameters to AWS System Manager Parameter Store

```
aws ssm put-parameter --name "/my/sns/email" --type String --value ${EMAIL}
aws ssm put-parameter --name "/my/db/username" --type String --value ${DATABASE_USERNAME}
aws ssm put-parameter --name "/my/db/password" --type String --value ${DATABASE_PASSWORD}
aws ssm put-parameter --name "/my/cloudfront/key" --type String --value ${CLOUDFRONT_KEY}
```

## Create Repositories

```
aws ecr create-repository --repository-name wp-web
aws ecr create-repository --repository-name wp-app
```

## Create database if you need

```
mysql -u${user} -p -h ${host} --default-character-set=utf8mb4 wordpress
CREATE DATABASE wordpress DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_as_cs;
```

## Restore database if you need

```
mysqldump -uroot -p --single-transaction --quick --default-character-set=utf8mb4 --set-charset wordpress > dump.sql
sed -i -e 's/utf8mb4_ja_0900_as_cs_ks/utf8mb4_0900_as_cs/' dump.sql
mysql -u${user} -p -h ${host} --default-character-set=utf8mb4 wordpress < dump.sql
```

## Dump database

```
mysqldump -u${user} -p -h ${host} --single-transaction --quick --default-character-set=utf8mb4 --set-charset wordpress > dump.sql
```

## Setting up WordPress using CloudFormation

Create Stacks

```
cfn create -t cloudformation/alb.yaml -s my-alb
cfn create -t cloudformation/wp-ecs-cluster-ec2-asg.yaml
cfn create -t cloudformation/wp-ecs-service-ec2.yaml
cfn create -t cloudformation/wp-cloudfront-alb.yaml
cfn create -t cloudformation/wp-cloudfront-media.yaml
cfn create -t cloudformation/wp-cloudfront-root.yaml

CLUSTER=$(aws cloudformation describe-stack-resource --stack-name wp-ecs-cluster-ec2-asg --logical-resource-id Cluster --query StackResourceDetail.PhysicalResourceId --output text)
SERVICE=$(aws cloudformation describe-stack-resource --stack-name wp-ecs-service-ec2 --logical-resource-id Service --query StackResourceDetail.PhysicalResourceId --output text)
aws ecs update-service --cluster $CLUSTER --service $SERVICE --desired-count 1
aws ecs update-service --cluster $CLUSTER --service $SERVICE --desired-count 0

BUCKET=$(aws cloudformation describe-stack-resource --stack-name wp-cloudfront-root --logical-resource-id S3Bucket --query StackResourceDetail.PhysicalResourceId --output text)
RECORDSET=$(aws cloudformation describe-stack-resource --stack-name wp-cloudfront-alb --logical-resource-id Route53RecordSet --query StackResourceDetail.PhysicalResourceId --output text)
aws s3api put-object --bucket $BUCKET --key ads.txt --body ads.txt --content-type text/plain
aws s3api head-object --bucket $BUCKET --key ads.txt

aws s3api put-object --bucket $BUCKET --key index.html --body index.html --content-type text/html --website-redirect-location "https://$RECORDSET"
aws s3api head-object --bucket $BUCKET --key index.html
```

## Mounting EFS if needed

```
yum install -y amazon-efs-utils
mkdir -p /root/efs

ACCESS_POINT=$(aws cloudformation describe-stack-resource --stack-name my-efs --logical-resource-id EFSAccessPoint --query StackResourceDetail.PhysicalResourceId --output text)
FILE_SYSTEM=$(aws cloudformation describe-stack-resource --stack-name my-efs --logical-resource-id EFSFileSystem --query StackResourceDetail.PhysicalResourceId --output text)
echo "mount -t efs -o tls,accesspoint=$ACCESS_POINT $FILE_SYSTEM /root/efs"
umount /root/efs
```

## Update Wordpress

Run commands inside the container to update WordPress

```
su-exec nginx wp core version --extra
su-exec nginx wp core update --locale=ja
su-exec nginx wp core update-db
su-exec nginx wp cache flush
```
