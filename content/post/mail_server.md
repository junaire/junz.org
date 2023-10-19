---
title : "Postfix & Dovecot 自建邮箱服务"
author : "Jun"
date : 2021-02-28T15:38:40+08:00
lastmod : 2021-02-28T15:38:40+08:00
categories : ["Linux"]
tags : ["Linux"]
---

**本文中example.com hostname password请修改为自己的配置**

#### 环境

- 系统CentOS 7

- postfix

- dovecot

- mariadb

- opendkim

- nginx

  
#### 安装必备软件

```bash
yum -y update && \
yum -y install epel-release && \
yum -y update && \
yum -y install dovecot dovecot-mysql mariadb-server nginx opendkim python2-certbot-nginx nginx
```



#### 修改hosts文件

```bash
127.0.0.1 localhost.localdomain localhost
你的公网ip hostname.example.com hostname
```



#### nginx 设置

配置好相关设置



#### 安装证书

```
certbot --nginx
```



#### 开启防火墙

```bash
firewall-cmd --zone=public --permanent --add-service=http
firewall-cmd --zone=public --permanent --add-service=https
firewall-cmd --zone=public --add-port=80/tcp --permanent
firewall-cmd --zone=public --add-port=25/tcp --permanent
firewall-cmd --zone=public --add-port=465/tcp --permanent
firewall-cmd --zone=public --add-port=587/tcp --permanent
firewall-cmd --zone=public --add-port=110/tcp --permanent
firewall-cmd --zone=public --add-port=995/tcp --permanent
firewall-cmd --zone=public --add-port=143/tcp --permanent
firewall-cmd --permanent --add-service={smtp,smtp-submission,smtps,imap,imaps}
firewall-cmd --reload
```





#### 配置数据库

##### 启动数据库

```bash
systemctl start mariadb
systemctl enable mariadb

mysql_secure_installation #安全安装mysql，设置root登录密码
```

##### 创建相关数据库


```sql
CREATE USER 'mail_sys'@'localhost' IDENTIFIED BY 'mail_sys'; //创建用户
CREATE DATABASE mail_sys; //创建数据库
GRANT SELECT ON mail_sys.* TO 'mail_sys'@'localhost' IDENTIFIED BY 'mail_sys'; //提权
FLUSH PRIVILEGES; //刷新权限
USE mail_sys;  //进入数据库

CREATE TABLE `domains` (
  `id` int(11) NOT NULL auto_increment,
  `name` varchar(50) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;   //创建域名表

CREATE TABLE `users` (
  `id` int(11) NOT NULL auto_increment,
  `domain_id` int(11) NOT NULL,
  `password` varchar(106) NOT NULL,
  `email` varchar(100) NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `email` (`email`),
  FOREIGN KEY (domain_id) REFERENCES virtual_domains(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8;    //创建用户表

CREATE TABLE `aliases` (
  `id` int(11) NOT NULL auto_increment,
  `domain_id` int(11) NOT NULL,
  `source` varchar(100) NOT NULL,
  `destination` varchar(100) NOT NULL,
  PRIMARY KEY (`id`),
  FOREIGN KEY (domain_id) REFERENCES virtual_domains(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8;    //创建别名表
```

##### 添加数据


```sql
INSERT INTO `mail_sys`.`domains`
  (`id` ,`name`)
VALUES
  ('1', 'example.com'),
  ('2', 'hostname.example.com'),
  ('3', 'hostname'),
  ('4', 'localhost.example.com');
  
INSERT INTO `mail_sys`.`users`
  (`id`, `domain_id`, `password` , `email`)
VALUES
  ('1', '1', ENCRYPT('PASSWORD', CONCAT('$6$', SUBSTRING(SHA(RAND()), -16))), 'user@example.com');
  
INSERT INTO `mail_sys`.`aliases`
  (`id`, `domain_id`, `source`, `destination`)
VALUES
  ('1', '1', 'user@example.com', 'user@example.com');
```

##### 检查配置

```sql
SELECT * FROM mail_sys.domains; //查看域名表
SELECT * FROM mail_sys.users;   //查看用户表
SELECT * FROM mail_sys.aliases;  //查看别名表
```

#### 设置邮箱专用用户

```bash
groupadd -g 2000 mail_sys
useradd -g mail_sys -u 2000 mail_sys -d /var/spool/mail -s /sbin/nologin
chown -R mail_sys:mail_sys /var/spool/mail
```
#### 配置OpenDKIM

##### 修改配置文件

```
/etc/opendkim.conf

Syslog yes
UMask 002
OversignHeaders From
Socket inet:8891@127.0.0.1
Domain example.com # 您的域名，需要修改
KeyFile /etc/opendkim/keys/mail.private
Selector mail
RequireSafeKeys no
```

##### 生成私钥

```bash
opendkim-genkey -D /etc/opendkim/keys/ -d example.com -s mail && \
chown -R opendkim:opendkim /etc/opendkim/keys/
```


#### 配置postfix

```bash
cp /etc/postfix/main.cf /etc/postfix/main.cf.bak  //备份文件
cp /etc/postfix/master.cf /etc/postfix/master.cf.bak 
```

