---
title: 小试 Docker swarm mode
tags:
  - Docker
  - Swarm
url: 59.html
id: 59
categories:
  - Tech
date: 2016-12-14 18:12:37
---

最近在公司尝试将测试环境 docker 化，由于组件比较多，考虑引入容器编排方案。首先尝试使用 docker-compose 来管理各个组件实例。可以跑通基本逻辑，但是目前没有服务发现框架，只能使用 docker 内置网络中的 DNS 发现其它服务，这限定了测试环境只能跑在单机环境上，时间久了各种 OOM。最近看到 docker 新版本附带的 swarm mode，尝试搭建了一套多机 docker 环境，感觉很满足当前的需求，并且对代码基本零侵入。简单的记录了一下搭建过程以及一些注意事项。

## Swarm mode 简介

### 背景

- docker swarm 是 docker 官方编排项目之一，提供将多个 docker engine 实例变成一台虚拟 docker engine 实例的能力，便于 docker 容器集群管理
- docker 1.12 版本推出了 swarm mode 功能，将 docker swarm 深度集成到 docker engine 中，提供了整套更为简洁的 api 对 swarm 集群进行管理。相对于原生的 docker swarm，swarm mode 部署更为简单，无需外置的服务发现框架，并且提供了容器路由服务

### 概念

- Node: node 是 swarm 集群中一个 docker engine 实例
  - Manager node: 对整体集群进行管理和编排，并维护整个集群
  - Worker node: 接收 manager node 分配的 tasks，并运行，默认情况下，manager node 本身也作为一个 worker node 运行 task，可以通过改变配置让 manager node 变成一个 manager-only 的纯管理节点
- Task: swarm 集群任务的基本单元，通常是一个 docker container 的实例
- Service: 由一组同类型 task 聚合而成，是 swarm 集群中主要操作对象
- Load balancing: swarm 集群有一套内置的负载均衡策略提供外部访问到目标容器的路由，在创建 Service 时可以发布端口到外部网络上，访问 swarm 集群任意节点该端口的请求都会被路由到该 Service

## Swarm mode quick start

### 初始化 swarm 集群

1. 初始化机器，安装 docker-engine （版本需要高于 1.12）
2. 选择一台机器作为 swarm manager 节点，在 console 中运行 `docker swarm init` 创建 swarm 集群，命令会返回加入集群的命令

    ``` bash
    Swarm initialized: current node (c8dzd8xek2c9csoxj27ycl1nh) is now a manager.

    To add a worker to this swarm, run the following command:

        docker swarm join \
        --token SWMTKN-1-07bc8loetz13sy5eqv2t1uycqfr9dqabsd7x08gr22w7gj79fc-eiz5ljflasv65rnoq36aj2mfc \
        192.168.65.2:2377

    To add a manager to this swarm, run 'docker swarm join-token manager' and follow the     instructions.
    ```

3. 在另一台机器上，运行提示中的命令加入集群，如果没有保存之前的提示，可以通过 `docker swarm join-token worker` 查询

    ``` bash
    docker swarm join \
        --token SWMTKN-1-07bc8loetz13sy5eqv2t1uycqfr9dqabsd7x08gr22w7gj79fc-eiz5ljflasv65rnoq36aj2mfc \
        192.168.65.2:2377
    ```

4. 在 manager 节点上运行 `docker info` 可以看到 swarm 集群的信息

    ``` bash
    Swarm: active
     NodeID: c8dzd8xek2c9csoxj27ycl1nh
     Is Manager: true
     ClusterID: cj75fozp5effsiyuuq2dixv95
     Managers: 1
     Nodes: 2
     Orchestration:
      Task History Retention Limit: 5
     Raft:
      Snapshot Interval: 10000
      Heartbeat Tick: 1
      Election Tick: 3
     Dispatcher:
      Heartbeat Period: 5 seconds
     CA Configuration:
      Expiry Duration: 3 months
     Node Address: 192.168.65.2
    ```

5. 在节点上运行 `docker swarm leave` 可以让当前节点离开集群，如果是管理节点，需要增加 `--force` 参数

### 创建一个 service

1. 在 manager 节点上运行命令 `docker service create --replicas 1 --name redis docker-registry-cn.easemob.com/redis`创建 redis 服务, `replicas` 参数指定该服务中的 task 数量
    - 如果是私有 registry，需要在 manager 节点上执行 `docker login`，然后创建服务时，增加 `--with-registry-auth` 参数将认证信息发送到 worker 节点
