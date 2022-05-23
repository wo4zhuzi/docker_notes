运行
```shell
docker-compose up -d
```

进入php容器
```shell
docker exec -it php73 /bin/bash
```

安装mysql扩展
```shell
docker-php-ext-install pdo pdo_mysql mysqli  #安装并启动扩展（常用）
docker-php-ext-enable pdo pdo_mysql mysqli   #启动PHP扩展
```

退出容器
```shell
exit
```

重启服务
```shell
docker-compose restart
```