修改main.cf配置文件

```
/etc/postfix/main.cf
```

```
# See /usr/share/postfix/main.cf.dist for a commented, more complete version

# Debian specific:  Specifying a file name will cause the first
# line of that file to be used as the name.  The Debian default
# is /etc/mailname.
#myorigin = /etc/mailname

smtpd_banner = $myhostname ESMTP $mail_name (CentOS)
biff = no

# appending .domain is the MUA's job.
append_dot_mydomain = no

# Uncomment the next line to generate "delayed mail" warnings
#delay_warning_time = 4h

readme_directory = no

# TLS parameters
smtpd_tls_cert_file=/etc/letsencrypt/live/example.com/fullchain.pem
smtpd_tls_key_file=/etc/letsencrypt/live/example.com/privkey.pem
smtpd_use_tls=yes
smtpd_tls_auth_only = yes
smtp_tls_security_level = may
smtpd_tls_security_level = may
smtpd_sasl_security_options = noanonymous, noplaintext
smtpd_sasl_tls_security_options = noanonymous

# Authentication
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
smtpd_sasl_auth_enable = yes

# See /usr/share/doc/postfix/TLS_README.gz in the postfix-doc package for
# information on enabling SSL in the smtp client.

# Restrictions
smtpd_helo_restrictions =
        permit_mynetworks,
        permit_sasl_authenticated,
        reject_invalid_helo_hostname,
        reject_non_fqdn_helo_hostname
smtpd_recipient_restrictions =
        permit_mynetworks,
        permit_sasl_authenticated,
        permit_auth_destination,
        reject_unauth_destination
smtpd_sender_restrictions =
        permit_mynetworks,
        permit_sasl_authenticated,
        reject_non_fqdn_sender,
        reject_unknown_sender_domain

# See /usr/share/doc/postfix/TLS_README.gz in the postfix-doc package for
# information on enabling SSL in the smtp client.

myhostname = example.com
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
mydomain = example.com
myorigin = $mydomain
mydestination = localhost
relayhost =
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
mailbox_size_limit = 0
recipient_delimiter = +
inet_interfaces = all
inet_protocols = all

# Handing off local delivery to Dovecot's LMTP, and telling it where to store mail
virtual_transport = lmtp:unix:private/dovecot-lmtp

# Virtual domains, users, and aliases
virtual_mailbox_domains = mysql:/etc/postfix/mysql_mailbox_domains.cf
virtual_mailbox_maps = mysql:/etc/postfix/mysql_mailbox_maps.cf
virtual_alias_maps = mysql:/etc/postfix/mysql_alias_maps.cf
smtpd_sender_login_maps = mysql:/etc/postfix/mysql_mailbox_maps.cf, mysql:/etc/postfix/mysql_alias_maps.cf
# Even more Restrictions and MTA params
disable_vrfy_command = yes
strict_rfc821_envelopes = yes
#smtpd_etrn_restrictions = reject
#smtpd_reject_unlisted_sender = yes
#smtpd_reject_unlisted_recipient = yes
smtpd_delay_reject = yes
smtpd_helo_required = yes
smtp_always_send_ehlo = yes
#smtpd_hard_error_limit = 1
smtpd_timeout = 30s
smtp_helo_timeout = 15s
smtp_rcpt_timeout = 15s
smtpd_recipient_limit = 40
minimal_backoff_time = 180s
maximal_backoff_time = 3h

# Reply Rejection Codes
invalid_hostname_reject_code = 550
non_fqdn_reject_code = 550
unknown_address_reject_code = 550
unknown_client_reject_code = 550
unknown_hostname_reject_code = 550
unverified_recipient_reject_code = 550
unverified_sender_reject_code = 550


milter_protocol = 2
milter_default_action = accept
smtpd_milters = inet:127.0.0.1:8891
non_smtpd_milters = inet:127.0.0.1:8891
```

修改master.cf配置文件

```
/etc/postfix/master.cf
```

```
smtp      inet  n       -       n       -       -       smtpd
submission inet n       -       n       -       -       smtpd
       -o smtpd_tls_security_level=encrypt
smtps     inet  n       -       n       -       -       smtpd
       -o smtpd_tls_wrappermode=yes
pickup    unix  n       -       n       60      1       pickup
cleanup   unix  n       -       n       -       0       cleanup
qmgr      unix  n       -       n       300     1       qmgr
tlsmgr    unix  -       -       n       1000?   1       tlsmgr
rewrite   unix  -       -       n       -       -       trivial-rewrite
bounce    unix  -       -       n       -       0       bounce
defer     unix  -       -       n       -       0       bounce
trace     unix  -       -       n       -       0       bounce
verify    unix  -       -       n       -       1       verify
flush     unix  n       -       n       1000?   0       flush
proxymap  unix  -       -       n       -       -       proxymap
proxywrite unix -       -       n       -       1       proxymap
smtp      unix  -       -       n       -       -       smtp
relay     unix  -       -       n       -       -       smtp
       -o smtp_helo_timeout=120 -o smtp_connect_timeout=120
showq     unix  n       -       n       -       -       showq
error     unix  -       -       n       -       -       error
retry     unix  -       -       n       -       -       error
discard   unix  -       -       n       -       -       discard
local     unix  -       n       n       -       -       local
virtual   unix  -       n       n       -       -       virtual
lmtp      unix  -       -       n       -       -       lmtp
anvil     unix  -       -       n       -       1       anvil
scache    unix  -       -       n       -       1       scache
policyd-spf    unix  -       n       n       -       0       spawn
       user=mail_sys argv=/usr/libexec/postfix/policyd-spf
```



