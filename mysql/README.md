### 直接运行
```shell
docker pull mysql:5.7.38

docker run --name my-mysql  -p 3306:3306  -v /data/mysql_data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=password -d mysql:5.7.38 --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci --bind-address=0.0.0.0

docker exec -it my-mysql /bin/bash
```