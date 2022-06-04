## 命名空间

> 以linux为环境进行开发，因为mac环境和linux环境调用的标准库不一样，所以在goland中需设置
> preferences => Build Tags & Vendoring 中的 os 为linux，否则ide加载不到正确的package

### UTS Namespace

UTS(UNIX Time Sharing) namespace 是最简单的一种命名空间。UTS中主要包含了主机名（hostname）、域名（domain name）和一些版本信息：

代码

```go
package main

import (
	"log"
	"os"
	"os/exec"
	"syscall"
)

func main() {
	cmd := exec.Command("sh")
	cmd.SysProcAttr = &syscall.SysProcAttr{
		Cloneflags: syscall.CLONE_NEWUTS,
	}
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Run(); err != nil {
		log.Fatal(err)
	}
}

```

首先，在宿主机的命令行输出如下命令，查看当前bash的pid

```shell
echo $$
```

得到结果

```text
24223
```

再执行

```shell
netstat -ntlp
```

得到结果

```text
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      1186/sshd           
tcp        0      0 0.0.0.0:8888            0.0.0.0:*               LISTEN      5602/docker-proxy   
tcp        0      0 127.0.0.1:1514          0.0.0.0:*               LISTEN      4742/docker-proxy   
tcp        0      0 0.0.0.0:111             0.0.0.0:*               LISTEN      795/rpcbind         
tcp        0      0 0.0.0.0:8080            0.0.0.0:*               LISTEN      1726/docker-proxy   
tcp        0      0 0.0.0.0:50000           0.0.0.0:*               LISTEN      1705/docker-proxy   
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      1009/systemd-resolv 
tcp6       0      0 :::8888                 :::*                    LISTEN      5608/docker-proxy   
tcp6       0      0 :::111                  :::*                    LISTEN      795/rpcbind         
tcp6       0      0 :::8080                 :::*                    LISTEN      1730/docker-proxy   
tcp6       0      0 :::50000                :::*                    LISTEN      1712/docker-proxy   
tcp6       0      0 :::80                   :::*                    LISTEN      1325/apache2       
```

拿当前bash的pid和sshd的pid进行对比

```shell
readlink /proc/24223/ns/uts \
readlink /proc/1186/ns/uts 
```

得到结果如下

```text
uts:[4026531838]
uts:[4026531838]
```

再执行命令

```shell
ps -ef | grep jenkins
```

得到结果

```text
1000      1775  1748  0 May27 ?        00:00:13 /sbin/tini -- /usr/local/bin/jenkins.sh
1000      1835  1775  0 May27 ?        00:21:25 java -Duser.home=/var/jenkins_home -Djenkins.model.Jenkins.slaveAgentPort=50000 -Dhudson.lifecycle=hudson.lifecycle.ExitLifecycle -jar /usr/share/jenkins/jenkins.war
1000      1886  1835  0 May27 ?        00:00:00 [jenkins.sh] <defunct>
root     30195 13641  0 23:31 pts/4    00:00:00 grep --color=auto jenkins

```

> jenkins 是运行在 docker中的

拿bash的pid和docker中运行的jenkins pid进行对比

```shell
readlink /proc/24223/ns/uts \
readlink /proc/1775/ns/uts
```

得到结果

```text
uts:[4026531838]
uts:[4026532259]
```

运行golang写的uts代码，并查询bash pid

```shell
./uts
echo $$
```

得到结果

```text
6320
```

拿bash的pid和docker中运行的jenkins pid、golang系统调用的pid进行对比

```shell
readlink /proc/24223/ns/uts \
readlink /proc/1775/ns/uts \
readlink /proc/6320/ns/uts 
```

得到结果

```text
uts:[4026531838]
uts:[4026532259]
uts:[4026532955]
```

**得出结论:**

bash 和 sshd在同一个uts namespace下，而bash和docker中运行的程序、golang 系统调用的uts程序，不在一个uts namespace下。

查看宿主机的hostname

```shell
root@VM-0-17-ubuntu:~# hostname
VM-0-17-ubuntu
```

修改golang中运行的sh的hostname并查询

```shell
# hostname
VM-0-17-ubuntu
# hostname -b test
# hostname
test
```

查询宿主机中的hostname

```shell
root@VM-0-17-ubuntu:~# hostname
VM-0-17-ubuntu
```

**得出结论:**

通过系统调用 syscall.CLONE_NEWUTS，使得golang中运行的sh和宿主机的hostname是相互隔离的。