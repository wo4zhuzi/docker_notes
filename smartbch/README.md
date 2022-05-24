直接运行

```shell
docker pull smartbch/smartbchd:v0.4.4-p1

docker run --name smartbch \
 -p 8545:8545 \
 -p 8546:8546 \
 --ulimit nofile=65535:65535 \
 -v /data/smartbchd:/root/.smartbchd \
 -d  smartbch/smartbchd:v0.4.4-p1  start --mainnet-genesis-height=698502
```

docker-compose 运行

```shell
docker-compose up -d
```