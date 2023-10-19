---
title : "Linux服务浅谈"
author : "Jun"
date : 2021-02-28T15:35:10+08:00
---



## `systemctl`相关命令
```
systemctl start daemon //启动一个服务
systemctl stop daemon //停止一个服务
systemctl enable daemon //开机自启动
systemctl disable daemon //取消开机自启动
systemctl restart daemon //重新启动服务
```
```
systemctl mask daemon //将整个服务链接到/dev/null上，使得不用管其他依赖服务便可关闭此服务
systemctl umask daemon
```

```
systemctl list-dependencies name.service //列出各个服务的依赖关系
```

## 配置文件

- 每个服务最主要的启动设置　`/usr/lib/systemd/system`
- 系统执行中所产生的服务脚本　`/run/systemd/system`
- 管理员所创建的服务脚本  `/etc/systemd/system`

上述配置文件的优先级依次升高
Linux 会依次读取下列目录中的配置文件
`/etc/systemd/system`　
            -> 
            `/run/systemd/system` 
                         -> 
                         `/usr/lib/systemd/system`

一般来说，`/etc/systemd/system`中主要都是软链接，真正配置文件位于`/lib/systemd/system`

不建议直接修改`usr/lib/systemd/sytem`中的配置文件

## `systemd`的unit 类型

|拓展名|功能|
|:---:|:---:|
|.service|指一般的服务类型|
|.target|service的集合|



## 编写自己的service文件

```
[Unit]
Description="此服务的描述"
Documentation="此服务的文档，可以是网页链接或者man page"
After="说明该unit在哪个daemon启动后再启动"
Before="说明该unit在哪个daemon启动前启动"

[Service]
User=nobody
NoNewPrivileges=true
ExecStart="实际执行此服务的脚本！注意尽量不要使用一些特殊字符！"
ExecStop="实际停止此服务的脚本"
Restart=on-failure
RestartPreventExitStatus=23

[Install]
WantedBy=multi-user.target　#表明此服务所在的target unit
```



## 定时任务

#### 编辑定时任务

```bash
crontab -e
```



#### 查看当前定时任务

```bash
crontab -l
```



#### 删除定时任务

```bash
crontab -r
```



#### 时间基本语法

```
*    *    *    *    *
-    -    -    -    -
|    |    |    |    |
|    |    |    |    +----- 星期中星期几 (0 - 7) (星期天 为0)
|    |    |    +---------- 月份 (1 - 12) 
|    |    +--------------- 一个月中的第几天 (1 - 31)
|    +-------------------- 小时 (0 - 23)
+------------------------- 分钟 (0 - 59)
```



#### 示例

```
* * * * * myCommand  //每1分钟执行一次myCommand
3,15 * * * * myCommand  //每小时的第3和第15分钟执行
3,15 8-11 * * * myCommand   //在上午8点到11点的第3和第15分钟执行
* */1 * * * /etc/init.d/smb restart  //每一小时重启smb
```

####  

#### `crontab`基本操作

```
//cent os 7
systemctl restart crond //重启
systemctl start crond  //启动
systemctl stop crond  //停止

tail -f /var/log/cron  //查看日志
```



#### python实例

```
*/5 * * * * /usr/bin/python3 /root/example.py //注意文件首行要加上 #!/usr/bin/python3
```



#### 注意点

- 用绝对路径，路径写全
- 注意环境变量
