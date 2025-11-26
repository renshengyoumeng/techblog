---
title: 如何安装snac
description: 介绍如何安装snac
tags: snac, how-to-install
category: tryit
---


## 内容参考：
- https://www.bandwagonhost.net/12445.html 
- https://blog.example_user.moe/posts/selfhost-snac-activitypub-on-debian-with-nginx/


<!--more-->
----------


## 添加一个新账户

使用 adduser 命令创建一个新用户帐户。为新用户使用强密码。您可以输入用户信息的值，或按 ENTER 将这些字段留空。

```bash
# adduser example_user
Adding user `example_user' ...
Adding new group `example_user' (1001) ...
Adding new user `example_user' (1001) with group `example_user' ...
Creating home directory `/home/example_user' ...
Copying files from `/etc/skel' ...
New password:
Retype new password:
passwd: password updated successfully
Changing the user information for example_user
Enter the new value, or press ENTER for the default
        Full Name []: Example User
        Room Number []:
        Work Phone []:
        Home Phone []:
        Other []:
Is the information correct? [Y/n] y
```


## 将用户添加到 Sudo 组
将新用户添加到 sudo 组。

```bash
# adduser example_user sudo
```


## 测试

切换到新用户。

```bash
# su - example_user
```

使用 `whoami` 验证您是新用户，然后使用 `sudo whoami` 测试 sudo 访问权限，这应该返回 root。

```bash
$ whoami
example_user
$ sudo whoami
[sudo] password for example_user:
root
```

新用户帐户已准备好使用。作为最佳实践，使用此 sudo 用户进行服务器管理。您应该避免使用 root 来执行维护任务。

## 编译安装 snac

```bash
sudo apt install git libssl-dev libcurl4-openssl-dev build-essential
```

拉取源码
确保当前工作目录为非 root 用户，因为 snac 始终使用普通用户运行

### 拉取最新源码
```bash
git clone https://codeberg.org/grunfink/snac2.git && cd snac2
```bash

### 开始编译
添加 -lrt 标志3 开始编译

```bash
make LDFLAGS=-lrt
```

编译过程只需要几秒钟，因为它实在是太小了

安装

```bash
$ sudo make install
# 正常输出
mkdir -p -m 755 /usr/local/bin
install -m 755 snac /usr/local/bin/snac
mkdir -p -m 755 /usr/local/man/man1
install -m 644 doc/snac.1 /usr/local/man/man1/snac.1
mkdir -p -m 755 /usr/local/man/man5
install -m 644 doc/snac.5 /usr/local/man/man5/snac.5
mkdir -p -m 755 /usr/local/man/man8
install -m 644 doc/snac.8 /usr/local/man/man8/snac.8
```bash

配置 snac
创建 snac 数据目录
你需要指定一个用于存储服务器和用户数据（如文章、媒体文件）的目录。请注意，这个目录在创建 snac 实例前必须不存在。下面以当前用户 home 目录下的 snac-data 为例：

```bash
# 初始化 snac 数据目录
$ snac init $HOME/snac-data
# 监听网络/Unix 套接字地址（默认 127.0.0.1）
Network address or full path to unix socket [127.0.0.1]:  
# 监听端口号（默认 8001）
Network port [8001]: 
# 主机名（输入自己的域名）
Host name: snac.your.domain
# URL 前缀（默认为空）
URL prefix: 
# 管理员邮箱（可选）
Admin email address (optional): admin@your-email.domain
Done.
```

# 添加 WebUI 翻译文件（可选，默认 en）
Wanted web UI language files (.po) must be copied manually to /home/example_user/snac-data/lang
初始化后的 snac-data 目录应该如下

```bash
~/snac-data$ ls -al
total 40
drwxrws---  7 example_user example_user 4096 Sep 18 14:37 .
drwxr-xr-x 13 example_user example_user 4096 Sep 18 14:37 ..
-rw-rw----  1 example_user example_user  923 Sep 18 14:37 greeting.html
drwxrws---  2 example_user example_user 4096 Sep 18 14:37 inbox
drwxrws---  2 example_user example_user 4096 Sep 18 14:37 lang
drwxrws---  2 example_user example_user 4096 Sep 18 14:37 object
drwxrws---  2 example_user example_user 4096 Sep 18 14:37 queue
-rw-rw----  1 example_user example_user  619 Sep 18 14:37 server.json
-rw-rw----  1 example_user example_user 2134 Sep 18 14:37 style.css
drwxrws---  2 example_user example_user 4096 Sep 18 14:37 user
```

配置文件保存在 server.json 文件中。详细的配置说明，请参考 snac 官方文档：Customization snac。

i18n 多语言
WebUI 默认是英文，可以自定义 i18n 语言，以简体中文为例

```bash
wget -P lang/ https://codeberg.org/grunfink/snac2/raw/branch/master/po/zh.po
```

添加用户
snac 的用户管理完全在服务端进行，这表示它 默认不开放注册。因此，你无需配置繁琐的 SMTP 服务来处理用户注册或找回密码等事宜。若需要重置用户密码，也只需通过命令行简单操作即可。下面添加我们的第一个用户：
```bash
snac adduser $HOME/snac-data mod # mod 是添加的用户名

Creating RSA key...
Done.

User password is <初始密码，首次登录及时修改>

Go to https://snac.example_user.moe/mod and continue configuring your user there.
```

守护进程
仅需添加一个 systemd 服务来守护 snac 的进程管理
```bash
sudo sh -c 'cat > /etc/systemd/system/snac.service << EOF
[Unit]
Description=snac
After=network.target

[Service]
Type=simple
Restart=always
RestartSec=5
ExecStart=/usr/local/bin/snac httpd /home/example_user/snac-data # <-注意修改路径

[Install]
WantedBy=multi-user.target
EOF'
```

启动 snac 服务

```bash
# 重新加载 systemd 配置
sudo systemctl daemon-reload
# 将 snac 服务设置为开机自启动
sudo systemctl enable snac.service
# 立即启动 snac 服务
sudo systemctl start snac.service
# 查看 snac 服务状态
sudo systemctl status snac
```

### 安装caddy
```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
chmod o+r /usr/share/keyrings/caddy-stable-archive-keyring.gpg
chmod o+r /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy
```