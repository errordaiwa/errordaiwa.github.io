---
title: DNS 配置有误导致监控误报的问题排查
tags:
  - DNS
  - Docker
  - Problem
url: 65.html
id: 65
categories:
  - Tech
date: 2018-08-09 12:58:33
---

背景
--

- [XXL-Job](https://github.com/xuxueli/xxl-job) 是一套开源的任务调度框架，目前服务端部署了一套 XXL-Job 服务，用于监控线上服务质量
- 由于 Ucloud 出现过几次内网域名服务器故障，我们在 XXL-Job 中增加了定时任务监控第三方域名连通状态，并报警到钉钉群

问题描述
----

- 添加三方域名报警之后，钉钉群中经常收到报警，但是实际观察时，发现服务并无问题，报警为误报

排查过程
----

- 查看报警对应的执行脚本，核心代码如下

    ``` bash
    ping -c 5 target.com
    if (($? == 0))
    then
        //正常
    else
        //报警
        exit 1;
    fi
    ```

- 查看报警对应的脚本执行日志，发现如下报错

    ``` bash
    ping: unknown host
    ```

- 初步判断是域名解析有问题导致报警，使用 `dig` 和 `nslookup` 查看域名解析情况，结果正常返回。由于报警也是偶发，觉得可能是域名解析偶发失败导致 `ping` 命令报错

    ``` bash
    [work@monitor01 ~]$ nslookup target.com
    Server:       10.9.255.1
    Address:  10.9.255.1#53

    Non-authoritative answer:
    Name: target.com
    Address: xx.xx.xx.xx
    ```

- 在 XXL-Job 部署的机器上尝试复现问题，脚本如下。执行脚本，直到有报警时停止，查看日志，没有失败的情况

    ``` bash
    while true
    do
      echo "`date`============"
      ping -c1 target.com
      echo "result: $?"
    done
    ```

- 在 XXL-Job 机器上观察 XXL-Job 进程，使用 `ps -elf` 命令追踪父进程。发现进程是运行在 Docker 容器中的

    ``` bash
    [work@monitor01 ~]$ ps -elf | grep 31525
    0 S root     31525  1689  0  80      0 - 86935 futex_ Jun29 ?           00:16:10    /usr/bin/docker-containerd-shim    current    6b0f5c6aa2f52318867466a32774071    f7bbf540779ef92bb16ccf00494c6c5    /var/run/docker/libcontainerd/6    0f5c6aa2f52318867466a327740711f    bbf540779ef92bb16ccf00494c6c5e    /usr/libexec/docker/docker-runc-current
    ```

- 容器和宿主机的网络环境是隔离的，所以 XXL-Job 报警时，宿主机上的脚本没有复现问题。于是在容器中继续执行脚本，尝试复现问题
- 脚本执行几分钟后，在日志中观察到有报错的情况。这里注意到，报错的地方似乎花费的时间比较久

    ``` bash
    Mon Jul  2 08:30:35 UTC     2018============
    ping: unknown host
    result: 1
    Mon Jul  2 08:30:37 UTC     2018============
    ```

- 对脚本进行调整，记录调用时间

    ``` bash
    time ping -c1 target.com
    ```

- 重新执行脚本，发现报错时，`ping` 命令的调用花费了一秒左右的时间。感觉可能是 DNS 解析超时了。

    ``` bash
    Mon Jul  2 09:43:12 UTC     2018============
    ping: unknown host

    real    0m1.049s
    user    0m0.002s
    sys     0m0.002s
    result: 1
    ```

- 查看容器 DNS 解析配置

    ``` bash
    root@f9c78250c2a0:~# cat /etc/resolv.conf
    search xxx.com
    nameserver 127.0.0.11
    options timeout:1 attempts:1 rotate single-request-reopen ndots:0
    ```

- 通过 `man resolv.conf` 查阅 `resolv.conf` 参数的含义。发现容器内的 DNS 解析超时时间配置成了一秒，并且只尝试一次。DNS 使用 UDP 协议传输数据。UDP 协议设计上就是不可靠的，所以会存在丢包的情况，而且 DNS 解析需要去公网域名服务器请求数据，延迟也有不确定性。当 DNS 解析失败时，`ping` 命令就会报错，进而触发报警。

    ``` properties
    timeout:n
      Sets the amount of time the resolver will wait for a response from a remote name server before retrying the query via a different name server.  Measured in seconds, the  default is RES_TIMEOUT (currently 5, see <resolv.h>).  The value for this option is silently capped to 30.

    attempts:n
      Sets  the  number  of  times  the resolver will send a query to its name servers before giving up and returning an error to the calling application.  The default is RES_DFLRETRY (currently 2, see <resolv.h>).  The value for this option is silently capped to 5.
    ```

- 问题到这里基本定位完成，开始尝试解决问题。查阅 Docker 官方文档中关于[网络的配置](https://docs.docker.com/config/containers/container-networking/#dns-services)，发现可以通过 `dns_opt` 参数配置容器的 DNS 解析配置

    ``` bash
    --dns-opt A key-value pair representing a DNS option and its value. See your operating system’s documentation for resolv.conf for valid options.
    ```

- 修改容器配置，将 DNS 解析超时设置为 2s，重试一次。重启服务，观察半个小时，不再出现误报，问题解决

    ``` yaml
    dns_opt:
      - timeout:2
      - attempts:2
      - single-request
      - rotate
    ```

一句话总结
-----

容器和宿主机 DNS 解析配置不同，导致容器内 ping 命令调用偶发失败，引发报警误报。通过 `dns_opt` 参数修改容器 DNS 超时时间和重试次数后，问题解决。