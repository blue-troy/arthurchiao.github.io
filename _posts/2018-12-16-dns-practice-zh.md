---
layout: post
title:  "DNS问题分析示例"
date:   2018-12-16
author: ArthurChiao
categories: dns nameserver
---

## 1 DNS基础知识

互联网基于TCP/IP协议。为了方便管理网络内的主机，整个互联网分为若干个[域
](https://en.wikipedia.org/wiki/Domain_name)（domain），每
个域又可以再分为若干个子域，例如，`.com`，`.org`，`.edu`都是顶级域，而
`google.com`是`.com`下面的子域。

网络中的任意一台主机（host）都会属于某个域，并且有自己的名字，称为主机名（
hostname）。例如`example.com`就是`.com`域中一台主机名为`example.com`（或
`example`，hostname和domain name的区别，见[这里
](https://en.wikipedia.org/wiki/Domain_name)）的主机。

域名/主机名是为了方便人记忆，而机器之间通信最终用的还是IP地址，因此需要一个将主
机名（域名）转换成IP地址的服务。域名服务系统（DNS, domain name system）做的就是
这个事情，对应的服务器称为域名服务器（Domain Name Server）。

例如，当通过浏览器访问[example.com](example.com)，浏览器会首先访问DNS服务器，查找
`example.com`对应的IP地址，然后和这个IP建立TCP连接，接下来才发起HTTP请求。

一个域名可以对应一个IP地址，也可以对应多个。对于后者，DNS服务算法会从中选择一个
地址返回。大部分网络服务为了实现高可用，都是对应多个地址，我们后面会看到，
baidu.com就对应多个IP。

有一些场景会导致访问DNS服务不稳定，例如DNS服务器的设置有问题、网络有丢包、主机
DNS配置错误等等。我们接下来查看几种case。

## 2 准备测试环境

为方便大家跟着上手练习，本文将搭建一个容器环境。

Pull Docker镜像:

```shell
$ sudo docker pull alpine:3.8
```

运行容器，注意这里一定要带`--privileged`参数 [2]，否则后面的部分`tc`命令无法执行:

```shell
$ sudo docker run -d --privileged --name ctn-1 alpine:3.8 sleep 3600d
$ sudo docker ps
CONTAINER ID    IMAGE        COMMAND         CREATED        STATUS          PORTS  NAMES
233bc36bde4b    alpine:3.8   "sleep 3600d"   1 minutes ago  Up 14 minutes           ctn-1
```

进入容器：

```shell
$ sudo docker exec -it ctn-1 sh
```

查看容器网络信息：

```shell
/ # ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:09
          inet addr:172.17.0.9  Bcast:0.0.0.0  Mask:255.255.0.0
```

## 3 DNS配置

### 3.1 查看DNS配置

Linux上的DNS配置在`/etc/resolv.conf`里面。我们先来查看容器的配置：

```shell
/ # cat /etc/resolv.conf
# Dynamic resolv.conf(5) file for glibc resolver(3) generated by resolvconf(8)
#     DO NOT EDIT THIS FILE BY HAND -- YOUR CHANGES WILL BE OVERWRITTEN
nameserver 192.168.1.11
nameserver 192.168.1.12
```

这其实是继承了宿主机的DNS配置，在宿主机上执行`cat /etc/resolv.conf`会看到一样的
结果。

### 3.2 修改DNS配置

可以通过修改`/etc/resolv.conf`里面的`nameserver`
来配置自己想用的DNS服务器。例如内网环境可能都会使用自己的DNS服务器，因为它除了
提供内网域名解析之外，公网域名解析也会比较快（相比于网络供应商的公网DNS服务器）
。

## 4 DNS问题排查

本节模拟几种导致DNS查询变慢的场景，如果在实际环境中遇到类似现象，可以考虑往这些
方向排查。

### 4.1 机器未配置DNS导致域名查找失败

* **现象**：网络是通的（例如ping IP通），但是DNS查询总是失败
* **可能的原因**：机器没有配置DNS服务器
* **解决办法**：修改`/etc/resolv.conf`，给机器配置合适的DNS服务器

有时新启动的机器（不管是物理机、虚拟机还是容器）没有设置DNS，导致访问域名不通。
我们来复现一下。

在正常的容器里用`nslookup`工具查看域名对应的IP地址：

```shell
/ # nslookup example.com

Name:      example.com
Address 1: 93.184.216.34
Address 2: 2606:2800:220:1:248:1893:25c8:1946
```

可以看到，我们获取到了该域名一个IPv4地址和一个IPv6地址。

将`/etc/resolv.conf`里的DNS服务器列表用`#`注释掉，模拟没有配置DNS服务器的场景。

再次测试：

```shell
/ # nslookup example.com

nslookup: can't resolve 'example.com': Try again
```

所以遇到这种问题，可以先去排查`/etc/resolv.conf`里面是否配置了DNS服务器。

### 4.2 DNS服务太慢

* **现象**：DNS查询太慢
* **可能的原因**：配置的DNS服务器不合理
* **解决办法**：修改`/etc/resolv.conf`，配置合适的DNS服务器

每个公司一般都有自维护的DNS服务器，不仅用来解析内网DNS，而且可以加速解析公网域名
。

`dig`是另外一个功能更强大的DNS查询工具，安装：

```shell
/ # apk update && apk add bind-tools
```

首先查看使用内网DNS，查询域名的延迟：

```shell
/ # dig example.com
...
example.com.            15814   IN      A       93.184.216.34

;; Query time: 0 msec
;; SERVER: 192.168.1.11#53(192.168.1.11)
```

可以看到非常快，在`1ms`以内。

然后我们测试如果使用Google的公网DNS服务器`8.8.8.8` [1]，延迟会是多少。

修改`/etc/resolv.conf`，将其他`nameserver`注释掉，添加一行`nameserver 8.8.8.8`。

再次测试：

```shell
/ # dig example.com
...
example.com.            15814   IN      A       93.184.216.34

;; Query time: 150 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
```

延迟变成了`150ms`，比原来大了150多倍。

因此，对于DNS查询特别慢的场景，首先要查看配置的DNS服务器是否合理。

### 4.3 hardcode `/etc/hosts`导致跳过DNS查询

* **现象**：某域名访问太慢、某域名总是指向相同IP（多IP情况下）、特定机器不可访问
  某域名等等
* **可能的原因**：`/etc/hosts`有hardcode域名及IP
* **解决办法**：修改`/etc/hosts`

前面提到，大部分公网域名都对应多个IP地址，因此每次DNS查询拿到的IP地址都可能不一
样，我们用ping来测试一下：

```shell
/ # ping baidu.com
PING baidu.com (220.181.57.216): 56 data bytes
64 bytes from 220.181.57.216: seq=0 ttl=45 time=26.895 ms
64 bytes from 220.181.57.216: seq=1 ttl=45 time=26.701 ms
^C

/ # ping baidu.com
PING baidu.com (123.125.115.110): 56 data bytes
64 bytes from 123.125.115.110: seq=0 ttl=43 time=27.587 ms
64 bytes from 123.125.115.110: seq=1 ttl=43 time=27.757 ms
^C
```

可以看到，两次ping测试（内部首先查询baidu.com对应的IP地址）拿到的IP地址是不一样
的。用`nslookup`可以看到它们都是baidu.com对应的IP地址：

```shell
/ # nslookup baidu.com
Name:   baidu.com
Address: 220.181.57.216
Name:   baidu.com
Address: 123.125.115.110
```

`/etc/hosts`
里面可以直接harcode一个域名对应的IP地址，这会导致机器跳过DNS查询，直接拿这个IP作
为该域名的IP。我们来验证一下。

修改`/etc/hosts`，添加一行 `123.125.115.110 baidu.com`，再次ping测试

```shell
/ # ping baidu.com
PING baidu.com (123.125.115.110): 56 data bytes
64 bytes from 123.125.115.110: seq=0 ttl=43 time=27.861 ms
^C
--- baidu.com ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 27.861/27.861/27.861 ms
/ # ping baidu.com
PING baidu.com (123.125.115.110): 56 data bytes
64 bytes from 123.125.115.110: seq=0 ttl=43 time=27.614 ms
^C
```

这是不管执行多少次，baidu.com
对应的IP地址都不会变了。而实际上，这个IP地址并不一定是最优的IP地址，甚至有可能这
个IP不可用，导致访问baidu.com失败。因此，实际中要极力避免在`/etc/hosts`中
hardcode。

### 4.4 DNS查询不稳定

* **现象**：DNS查询不稳定，时快时慢
* **可能的原因**：机器上有`tc`或`iptables`规则，导致到DNS服务器的packet变慢或丢
  失
* **解决办法**：修改或删除`tc`/`iptables`规则

我们用`tc`来模拟网络延迟：

```shell
/ # apk add iproute2
```

首先查看有没有tc规则：

```shell
/ # tc -p qdisc ls dev eth0
```

默认没有任何规则。

然后我们加一条：每个packet延迟600ms：

```shell
/ # tc qdisc add dev eth0 root netem delay 600ms

/ # tc -p qdisc ls dev eth0
/ # qdisc netem 8001: root refcnt 2 limit 1000 delay 600.0ms
```

测试：

```shell
/ # dig example.com
...
example.com.            15814   IN      A       93.184.216.34

;; Query time: 600 msec
;; SERVER: 192.168.1.11#53(192.168.1.11)
```

可以看到，DNS查询变成了600ms。

这里我们测试的是**固定**延迟，这种问题很容易发现。我们还可以测试随机延迟，或者按
比例延迟等 [2]：

```shell
/ # tc qdisc change dev eth0 root netem delay 600ms 10ms 25%
/ # tc qdisc change dev eth0 root netem delay 600ms 20ms distribution normal
```

此类规则会导致DNS查询速度更有随机性。

最后删除tc规则：

```shell
/ # tc qdisc del dev eth0 root
```

`iptables`规则也会导致类似的问题。

很多软件在运行之后，会在宿主机上添加`tc`或`iptables`规则，例如OpenStack，K8S等等
。因此遇到这种随机延迟问题，首先可以查看机器上是否有`tc`或`iptables`规则。

### 4.5 DNS反向查询不稳定

线上遇到过这样一个问题：从一台机器ping一个内网域名，每个ping包看起来都会卡5～30s
不等，但是CTL-C关闭ping之后，打印出来的统计信息里，既没有丢包，ping的延迟也很低
（毫秒级），这就很奇怪。接下来：

1. `dig <URL>`，很快，毫秒级，说明DNS查询没有问题
1. `dig`能看到域名对应的IP，直接ping这个IP，发现是没有卡顿的
1. 仍然ping域名，用tcpdump抓包，`tcpdump -i eth0 host <URL> and icmp`，发现ping
   包都是立即响应的，印证了统计信息里，ping延迟很低的事实

根据以上信息，说明ping卡顿的问题出在这台机器，而且应该就是ping程序本身在做什么耗
时的操作。继续：

1. 仍然ping域名，同时，用`ltrace -p <PID>`跟踪ping进程，发现卡在一个叫
   `gethostbyaddr()`的函数
1. 查阅文档，发现这个函数是根据IP反向查询hostname，需要和DNS交互

到这里，基本确定了是DNS服务器反向查询的问题，我们用另外几个命令行工具验证一下，
以下三个命令都是根据IP反查hostname：

1. `nslookup <IP>`
1. `host <IP>`
1. `dig -x <IP>`

果然，以上三个命令都会卡住。修改`/etc/resolv.conf`，换一个DNS服务器之后，问题
消失了。接下来，就去查DNS服务器的问题吧。

1. [TOP 10 DNS Servers](https://whatsabyte.com/internet/best-public-dns-servers/)，
2. [`tc` drop packet example](https://stackoverflow.com/questions/614795/simulate-delayed-and-dropped-packets-on-linux)
3. [`docker run` parameter: `--privileged`](https://docs.docker.com/engine/reference/commandline/run/#options)
4. [Difference between Hostname and Domain Name](https://support.suso.com/supki/What_is_the_difference_between_a_hostname_and_a_domain_name)
5. [Wikipedia: Domain Name](https://en.wikipedia.org/wiki/Domain_name)