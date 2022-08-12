# docker_notes

Docker notes

### 官方文档

[https://docs.docker.com](https://docs.docker.com)

### 镜像命令

#### 查看镜像

```shell
docker images
```

以本机为例，返回结果

```text
REPOSITORY            TAG                 IMAGE ID            CREATED             SIZE
phpswoole/swoole      4.6.2-php7.3        fcb0c50d89a7        15 months ago       490MB
portainer/portainer   latest              62771b0b9b09        21 months ago       79.1MB
mongo                 latest              a0e2e64ac939        2 years ago         364MB
```

+ REPOSITORY: 镜像仓库
+ TAG: 镜像标签，标记本地镜像，将其归入某一仓库。
+ IMAGE ID: 镜像id
+ CREATED: 创建时间
+ SIZE: 镜像大小

#### 拉取镜像

```shell
docker pull [OPTIONS] NAME[:TAG|@DIGEST]
```

example:
以kafka为例

##### 搜索镜像

```shell
docker search kafka
```

返回结果

```text
NAME                                      DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
bitnami/kafka                             Apache Kafka is a distributed streaming plat…   441                                     [OK]
kafkamanager/kafka-manager                Docker image for Kafka manager                  146                                     
bitnami/kafka-exporter                                                                    6                                       
ibmcom/kafka                              Docker Image for IBM Cloud Private-CE (Commu…   5                                       
```

从返回结果可以看出没有官方镜像，官方镜像的 OFFICIAL为[OK]，没有官方镜像就使用点赞最多的bitnami/kafka为例

##### 拉取最新版本

```shell
docker pull bitnami/kafka
```

等同于

```shell
docker pull bitnami/kafka:latest
```

返回结果

```shell
REPOSITORY            TAG                 IMAGE ID            CREATED             SIZE
bitnami/kafka         latest              191c4920968e        13 hours ago        618MB
phpswoole/swoole      4.6.2-php7.3        fcb0c50d89a7        15 months ago       490MB
portainer/portainer   latest              62771b0b9b09        21 months ago       79.1MB
mongo                 latest              a0e2e64ac939        2 years ago         364MB
```

##### 拉取指定版本

```shell
docker pull bitnami/kafka:3.1.0
```

返回结果

```shell
bitnami/kafka         3.1.0               92335cbe7395        23 hours ago        618MB
phpswoole/swoole      4.6.2-php7.3        fcb0c50d89a7        15 months ago       490MB
portainer/portainer   latest              62771b0b9b09        21 months ago       79.1MB
mongo                 latest              a0e2e64ac939        2 years ago         364MB
```

#### 删除镜像

```shell
docker rmi [OPTIONS] IMAGE [IMAGE...]
```

```shell
docker rmi bitnami/kafka:latest 
```

或

```shell
docker rmi 191c4920968e
```

### 容器命令

```shell
docker run [OPTIONS] IMAGE[:TAG|@DIGEST] [COMMAND] [ARG...]
```

未拉去镜像，直接运行 以zookeeper为例:

```shell
docker run -d --name zookeeper-server \
    --network my-network \
    -e ALLOW_ANONYMOUS_LOGIN=yes \
    bitnami/zookeeper:latest
```

运行kafka:

```shell
docker run -d --name kafka-server \
    --network my-network \
    -e ALLOW_PLAINTEXT_LISTENER=yes \
    -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://192.168.50.23:9092 \
    -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092 \
    -e KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper-server:2181 \
    -p 9092:9092 \
    bitnami/kafka:3.1.0
```

> -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://192.168.50.23:9092 在docker中运行，这个参数必填docker宿主机的ip地址，
> 否则客户端连发送消息会报错 dial tcp: lookup 4ac9cc32e26d: no such host

测试kafka客户端

```shell
docker run -it --rm \
    --network my-network \
    -e KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper-server:2181 \
    bitnami/kafka:3.1.0 kafka-topics.sh --list  --bootstrap-server kafka-server:9092
```

创建一个topic

```shell
docker run -it --rm \
    --network my-network \
    -e KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper-server:2181 \
    bitnami/kafka:3.1.0 kafka-topics.sh --create --topic test-topic --replication-factor 1 --partitions 1  --bootstrap-server kafka-server:9092
```

通过docker ps 查询运行的容器，发现kafka已经成功运行

```text
CONTAINER ID        IMAGE                      COMMAND                  CREATED             STATUS              PORTS                                    NAMES
193d84fff515        bitnami/kafka:3.1.0        "/opt/bitnami/script…"   3 minutes ago       Up 3 minutes        0.0.0.0:9092->9092/tcp                   kafka-server
1d5b15bd49df        bitnami/zookeeper:latest   "/opt/bitnami/script…"   12 minutes ago      Up 12 minutes       2181/tcp, 2888/tcp, 3888/tcp, 8080/tcp   zookeeper-server
72b8b29ea103        mongo                      "docker-entrypoint.s…"   7 months ago        Up 6 hours          0.0.0.0:27018->27017/tcp                 mongo
e64f0cd77157        portainer/portainer        "/portainer"             17 months ago       Up 6 hours          0.0.0.0:9000->9000/tcp                   prtainer-test
```

### 网络

> 默认docker run 创建的容器使用的驱动是bridge，容器之间的网关是一样的，各个容器之前可以相互
> ping通，但是每次容器重启的时候容器的内网地址会变，所以当容器重启都需要重新配置ip，比较麻烦，
> 所以可以自定一个网络，那么在自定义的网络中可以通过 [容器名称:端口]进行通信。

#### 查看当前网络

```shell
docker network ls
```

返回结果

```text
NETWORK ID          NAME                DRIVER              SCOPE
3445a5b558e8        bridge              bridge              local
73d44af35e7e        host                host                local
4d5af0c1f294        none                null                local
```

#### 创建自定义网络

```shell
docker network create my-network --driver bridge
```

#### docker-compose 中的网络

1.没有自定义网络名 实际使用网络是：<当前路径名_default>，如果<当前路径名>太长，会截取前缀部分。

2.自定义后缀

```text
networks
   - lnmp
```

定义网络名为lnmp，那么最终生产的网络名为：<当前路径名_lnmp>。

#### 修改默认网段

查询当前网络

```shell
ifconfig
```

返回

```shell
br-45c33e022655: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.1  netmask 255.255.0.0  broadcast 172.17.255.255
        inet6 fe80::42:6ff:fea5:9c15  prefixlen 64  scopeid 0x20<link>
        ether 02:42:06:a5:9c:15  txqueuelen 0  (Ethernet)
        RX packets 53333  bytes 25616397 (25.6 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 36677  bytes 7122938 (7.1 MB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
        ...
```

关闭docker，找到br网段为 172.17 的网桥删除、同时删除默认的docker0

```shell
service docker stop 

sudo ip link set dev docker0 down
sudo ip link set dev br-45c33e022655 down

apt install bridge-utils

sudo brctl delbr docker0
sudo brctl delbr br-45c33e022655
```

修改配置文件

```shell
vim /etc/docker/daemon.json

{
  "bip": "192.161.20.1/24"
}
```

打开重启docker service docker start

重启后docker0的网段变了，但是发觉br-45c33e022655的桥网段又变回来了， 针对这种情况，可以先把占用指定网段的network删除，重新创建即可

```shell
docker network ls

docker network inspect [:网络名称]

docker network rm [:网络名称]

//解除绑定
docker network disconnect  [:网络名称]  [:容器名称]
```

### docker-compose

下载地址
[https://github.com/docker/compose/releases](https://github.com/docker/compose/releases)

下载

```shell
wget https://github.com/docker/compose/releases/download/v2.5.1/docker-compose-linux-x86_64
```

加入到环境变量

```shell
mv docker-compose-linux-x86_64 /usr/bin/docker-compose

chmod +x /usr/bin/docker-compose
```