##### 修改postfix相关配置文件，与MySQL对接

```
/etc/postfix/mysql_mailbox_domains.cf

user = mail_sys
password = mail_sys
hosts = localhost
dbname = mail_sys
query = SELECT 1 FROM domains WHERE name='%s'
```

```
/etc/postfix/mysql_mailbox_maps.cf

user = mail_sys
password = mail_sys
hosts = localhost
dbname = mail_sys
query = SELECT email FROM users WHERE email='%s'
```

```
/etc/postfix/mysql_alias_maps.cf

user = mail_sys
password = mail_sys
hosts = localhost
dbname = mail_sys
query = SELECT destination FROM aliases WHERE source='%s'
```

##### 测试postfix与MySQL对接状态

```bash
systemctl start postfix
postmap -q example.com mysql:/etc/postfix/mysql_mailbox_domains.cf #应返回1
postmap -q user@example.com mysql:/etc/postfix/mysql_mailbox_maps.cf #应返回1
postmap -q user@example.com mysql:/etc/postfix/mysql_alias_maps.cf  #应返回用户名
```

#### 配置dovecot



##### 修改配置文件

```
/etc/dovecot/dovecot-sql.conf.ext


driver = mysql
connect = host=localhost dbname=mail_sys user=mail_sys password=mail_sys
default_pass_scheme = SHA512-CRYPT
password_query = SELECT email as user, password FROM users WHERE email='%u';
```



```
/etc/dovecot/conf.d/10-master.conf


service imap-login {
  inet_listener imap {
    port = 143
  }
  inet_listener imaps {
    port = 993
    ssl = yes
  }
}

service lmtp {
    unix_listener /var/spool/postfix/private/dovecot-lmtp {
    mode = 0600
    user = postfix
    group = postfix
  }
}

service imap {

}

service auth {
  unix_listener /var/spool/postfix/private/auth {
    mode = 0666
    user = postfix
    group = postfix
  }

  unix_listener auth-userdb {
    mode = 0600
    user = mail_sys
  }
  user = dovecot
}

service auth-worker {
  user = mail_sys
}
```



```
/etc/dovecot/dovecot.conf

protocols = imap lmtp
dict {
}
!include conf.d/*.conf
!include_try local.conf
```

```
/etc/dovecot/conf.d/10-mail.conf

namespace inbox {
  inbox = yes
}
first_valid_uid = 1000
mbox_write_locks = fcntl
mail_location = maildir:/var/spool/mail/%d/%n
mail_privileged_group = mail
```

```
/etc/dovecot/conf.d/15-mailboxes.conf

namespace inbox {
  mailbox Drafts {
    auto = create
    special_use = \Drafts
  }
  mailbox Trash {
    auto = create
    special_use = \Trash
  }
  mailbox Sent {
    auto = create
    special_use = \Sent
  }
}
```

```
/etc/dovecot/conf.d/10-auth.conf

auth_mechanisms = plain login
!include auth-sql.conf.ext
```

```
/etc/dovecot/conf.d/auth-sql.conf.ext

passdb {
  driver = sql
  args = /etc/dovecot/dovecot-sql.conf.ext
}
userdb {
  driver = static
  args = uid=mail_sys gid=mail_sys home=/var/spool/mail/%d/%n
}
```

```
/etc/dovecot/conf.d/10-ssl.conf 

ssl = required
ssl_cert = </etc/letsencrypt/live/example.com/fullchain.pem
ssl_key = </etc/letsencrypt/live/example.com/privkey.pem
ssl_protocols = TLSv1.2 TLSv1.1 !TLSv1 !SSLv2 !SSLv3
ssl_cipher_list = ALL:!MD5:!DES:!ADH:!RC4:!PSD:!SRP:!3DES:!eNULL:!aNULL
```

```
/etc/dovecot/conf.d/15-lda.conf

protocol lda {
}
```



#### DNS解析

| hostname        | TYPE | VALUE              |
| --------------- | ---- | ------------------ |
| @               | A    | IP address         |
| @               | MX   | example.com        |
| @               | TXT  | v=spf1 mx -all     |
| _dmarc          | TXT  | v=DMARC1; p=reject |
| mail._domainkey | TXT  | 密钥，见下文       |

密钥文件

```bash
cat /etc/opendkim/keys/mail.txt #密钥即为第二个引号里的全部内容
```



#### 检查邮箱状态

```
yum install mailx
mail email1@example.com
tail /var/log/maillog
```