2. 在 manager 节点上运行 `docker service ls` 查看所有服务状态

    ``` bash
    ID            NAME   REPLICAS      IMAGE                                     COMMAND
    0nqfp11n2xgb  redis  1/1           docker-registry-cn.easemob.    com/redis
    ```

3. 在 manager 节点上运行 `docker service ps redis` 查看 redis 服务状态

    ``` bash
    ID                         NAME     IMAGE                                 NODE  DESIRED STATE  CURRENT STATE         ERROR
    3qpkmxptn8o336h161183a0vz  redis.1  docker-registry-cn.easemob.com/redis  moby  Running        Running 17 hours ago
    ```

4. 在 manager 节点上运行 `docker service update redis` 更新服务。例如可以用 `docker service update redis --publish-add 6379:6379` 发布 redis 服务到 swarm 集群 6379 端口上（也可以在创建服务时使用 --publish 参数发布），在 swarm 任意节点上运行 `redis-cli` 都可以连接到 redis 服务
5. 在 manager 节点上运行 `docker service rm redis` 可以删除服务

### Service 间网络调用

- 如果 service publish 了端口，可以通过任意 swarm 节点 publish 端口进行调用

    ``` bash
    ➜  ~ docker service update --publish-add 6379:6379 redis1
    redis1
    ➜  ~ telnet localhost 6379
    Trying ::1...
    Connected to localhost.
    Escape character is '^]'.
    set test aaa
    +OK
    ```

- 创建一个 overlay 的网络，创建服务时，通过 `--network` 参数连接到 overlay 网络上。通过内建的 DNS 服务，可以通过 service name 解析出 task IP

    ``` bash
    ➜  ~ docker network create --driver overlay --subnet 10.0.10.0/24 sandbox
    e7q4c68ybbw7xhrkb0at47k00
    ➜  ~ docker service create --with-registry-auth --replicas 1 --network sandbox --name redis1  docker-registry-cn.easemob.com/redis
    0dfqszxp925id4ma9blolyjmz
    ➜  ~ docker service create --with-registry-auth --replicas 1 --network sandbox --name redis2  docker-registry-cn.easemob.com/redis
    c3khd0snpu13ixx8popuoludx
    ➜  ~ docker ps
    CONTAINER ID        IMAGE                                         COMMAND                  CREATED             STATUS              PORTS               NAMES
    4c92e01d25fd        docker-registry-cn.easemob.com/redis:latest   "/entrypoint.sh redis"   12 seconds ago      Up 9 seconds        6379/tcp            redis2.1.4qbw5c0hawat5rr50hgzpxj7l
    c60b4659e768        docker-registry-cn.easemob.com/redis:latest   "/entrypoint.sh redis"   6 minutes ago       Up 6 minutes        6379/tcp            redis1.1.4pvn93a6i2n5teou41mddmlxr
    ➜  ~ docker exec -it c60b4659e768 /bin/bash
    root@c60b4659e768:/data# ping redis2
    PING redis2 (10.0.10.4): 56 data bytes
    64 bytes from 10.0.10.4: icmp_seq=0 ttl=64 time=0.109 ms
    ```

### 一些坑点

- 截止 docker 最新版本（1.12.3），在 overlay 网络中，以 VIP 方式启动的服务，可以解析 DNS，但是无法 `ping` IP，某些依赖 `ping` 检查网络连通的程序会有问题，见 [https://github.com/docker/docker/issues/25497](https://github.com/docker/docker/issues/25497)。解决方案是启动模式换成 dnsrr （创建服务时增加参数 --endpoint-mode dnsrr），或者使用 tasks.${service_name} 的方式调用
- 在某些操作系统（例如 Suse）上，service publish 的端口无法连接，原因不明，可能和缺少某些内核模块有关。建议使用 Centos7
- swarm manager 会在 service 中 task 退出时重新提交 task，某些需要一次性运行的 docker image （例如环境初始化脚本的执行）endpoint 或者 cmd 需要修改成 block 形式，以免退出后被重复提交。Docker 1.13 版本允许 `docker run` 启动的容器 attach 到 swarm service 使用的 network 上，可以将一次性运行的 image 直接以 `docker run` 的方式启动
- `docker service update --image myimage:latest` 不会拉取新镜像，即使远端有更新的版本，只能通过手动拉取镜像后重启 service 来更新服务。这个 bug 会在 docker 1.13 上被修复

Have fun :)