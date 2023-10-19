---
title: "Linux下使用v2ray"
author : "Jun"
date: 2021-02-28T16:06:32+08:00
categories : ["proxy"]
tags : ["Linux","proxy"]
---

v2ray是一个强大的代理工具，但苦于Linux下一直没有一个好用的客户端，便萌生了直接使用裸v2ray的念头．v2ray本身是不区分服务端和客户端的，只要配置好相关文件，即可正常使用．


## 安装v2ray

下载v2ray core

```
https://github.com/v2ray/v2ray-core/releases/
```

解压：
```
unzip v2ray-linux-64.zip -d v2ray-linux-64
```

解压后使用`mv`将相应的文件放置到对应的路径

```
v2ray -> /usr/local/bin/v2ray
v2ctl -> /usr/local/bin/v2ctl
geoip.dat -> /usr/local/share/v2ray/geoip.dat
geosite.dat -> /usr/local/share/v2ray/geosite.dat
config.json -> /usr/local/etc/v2ray/config.json
access.log -> /var/log/v2ray/access.log
error.log -> /var/log/v2ray/error.log
v2ray.service -> /etc/systemd/system/v2ray.service
v2ray@.service -> /etc/systemd/system/v2ray@.service

```

- 日志文件要保证所有人都有读写权限

- 要在配置文件中指定日志路径


## 配置文件

注意原生的V2ray并不支持订阅。

以下配置文件仅为参考，你可以将其它客户端中的配置文件完全导出，然后直接替换`/usr/local/etc/v2ray/config.json`

```
{
  "dns": {
    "hosts": {
      "domain:googleapis.cn": "googleapis.com"
    },
    "servers": [
      "1.1.1.1"
    ]
  },
  "inbounds": [
    {
      "listen": "127.0.0.1",
      "port": 10808,
      "protocol": "socks",
      "settings": {
        "auth": "noauth",
        "udp": true,
        "userLevel": 8
      },
      "sniffing": {
        "destOverride": [
          "http",
          "tls"
        ],
        "enabled": true
      },
      "tag": "socks"
    },
    {
      "listen": "127.0.0.1",
      "port": 10809,
      "protocol": "http",
      "settings": {
        "userLevel": 8
      },
      "tag": "http"
    }
  ],
  "log": {
    "loglevel": "warning",
    "access":"/var/log/v2ray/access.log",
    "error":"/var/log/v2ray/error.log"
  },
  "outbounds": [
    {
    //此处根据具体设置
      "mux": {
        "concurrency": -1,
        "enabled": false
      },
      "protocol": "vmess",
      "settings": {
        "vnext": [
          {
          //此处根据具体设置
            "address": "example.com",
            "port": 10000,
            "users": [
              {
                "alterId": 0,
                "id": "xxxxxxxxxxxx",
                "level": 8,
                "security": "auto"
              }
            ]
          }
        ]
      },
      "streamSettings": {
        "network": "tcp",
        "security": ""
      },
      "tag": "proxy"
    },
    {
      "protocol": "freedom",
      "settings": {},
      "tag": "direct"
    },
    {
      "protocol": "blackhole",
      "settings": {
        "response": {
          "type": "http"
        }
      },
      "tag": "block"
    }
  ],
  "policy": {
    "levels": {
      "8": {
        "connIdle": 300,
        "downlinkOnly": 1,
        "handshake": 4,
        "uplinkOnly": 1
      }
    },
    "system": {
      "statsOutboundUplink": true,
      "statsOutboundDownlink": true
    }
  },
  "routing": {
    "domainStrategy": "IPIfNonMatch",
    "rules": []
  },
  "stats": {}
}
```

## 使用V2ray
```
# 启动V2ray
sudo systemctl start v2ray

# 检查V2ray状态
sudo systemctl status v2ray

# 设置V2ray开机自启动
sudo systemctl enable v2ray
```

## 检验代理是否成功生效

终端下使用curl，查看它在代理模式下是否能返回数据：
```
curl -x socks5://127.0.0.1:10808 https://www.google.com -v
```
如果能返回google.com的源代码，即表示配置成功。

如果显示超时或者无法建立连接，即表示配置有错误，具体可以查看日志排查原因。

## 系统设置

当我们配置好代理后，我们很多情况下并不能开箱即用，而是需要将代理信息写入相关配置文件后，才能使用。具体操作如下：

### 设置处启用代理，并填入相关端口

打开设置，选择网络->代理->手动 并填入相应端口

此操作后可以使用浏览器测试是否能正常打开网页,如果还是失败的话，考虑修改浏览器自带的代理配置。


### 将代理设置写入shell profile

虽然我们设置了系统代理，但是终端下并不会走代理，所以还要配置以下设置

将此行写入用户的`shell profile`
如果使用的是`bash`，写入`~/.bashrc`，如果是`zsh`，写入`~/.zshrc`
如果不确定，就直接写入`~/.bashrc`
  ```
  //端口具体情况具体对待，不清楚打开代理工具看一下．
  export ALL_PROXY="socks5://127.0.0.1:10808"
  export http_proxy="http://127.0.0.1:10809"
  ```
然后我们要让配置文件生效．

```
source ~/.bashrc
```
如果还是无法走代理可以试试重开一个终端．

### 使用proxychains

对于某些不会走`socks5`的应用，我们还可以通过[proxychains](https://github.com/haad/proxychains)，使它强制走代理

  ```
  /etc/proxychains.conf
  //编辑配置文件，取消sock4配置，加入这一行
  socks5 127.0.0.1 10808
  ```
此后，直接在原始命令前加一个`proxychains`，便可走代理．比如：
```
proxychains wget https://example.com/index.html
```

### `ssh`走`socks5`代理

```
ssh -o "ProxyCommand=nc -X 5 -x 127.0.0.1:10808 %h %p" user@hostname
```
其中端口按实际情况修改，
如果想永久保持使用，可以使用`alias`做一个别名。

注意这里需要先安装`nc`

```
alias pssh='ssh -o "ProxyCommand=nc -X 5 -x 127.0.0.1:10808 %h %p"'
```
这里为了区分原始的`ssh`，我们创建了一个`pssh`，下次对于需要使用代理的，把`ssh`换成`pssh`即可．
将此行写入用户的`shell profile`
如果使用的是`bash`，写入`~/.bashrc`，如果是`zsh`，写入`~/.zshrc`
如果不确定，就直接写入`~/.bashrc`
