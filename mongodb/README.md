创建宿主机mongodb文件存储目录

```shell
mkdir -p /data/mongodata
```

运行mongodb
```shell
docker-compose up -d
```

进入容器
```shell
docker exec -it mongo /bin/bash
```

测试登录
```shell
root@d65a03944ebc:/# mongo
MongoDB shell version v4.2.19
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("1a000f65-b5cf-4f65-ac73-04f4250d26d4") }
MongoDB server version: 4.2.19
> show dbs
> use admin
switched to db admin
> db.auth('root', 'password')
1
> show dbs
admin   0.000GB
config  0.000GB
local   0.000GB
> 
```