---
title: WireGuard 客户端的安装与配置使用
---

记录 mac os 下安装 WireGuard 的使用过程

## 安装 wireGuard-tools

```bash
sudo brew install wireguard-tools
```

## 配置 wireGuard-tools

```bash
# 创建文件夹 (以管理员身份)
sudo mkdir /etc/wireguard

# 设置文件夹权限 (以管理员身份)
sudo chmod 777 /etc/wireguard

# 切入到创建的目录下
cd /etc/wireguard

# 生成公钥与私钥
wg genkey | tee privatekey | wg pubkey > publickey

# 创建虚拟网卡配置文件
touch wg0.conf

# 编辑虚拟网卡配置文件内容
vi wg0.conf
```

在wg0.conf文件中写入如下内容，需要自己修改文件内容。

```bash
[Interface]
Address = 10.0.0.7/24
ListenPort = 21363
PrivateKey = 客户端的私钥（刚刚生成的privatekey文件的内容）
DNS = 64.6.64.6
MTU = 1420

[Peer]
AllowedIPs = 0.0.0.0/0
Endpoint = 服务器的物理ip地址:41821
PersistentKeepalive = 25
PublicKey = 服务器的公钥(需要去服务器查看服务器的公钥)
```

## 服务器参数配置

除了客户端需要修改之后，还要将服务器网卡禁用，再修改服务器端的配置文件，在文件尾部加入当前客户端的信息。

```bash
[Peer]
PublicKey = 服务器的公钥（需要到服务里查看公钥）
AllowedIPs = 10.0.0.7/24
```

## Mac OS下启动客户端的网卡

```bash
# 启动网卡
wg-quick up wg0
```

其他命令:

```bash
# 其他命令
wg-quick down wg0 #停止服务
wg-quick strip wg0 #查看配置
wg-quick #查看所有支持的命令
```






