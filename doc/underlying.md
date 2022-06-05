## 命名空间

> 以linux系统作为开发环境，因为mac环境和linux环境调用的标准库不一样，所以在goland中需设置
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

### IPC Namespace

IPC Namespace 用来隔离System V IPC 和 POSIX message queues。每一个IPC Namespaces 都有自己的System V IPC 和 POSIX message queue， 也就是说
IPC Namespace 的作用是使划分到不通 IPC Namespace的进程组通信上隔离，无法通过消息队列、共享内存、信号量的方式通信。

查看宿主机的 ipc Message Queues

```text
root@VM-0-17-ubuntu:/data/build# ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
```

创建一个消息队列，并查看

```text
root@VM-0-17-ubuntu:/data/build# ipcmk -Q
Message queue id: 32768
root@VM-0-17-ubuntu:/data/build# ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
0x35d14328 32768      root       644        0            0           
```

进入到只进行了uts namespace的程序中，并查看消息队列

```text
root@VM-0-17-ubuntu:/data/build# ./uts 
# ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
0x35d14328 32768      root       644        0            0           
```

**得出结论**

在同一个 ipc namespace 下，消息队列是未进行隔离的

编写代码， 刚才的uts代码的基础上，加上ipc隔离

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
		Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWIPC,
	}
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Run(); err != nil {
		log.Fatal(err)
	}
}
```

启动程序，并查看结果

```text
root@VM-0-17-ubuntu:/data/build# ./ipc 
# ipcs -q

------ Message Queues --------
key        msqid      owner      perms      used-bytes   messages    
```

**得出结论**

在不同 ipc namespace 下，消息队列是隔离的

### PIO Namespace

PID Namespace 是用来隔离进程ID的。同一个进程在不同的PID Namespace里可以拥有不同的PID

进入到只进行了uts namespace的程序中，并查看当前sh的pid

```text
root@VM-0-17-ubuntu:/data/build# ./uts 
# echo $$
7381
```

在宿主机查询进程树

```text
root@VM-0-17-ubuntu:/data/build# pstree -pl | grep uts
           |            |-sshd(17140)---sshd(17230)---bash(17232)---sudo(3127)---su(3128)---bash(3129)---uts(7377)-+-sh(7381)
           |            |                                                                                          |-{uts}(7378)
           |            |                                                                                          |-{uts}(7379)
           |            |                                                                                          `-{uts}(7380)
```

**得出结论**

在没有进行 PID Namespace 的程序中查询到的pid，和宿主机查询进程树的pid是一致的

编写代码， 刚才的ipc代码的基础上，加上pid隔离

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
		Cloneflags: syscall.CLONE_NEWUTS | syscall.CLONE_NEWIPC | syscall.CLONE_NEWPID,
	}
	cmd.Stdin = os.Stdin
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	if err := cmd.Run(); err != nil {
		log.Fatal(err)
	}
}
```

运行pid namespace程序，并查询pid

```text
root@VM-0-17-ubuntu:/data/build# ./pid 
# echo $$
1
```

查询宿主机进程树

```text
ubuntu@VM-0-17-ubuntu:/data$ pstree -pl | grep pid
           |            |-sshd(17140)---sshd(17230)---bash(17232)---sudo(3127)---su(3128)---bash(3129)---pid(10192)-+-sh(10196)
           |            |                                                                                           |-{pid}(10193)
           |            |                                                                                           |-{pid}(10194)
           |            |                                                                                           `-{pid}(10195)
```

**得出结论**

在进行pid namespace隔离的程序中，pid为1，而从宿主机中查询sh的pid为10196, 所以说宿主机中10196的pid，映射到 做了pid namespace的程序中是 1，实现了pid的隔离。

