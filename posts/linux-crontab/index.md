# Linux Cron 定时任务


## 0x00 前言

在做一道涉及到 redis 未授权访问的题的时候，看到了使用到 `/var/spool/cron/` 目录反弹 shell

后查询得知 Linux Cron 定时任务，在此记录下来



## 0x01 相关文件

与 Cron 相关的文件主要存在以下几个目录

- `/var/spool/cron/` 目录下存放的是每个用户的 crontab 任务，每个任务以创建者的名字命名
- `/etc/crontab` 这个文件负责调度各种管理和维护任务
- `/etc/cron.d/` 这个目录用来存放任何要执行的 crontab 文件或脚本
- 我们还可以把脚本放在`/etc/cron.hourly` 、 `/etc/cron.daily` 、 `/etc/cron.weekly` 、 `/etc/cron.monthly` 目录中，让它每小时/天/星期/月执行一次

其中 `/var/spool/cron/` 为最常用的脚本存放位置，一般由 `crontab` 命令操作



## 0x02 Crontab 命令操作

查看 `crontab` 命令的帮助如下

```
Usage:
 crontab [options] file
 crontab [options]
 crontab -n [hostname]

Options:
 -u <user>  define user
 -e         edit user's crontab
 -l         list user's crontab
 -r         delete user's crontab
 -i         prompt before deleting
 -n <host>  set host in cluster to run users' crontabs
 -c         get host in cluster to run users' crontabs
 -V         print version and exit
 -x <mask>  enable debugging
```

其中常用的选项有 `-e` 、 `-l` 、 `-r`

- `-e` 用于编辑配置文件（即使用默认编辑器编辑 `/var/spool/cron/$USER` 文件）
- `-l` 用于读取配置文件（即显示 `/var/spool/cron/$USER` 文件内容）
- `-r` 用于删除配置文件（即删除 `/var/spool/cron/$USER` 文件）



## 0x03 配置文件内容

做题过程中用到的文件为 `/var/spool/cron/root` 

> **注意：**
>
> `/var/spool/cron/` 目录所有者为 root，权限为 755，只有 root 用户才能创建文件
>
> 但 crontab 生成的配置文件所有者为创建的用户为创建用户，权限为 600
>
> 所以只有在以 root 用户登录或当前登录用户已创建定时任务时才可直接修改该文件

写入的内容为（其中 `command` 即为反弹 shell 的命令）

```
* * * * * command
```

该文件中一行命令由六部分组成，前五部分分别为 分、时、日、月、周，最后一部分为待执行的命令

时间部分可用的操作符有

- `*` 所有时间
- `/` 时间间隔
- `-` 时间范围
- `,` 单独的时间

所以上述写入内容为每隔一分钟执行一次 command，即可达到自动执行的目的

另附几个上述操作符的使用示例

- `*/10 8-16 * * * command` 每天的 8-16 时每 10 分钟执行一次
- `0,30 * * * * command` 每小时的 0 分和 30 分各执行一次 
- `0 9 */2 * * command` 每隔两天的 9:00 执行一次
- `0,20,40 8-16 * * 6,0 command` 每周六和周日的 8-16 时的 0 20 40 分各执行一次
