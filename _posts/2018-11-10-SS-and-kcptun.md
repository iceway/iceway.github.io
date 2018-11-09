---
layout: post
title:  "$$使用KCPTUN加速"
date: Sat, 10 Nov 2018 00:03:53 +0800
categories: 加速
tags: KCPTUN VPS
---

本文记录在OpenVZ架构的VPS中安装$$以及配置KCPTUN加速的方法，以及在客户主机的安装配置和连接。

## Server

服务端使用的操作系统是CentOS7。

### 安装$$

直接使用以下的一键安装脚本即可。

```bash
wget --no-check-certificate -O shadowsocks-libev.sh \
	https://raw.githubusercontent.com/teddysun/shadowsocks_install/master/shadowsocks-libev.sh
bash shadowsocks-libev.sh
# port建议选择443
```

配置文件大致如下

```json
{
	"server":"0.0.0.0",
	"server_port":443,
	"password":"<pass1>",
	"timeout":300,
	"user":"nobody",
	"method":"aes-256-cfb",
	"fast_open":false,
	"nameserver":"8.8.8.8",
	"mode":"tcp_and_udp"
}
```

### 安装KCPTUN

```bash
wget --no-check-certificate -O kcptun-linux-amd64-20180810.tar.gz \
	https://github.com/xtaci/kcptun/releases/download/v20180810/kcptun-linux-amd64-20180810.tar.gz
tar zxvf kcptun-linux-amd64-20180810.tar.gz && rm -f client_linux_amd64
mv server_linux_amd64 /usr/bin && chmod +x /usr/bin/server_linux_amd64

cat >/etc/kcp-conf.json <<EOF
{
	"listen":"<pubip>:44333",
	"target":"127.0.0.1:443",
	"key":"<pass2>",
	"crypt":"aes-192",
	"mode":"fast2"
}
EOF

cat >/etc/systemd/system/kcp-server.service <<EOF
[Unit]
Description=Kcptun server
After=network.target

[Service]
ExecStart=/usr/bin/server_linux_amd64 -c /etc/kcp-conf.json
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable kcp-server
systemctl restart kcp-server
```

## Client

Client使用的是Ubuntu系统。

### 安装$$

```bash
apt install shadowsocks-libev
systemctl stop shadowsocks-libev
systemctl disable shadowsocks-libev
```

配置大致如下：

```json
{
	"server":"127.0.0.1",
	"server_port":2080,
	"local_address":"127.0.0.1",
	"local_port":1080,
	"password":"<pass1>",
	"timeout":600,
	"method":"aes-256-cfb"
}
```

启动

```bash
ss-local -c '<config-file>' 2>/dev/null &
```

### 安装kcptun

```bash
wget --no-check-certificate -O kcptun-linux-amd64-20180810.tar.gz \
	https://github.com/xtaci/kcptun/releases/download/v20180810/kcptun-linux-amd64-20180810.tar.gz
tar zxvf kcptun-linux-amd64-20180810.tar.gz && rm -f server_linux_amd64
mv client_linux_amd64 /usr/bin/kcptun-client && chmod +x /usr/bin/kcptun-client
```

配置文件大致如下:

```json
{
	"localaddr" : ":2080",
	"remoteaddr" : "<pubip>:44333",
	"key" : "<pass2>",
	"crypt": "aes-192",
	"mode" : "fast2"
}
```

启动

```bash
kcptun-client -c '<config-file>' >/dev/null 2>&1 &
```

最后就是在浏览器中启用socks代理了，这里不再赘述。
