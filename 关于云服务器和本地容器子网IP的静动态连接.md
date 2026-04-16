# 关于云服务器和本地容器子网IP的静动态连接

> **部署日期**: 2026年4月14日 星期二
> **opencode上下文**: 本文件是opencode与用户共同部署的记录，opencode需要阅读此文件延续未完成的工作
> **服务器IP**: 101.133.135.3
> **SSH密钥**: /home/jj/Jiang.pem
> **SSH用户**: root

---

## 一、架构目标

```
用户请求 ──────────────────────────────────────────────────────────────────
    │                                                                      │
    ▼                                                                      │
┌────────────────────────────────────────────────────────────────────────┐  │
│                     阿里云服务器 (公网IP: 101.133.135.3)               │  │
│                                                                        │  │
│   ┌─────────────────┐     ┌─────────────────┐     ┌────────────────┐  │  │
│   │  Nginx (80)     │────▶│  静态服务       │     │ Docker Swarm   │  │  │
│   │  反向代理       │     │  static-server  │     │ Manager        │  │  │
│   │                 │     │  端口:8080      │     │                │  │  │
│   └────────┬────────┘     └─────────────────┘     └────────────────┘  │  │
│            │                    ▲                                       │  │
│            │                    │                                       │  │
│            ▼                    │                                       │  │
│   ┌─────────────────┐      │                                       │  │
│   │  动态请求转发    │──────┘                                       │  │
│   │  到VPN内网       │                                              │  │
│   └────────┬────────┘                                                 │  │
│            │                                                          │  │
│   ┌────────▼────────┐     ┌─────────────────┐                        │  │
│   │  WireGuard     │◀───▶│  Docker Overlay │                        │  │
│   │  10.0.0.1:51820│     │  app-network    │                        │  │
│   └────────┬────────┘     └─────────────────┘                        │  │
│            │                                                          │  │
└────────────┼──────────────────────────────────────────────────────────┘  │
             │ WireGuard VPN隧道 (10.0.0.0/24)                              │
             ▼                                                              │
┌────────────────────────────────────────────────────────────────────────────┐ │
│                         本地电脑                                           │ │
│                                                                            │ │
│   ┌─────────────────┐     ┌─────────────────┐                           │ │
│   │  WireGuard      │     │  Docker Swarm   │                           │ │
│   │  10.0.0.2      │◀───▶│  Worker         │                           │ │
│   └────────┬────────┘     └────────┬────────┘                           │ │
│            │                        │                                     │ │
│            │                        ▼                                     │ │
│            │               ┌─────────────────┐                            │ │
│            │               │  动态服务        │                            │ │
│            │               │  app-dynamic     │                            │ │
│            │               │  端口:3000       │                            │ │
│            │               └─────────────────┘                            │ │
│            │                                                              │ │
│   VPN已连接 ✅  |  Swarm已加入 ✅                                        │ │
└────────────────────────────────────────────────────────────────────────────┘ │
                                                                              │
                     用户只能看到 101.133.135.3，真实IP被隐藏                 ─┘
```

---

## 二、已完成步骤（带结果）

### ✅ 2.1 SSH连接配置

#### 执行命令
```bash
ssh -i /home/jj/Jiang.pem root@101.133.135.3
```

#### 报错1: Connection timeout
```
错误: Connection timed out during banner exchange
原因: 
  1. 阿里云安全组未开放22端口
  2. SSH服务未启动
  3. 网络延迟过高
排查命令:
  nc -zv 101.133.135.3 22        # 测试端口连通性
  telnet 101.133.135.3 22        # 测试SSH握手
排障思路:
  1. 登录阿里云控制台 → ECS → 安全组 → 入方向规则 → 添加TCP 22
  2. 通过VNC连接服务器执行: systemctl status ssh
  3. 等待安全组规则生效(通常1-2分钟)
```

#### 报错2: Permission denied
```
错误: Permission denied (publickey)
原因:
  1. 密钥文件权限不正确
  2. 用户名错误(不是root)
  3. 密钥对未绑定到ECS实例
排查命令:
  ls -la /home/jj/Jiang.pem      # 检查密钥权限(应为600)
  cat /home/jj/Jiang.pem         # 确认密钥格式正确
排障思路:
  1. chmod 600 /home/jj/Jiang.pem
  2. 尝试不同用户: ubuntu, root, admin
  3. 阿里云控制台 → 实例 → 密码/密钥 → 确认密钥已绑定
```

#### 报错3: Connection refused
```
错误: Connection refused
原因: SSH服务未运行
排查命令:
  nc -zv 101.133.135.3 22        # 端口是否开放
  telnet 101.133.135.3 22        # 是否能连接
排障思路:
  1. 通过VNC登录服务器
  2. systemctl start ssh
  3. systemctl enable ssh
```

**结果**: 连接成功 ✅

---

### ✅ 2.2 服务器环境确认

#### 执行命令
```bash
ssh -i Jiang.pem root@101.133.135.3 "uname -a && cat /etc/os-release && nproc && free -h && docker --version && nginx -v"
```

#### 报错: Command timed out
```
错误: SSH命令执行超时
原因: 服务器负载过高或网络问题
排查命令:
  ssh -v -i Jiang.pem root@101.133.135.3 "echo test"   # 简化测试
排障思路:
  1. 检查服务器CPU/内存: top, free -m
  2. 重启SSH服务: systemctl restart ssh
  3. 增加超时时间: ssh -o ConnectTimeout=60
```

#### 报错: command not found
```
错误: xxx: command not found
原因: 软件未安装
排查命令:
  which docker nginx wg         # 检查命令是否存在
  apt list --installed         # 列出已安装包
排障思路:
  Docker: apt update && apt install docker.io
  Nginx:  apt update && apt install nginx
  WireGuard: apt update && apt install wireguard
```

| 项目 | 值 |
|------|-----|
| 系统 | Ubuntu 24.04.2 LTS |
| Docker | 29.3.1 |
| Nginx | 1.24.0 |
| WireGuard | 已安装 |
| CPU | 2核 |
| 内存 | 1.6GB |

---

### ✅ 2.3 Docker Swarm初始化

#### 执行命令
```bash
ssh -i Jiang.pem root@101.133.135.3 "docker swarm init"
```

#### 报错1: node is already part of a swarm
```
错误: This node is already part of a swarm
原因: 服务器已经初始化过Swarm
排查命令:
  docker info | grep Swarm
  docker node ls
排障思路:
  1. 如果要重新初始化: docker swarm leave --force && docker swarm init
  2. 如果保留现有: 直接获取token使用
```

#### 报错2: could not choose an IP address
```
错误: could not choose an IP address error
原因: 服务器有多个网络接口，Docker不确定使用哪个IP
排查命令:
  ip addr show
  hostname -I
排障思路:
  明确指定IP地址:
  docker swarm init --advertise-addr 172.24.44.195
```

#### 报错3: port is already allocated
```
错误: Error starting daemon: port 2377 is already allocated
原因: 2377端口被占用
排查命令:
  ss -tlnp | grep 2377
  netstat -tulnp | grep 2377
排障思路:
  1. 停止占用进程: systemctl stop <service>
  2. 或更换端口: docker swarm init --listen-addr <IP>:2378
```

**结果**:
- Manager节点ID: `ukwgr0gh9fxs9z29ltn05o0z9`
- Join Token: `SWMTKN-1-5lwg7wgokmp4x5k13dz9w2l260uba1p4wt87zqwf0id66jif1t-af7ovpp0u8dq147dwq0iatfcj`
- 状态: `Swarm: active`

---

### ✅ 2.4 Overlay网络创建

#### 执行命令
```bash
ssh -i Jiang.pem root@101.133.135.3 "docker network create --driver=overlay --attachable app-network"
```

#### 报错1: This node is not a swarm manager
```
错误: Error response from daemon: This node is not a swarm manager
原因: 未初始化Swarm就创建overlay网络
排查命令:
  docker info | grep Swarm
排障思路:
  docker swarm init
```

#### 报错2: network with name already exists
```
错误: network with name app-network already exists
原因: 网络已存在
排查命令:
  docker network ls | grep app-network
排障思路:
  1. 使用现有网络
  2. 或删除重建: docker network rm app-network
```

#### 报错3: attachable标志缺失
```
错误: container cannot connect to network (非swarm服务)
原因: 创建overlay网络时未加--attachable标志
排查命令:
  docker network inspect app-network | grep attachable
排障思路:
  删除重建:
  docker network rm app-network
  docker network create --driver=overlay --attachable app-network
```

**结果**:
- 网络ID: `ld0ujh36qt60x63e78bxx7bu4`
- 类型: overlay (swarm)
- 状态: attachable (普通容器可连接)

---

### ✅ 2.5 WireGuard VPN配置

#### 执行命令
```bash
# 生成密钥
wg genkey | tee server_private.key | wg pubkey > server_public.key

# 服务端配置
cat > /etc/wireguard/wg0.conf << 'EOF'
[Interface]
PrivateKey = <服务端私钥>
Address = 10.0.0.1/24
ListenPort = 51820
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <客户端公钥>
AllowedIPs = 10.0.0.2/32
EOF

wg-quick up wg0
```

#### 报错1: RTNETLINK answers: File exists
```
错误: RTNETLINK answers: File exists
原因: wg0接口已存在
排查命令:
  ip link show wg0
  wg show
排障思路:
  sudo wg-quick down wg0
  sudo wg-quick up wg0
```

#### 报错2: Permissions 0644 for 'wg0.conf' are too open
```
错误: Permissions 0644 for '/etc/wireguard/wg0.conf' are too open
原因: 配置文件权限过于开放
排查命令:
  ls -la /etc/wireguard/wg0.conf
排障思路:
  chmod 600 /etc/wireguard/wg0.conf
```

#### 报错3: wg: command not found
```
错误: wg: command not found
原因: WireGuard未安装
排查命令:
  which wg
排障思路:
  Ubuntu: apt update && apt install wireguard
  CentOS: yum install epel-release && yum install wireguard-tools
```

#### 报错4: UDP 51820端口未开放
```
错误: 无法接收UDP包(无错误信息,但ping不通)
原因: 阿里云安全组未开放UDP 51820
排查命令:
  nc -vzu 101.133.135.3 51820
排障思路:
  阿里云控制台 → 安全组 → 入方向规则 → 添加UDP 51820
```

#### 报错5: VPN连接成功但100%丢包
```
错误: ping -c 3 10.0.0.1 → 3 packets transmitted, 3 received, 100% packet loss
原因: 公钥不匹配
排查思路:
  1. 检查服务器wg show:
     - 服务端peer的PublicKey应该是客户端的公钥
     - 服务端[Interface]的PublicKey应该是客户端[Peer]的PublicKey
  
  2. 检查客户端wg show:
     - 客户端[Interface]的PublicKey应该是服务端的公钥
  
  3. 常见错误: 客户端配置里的服务器公钥填错了
     正确: Sp3uYj6Mg8IcWRdHmHzs84Z6rvD7KYKRB1bZWLwUwRo=
     错误: ohunSsvCAcZd2+efNUaUjwhrC1iw+gNwm/93u5SP+BI=

  排障命令:
    服务器: wg show (查看peer的allowed ips是否有10.0.0.2)
    客户端: sudo wg show (查看是否有endpoint和transfer数据)
```

#### 报错6: DNS解析异常
```
错误: Warning: /etc/resolv.conf is not a symbolic link
原因: resolvconf配置问题(不影响VPN连接)
排查命令:
  cat /etc/resolv.conf
排障思路:
  可忽略,不影响使用
  或修复: ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
```

#### 报错7: iptables规则未生效
```
错误: VPN连接成功但服务器无法转发流量
原因: PostUp规则未执行或被覆盖
排查命令:
  iptables -L FORWARD -n | grep wg0
  iptables -t nat -L POSTROUTING -n | grep wg0
排障思路:
  手动添加规则:
  iptables -A FORWARD -i wg0 -j ACCEPT
  iptables -A FORWARD -o wg0 -j ACCEPT
  iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE
  
  检查IP转发:
  cat /proc/sys/net/ipv4/ip_forward (应为1)
  echo 1 > /proc/sys/net/ipv4/ip_forward
```

#### 服务端配置 (/etc/wireguard/wg0.conf)
```ini
[Interface]
PrivateKey = qPveyFyStsra6ZWysGIRRg9d21W2rrl5EyLcRkTWX08=
Address = 10.0.0.1/24
ListenPort = 51820
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = qodJ/NYcz4ZRfdYVbDiNQkrFConuQDCpGDPPIm0GrT0=
AllowedIPs = 10.0.0.2/32
```

#### 客户端配置 (/etc/wireguard/wg0.conf)
```ini
[Interface]
PrivateKey = EJmvOnVarCE0/AHQpEPGnh6hqEOy7eD+h7CjKR2VHUg=
Address = 10.0.0.2/24
DNS = 8.8.8.8

[Peer]
PublicKey = Sp3uYj6Mg8IcWRdHmHzs84Z6rvD7KYKRB1bZWLwUwRo=
Endpoint = 101.133.135.3:51820
AllowedIPs = 10.0.0.0/24
PersistentKeepalive = 25
```

#### 密钥对应关系
| 机器 | PrivateKey | PublicKey |
|------|------------|-----------|
| 服务器 | `qPveyFyStsra6ZWysGIRRg9d21W2rrl5EyLcRkTWX08=` | `Sp3uYj6Mg8IcWRdHmHzs84Z6rvD7KYKRB1bZWLwUwRo=` |
| 本地 | `EJmvOnVarCE0/AHQpEPGnh6hqEOy7eD+h7CjKR2VHUg=` | `qodJ/NYcz4ZRfdYVbDiNQkrFConuQDCpGDPPIm0GrT0=` |

#### VPN状态
- VPN连接: ✅ **成功**
- 延迟: ~70ms
- ping测试: `3 packets transmitted, 3 received, 0% packet loss`

---

### ✅ 2.6 本地加入Swarm

#### 执行命令
```bash
docker swarm join --token SWMTKN-1-5lwg7wgokmp4x5k13dz9w2l260uba1p4wt87zqwf0id66jif1t-af7ovpp0u8dq147dwq0iatfcj 10.0.0.1:2377
```

#### 报错1: This node is already part of a swarm
```
错误: This node is already part of a swarm
原因: 本地已加入其他Swarm或状态异常
排查命令:
  docker info | grep Swarm
  docker node ls
排障思路:
  docker swarm leave --force
  docker swarm join --token <token> 10.0.0.1:2377
```

#### 报错2: This node could not reach the advertised manager
```
错误: This node could not reach the advertised manager
原因: 
  1. VPN未连接
  2. 2377端口未开放
  3. manager地址不可达
排查命令:
  ping 10.0.0.1
  nc -zv 10.0.0.1 2377
排障思路:
  1. 确认VPN已连接: wg show
  2. 确认服务器2377端口开放
  3. 使用正确的manager地址
```

#### 报错3: manager not addressable
```
错误: manager not addressable
原因: 使用主机名而非IP地址
排查命令:
  hostname -I
排障思路:
  使用IP地址而非主机名:
  docker swarm join --token <token> 10.0.0.1:2377
```

#### 报错4: node is incompatible
```
错误: Error response from daemon: node is incompatible with existing node
原因: Docker版本差异过大
排查命令:
  docker version (本地和服务器对比)
排障思路:
  升级本地Docker版本与服务器一致
```

**结果**: `This node joined a swarm as a worker.`

---

### ✅ 2.7 服务器Nginx配置

#### 执行命令
```bash
scp -i Jiang.pem nginx.conf root@101.133.135.3:/tmp/nginx.conf
ssh -i Jiang.pem root@101.133.135.3 "cp /tmp/nginx.conf /etc/nginx/nginx.conf && nginx -t && systemctl restart nginx"
```

#### 报错1: nginx: [emerg] could not build optimal types_hash
```
错误: nginx: [emerg] could not build optimal types_hash
原因: types_hash_max_size配置问题
排查命令:
  cat /etc/nginx/nginx.conf | grep types_hash
排障思路:
  在http块添加: types_hash_max_size 2048;
```

#### 报错2: nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
```
错误: nginx: [emerg] bind() to 0.0.0.0:80 failed (98: Address already in use)
原因: 80端口被占用
排查命令:
  ss -tlnp | grep :80
  lsof -i :80
排障思路:
  1. 停止占用服务: systemctl stop <service>
  2. 或杀死进程: kill $(lsof -t -i:80)
  3. 常见占用: apache2, nginx(其他实例)
```

#### 报错3: nginx: [emerg] unexpected "}"
```
错误: nginx: [emerg] unexpected "}" in...
原因: 配置文件语法错误
排查命令:
  nginx -t -c /path/to/nginx.conf
排障思路:
  1. 检查括号匹配
  2. 检查引号配对
  3. 检查分号结尾
  4. 使用nginx -T查看完整错误
```

#### 报错4: upstream timed out
```
错误: upstream timed out (110: Connection timed out)
原因: 后端服务不可达
排查命令:
  curl -v http://localhost:8080
  curl -v http://172.20.0.2:3000
排障思路:
  1. 检查服务是否运行: docker service ls
  2. 检查网络连通性: ping 172.20.0.2
  3. 检查端口是否监听: ss -tlnp | grep 8080
```

**配置文件**: `/etc/nginx/nginx.conf`

**关键配置**:
```nginx
upstream dynamic_backend {
    server 172.20.0.2:3000;
    keepalive 32;
}

upstream static_backend {
    server localhost:8080;
    keepalive 16;
}

server {
    listen 80;
    server_name 101.133.135.3;
    
    # 静态资源
    location /static/ {
        proxy_pass http://static_backend/static/;
        expires 30d;
    }
    
    # 动态API - 隐藏真实IP
    location /api/ {
        proxy_pass http://dynamic_backend/api/;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

**结果**: `nginx -t` ✅ `systemctl restart nginx` ✅

---

### ✅ 2.8 创建静态目录

#### 执行命令
```bash
ssh -i Jiang.pem root@101.133.135.3 "mkdir -p /var/www/static"
```

#### 报错: Permission denied
```
错误: Permission denied
原因: 没有写权限
排查命令:
  ls -la /var/www/
排障思路:
  使用sudo或切换到有权限的用户
  sudo mkdir -p /var/www/static
```

**结果**: 目录已创建

---

## 三、未完成步骤

### ❌ 3.1 部署静态服务（服务器）
**状态**: 未执行
**需要**:
1. 创建静态测试文件
2. 部署static-server服务
3. 验证服务运行

### ❌ 3.2 拉取Docker镜像（本地）
**状态**: 未执行
**需要**: 在本地拉取 `nginx:alpine` 镜像

### ❌ 3.3 部署动态服务
**状态**: 服务已创建但失败
**原因**: 本地没有镜像，导致worker节点无法启动容器
**服务ID**: `0mi8uibq168v4i33blfp9y651`
**当前状态**: `0/1` (准备中但失败)
**错误信息**: `"No such image: nginx:alpine"`

### ❌ 3.4 端到端测试
**状态**: 未执行
**需要**:
1. 测试静态服务访问
2. 测试动态服务访问
3. 验证IP隐藏效果

---

## 四、当前状态快照

### 服务器端 - 服务状态
```bash
$ ssh -i Jiang.pem root@101.133.135.3 "docker service ls"
ID             NAME          MODE         REPLICAS   IMAGE
0mi8uibq168v   app-dynamic   replicated   0/1        nginx:alpine
```
**问题**: app-dynamic服务处于0/1状态，因为worker节点没有镜像

### 服务器端 - 服务详细状态
```bash
$ ssh -i Jiang.pem root@101.133.135.3 "docker service ps app-dynamic"
ID             NAME                IMAGE          NODE                            DESIRED STATE   CURRENT STATE           ERROR
e2vnjpqv10a8   app-dynamic.1       nginx:alpine   jj-Jiaolong16K-Series-GM6BG0Q   Running         Preparing 52 seconds ago
wnw8il77edug    \_ app-dynamic.1   nginx:alpine   jj-Jiaolong16K-Series-GM6BG0Q   Shutdown        Rejected 52 seconds ago   "No such image: nginx:alpine@s…"
```
**说明**: 容器被拒绝启动，原因是worker节点（本地电脑）没有`nginx:alpine`镜像

### 本地端 - VPN状态
```bash
$ sudo wg show
interface: wg0
  public key: qodJ/NYcz4ZRfdYVbDiNQkrFConuQDCpGDPPIm0GrT0=
  peer: Sp3uYj6Mg8IcWRdHmHzs84Z6rvD7KYKRB1bZWLwUwRo=
  endpoint: 101.133.135.3:51820
```
**状态**: VPN已连接 ✅

### Swarm集群状态
```bash
$ ssh -i Jiang.pem root@101.133.135.3 "docker node ls"
ID                            HOSTNAME                        STATUS    AVAILABILITY   MANAGER STATUS
ukwgr0gh9fxs9z29ltn05o0z9 *   iZuf6ei4xwgixld5kgkusxZ        Ready     Active         Leader
yyb5w6d4r1xz9k22n1xq8w5z3     jj-Jiaolong16K-Series-GM6BG0Q   Ready     Active
```
**状态**: 2个节点 ✅

---

## 五、继续部署（opencode请按此执行）

### 第一步：本地拉取镜像（必须先执行）

#### 报错: docker: command not found
```
错误: docker: command not found
原因: Docker未安装
排查命令:
  which docker
排障思路:
  Ubuntu: curl -fsSL https://get.docker.com | sh
  或: apt update && apt install docker.io
```

#### 报错: permission denied while trying to connect to the Docker daemon
```
错误: permission denied while trying to connect to the Docker daemon socket
原因: 当前用户不在docker组
排查命令:
  groups $USER
排障思路:
  sudo usermod -aG docker $USER
  newgrp docker
  或每次命令前加sudo
```

**方式1 - 用户手动在本地终端执行**:
```bash
docker pull nginx:alpine
```

**方式2 - opencode通过SSH执行（如果本地开放了SSH）**:
```bash
# 注意：需要知道本地SSH的用户名和密码/密钥
ssh -i /home/jj/Jiang.pem jj@192.168.62.72 "docker pull nginx:alpine"
```

### 第二步：服务器部署静态服务

#### 报错1: network app-network not found
```
错误: network app-network declared as external, but could not be found
原因: 网络未创建
排查命令:
  docker network ls | grep app-network
排障思路:
  docker network create --driver=overlay --attachable app-network
```

#### 报错2: port is already allocated
```
错误: bind: address already in use
原因: 8080端口被占用
排查命令:
  ss -tlnp | grep 8080
排障思路:
  1. 更换端口: --publish 8081:80
  2. 或停止占用服务
```

#### 报错3: Volume mount failed
```
错误: volume mount failed: path not exist
原因: 挂载的目录不存在
排查命令:
  ls -la /var/www/static
排障思路:
  mkdir -p /var/www/static
  chown -R 777 /var/www/static
```

```bash
ssh -i /home/jj/Jiang.pem -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null root@101.133.135.3 '
# 创建静态测试文件
echo "<h1>静态页面 - 来自阿里云服务器</h1>" > /var/www/static/index.html

# 部署静态服务
docker service create \
  --name static-server \
  --network app-network \
  --publish 8080:80 \
  --mount type=bind,source=/var/www/static,target=/usr/share/nginx/html,readonly=true \
  --constraint '"'"'node.role==manager'"'"' \
  nginx:alpine
'
```

### 第三步：检查服务状态

#### 报错: service replicas 0/1
```
错误: app-dynamic 0/1, static-server 0/1
原因: 镜像未拉取或节点不可达
排查命令:
  docker service ps <service-name>
  docker node ls
排障思路:
  1. 查看详细错误: docker service ps <service-name> --no-trunc
  2. 检查节点状态: docker node ls
  3. 确认worker节点可达
  4. 拉取镜像: docker pull <image>
```

```bash
ssh -i /home/jj/Jiang.pem -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null root@101.133.135.3 "
sleep 15
docker service ls
docker service ps app-dynamic
docker service ps static-server
"
```

### 第四步：端到端测试

#### 报错1: curl: (7) Failed to connect
```
错误: Failed to connect to 101.133.135.3 port 80: Connection refused
原因: Nginx未运行或防火墙阻断
排查命令:
  ssh root@101.133.135.3 "systemctl status nginx"
  ss -tlnp | grep :80
排障思路:
  1. systemctl start nginx
  2. 检查阿里云安全组80端口
```

#### 报错2: 502 Bad Gateway
```
错误: 502 Bad Gateway
原因: upstream服务不可达
排查命令:
  ssh root@101.133.135.3 "curl http://localhost:8080"
  ssh root@101.133.135.3 "curl http://app-dynamic"
排障思路:
  1. 检查后端服务是否运行
  2. 检查网络连通性
  3. 检查服务日志: docker logs <container>
```

#### 报错3: 504 Gateway Timeout
```
错误: 504 Gateway Timeout
原因: upstream响应超时
排查命令:
  curl -v --max-time 10 http://upstream
排障思路:
  1. 检查upstream服务性能
  2. 增加proxy超时时间
  3. 检查网络延迟
```

```bash
# 测试静态服务（服务器内部）
ssh -i /home/jj/Jiang.pem -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null root@101.133.135.3 "curl -s http://localhost:8080/"

# 测试动态服务（通过Swarm网络）
ssh -i /home/jj/Jiang.pem -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null root@101.133.135.3 "curl -s http://app-dynamic/"

# 测试公网访问
curl -s http://101.133.135.3/ -I || curl -s http://101.133.135.3/
```

---

## 六、关键问题说明

### 为什么动态服务创建失败？
**原因**: Docker Swarm的工作机制是当服务调度到某个节点时，该节点需要有对应的镜像。本地电脑（worker节点）没有拉取`nginx:alpine`镜像，导致容器无法启动。

**解决方案**: 在本地电脑执行 `docker pull nginx:alpine` 即可。服务会自动重试启动。

### 为什么用docker service而不是docker-compose？
**原因**: Docker Swarm的service可以在集群中跨节点调度。通过`--constraint 'node.role==worker'`可以确保动态服务只运行在本地电脑（worker节点）上。

---

## 七、完整排障手册

### SSH连接排障流程
```
1. 测试端口连通性
   nc -zv 101.133.135.3 22
   
2. 检查SSH服务状态(通过VNC)
   systemctl status ssh
   
3. 检查安全组规则
   阿里云控制台 → ECS → 安全组
   
4. 验证密钥权限
   ls -la Jiang.pem (应为600)
   
5. 尝试不同用户名
   ubuntu, root, admin
```

### WireGuard排障流程
```
1. 检查服务端监听
   wg show
   
2. 检查客户端连接状态
   sudo wg show
   
3. 验证公钥匹配
   - 服务端[Interface]PublicKey = 客户端[Peer]PublicKey
   - 客户端[Interface]PublicKey = 服务端[Peer]PublicKey
   
4. 检查防火墙规则
   iptables -L FORWARD -n | grep wg0
   
5. 测试UDP端口
   nc -vzu 101.133.135.3 51820
   
6. 检查IP转发
   cat /proc/sys/net/ipv4/ip_forward (应为1)
```

### Docker Swarm排障流程
```
1. 检查节点状态
   docker node ls
   
2. 检查Swarm状态
   docker info | grep Swarm
   
3. 检查服务状态
   docker service ls
   docker service ps <name>
   
4. 查看详细日志
   docker service ps <name> --no-trunc
   docker logs <container-id>
   
5. 检查网络连通性
   docker network inspect app-network
   
6. 重新加入(如果异常)
   docker swarm leave --force
   docker swarm join --token <token> <manager-ip>:2377
```

### Nginx排障流程
```
1. 测试配置
   nginx -t
   
2. 检查服务状态
   systemctl status nginx
   
3. 查看错误日志
   tail -20 /var/log/nginx/error.log
   
4. 测试upstream连通性
   curl http://localhost:8080
   curl http://<upstream-ip>:<port>
   
5. 检查端口占用
   ss -tlnp | grep :80
   
6. 重启服务
   systemctl restart nginx
```

---

## 八、安全组配置要求

| 协议 | 端口 | 来源 | 说明 |
|------|------|------|------|
| TCP | 22 | 0.0.0.0/0 | SSH |
| TCP | 80 | 0.0.0.0/0 | HTTP |
| TCP | 443 | 0.0.0.0/0 | HTTPS |
| TCP | 2377 | 0.0.0.0/0 | Docker Swarm |
| UDP | 51820 | 0.0.0.0/0 | WireGuard VPN |

---

## 九、关键文件路径

| 用途 | 路径 |
|------|------|
| SSH密钥 | `/home/jj/Jiang.pem` |
| 服务器WireGuard | `/etc/wireguard/wg0.conf` |
| 本地WireGuard | `/etc/wireguard/wg0.conf` |
| 服务器Nginx | `/etc/nginx/nginx.conf` |
| 静态文件目录 | `/var/www/static/` |
| Docker网络 | `app-network` |

---

## 十、部署完成后的验证清单

- [ ] `ping 10.0.0.1` 成功（VPN连通）
- [ ] `docker node ls` 显示2个节点
- [ ] `docker service ls` 显示2个服务，状态为1/1
- [ ] `curl http://localhost:8080/` 返回静态页面
- [ ] `curl http://app-dynamic/` 返回动态页面（通过Swarm网络）
- [ ] `curl http://101.133.135.3/` 返回页面（公网访问）
- [ ] 检查Nginx日志无异常

---

## 十一、下一步扩展

部署完成后可添加：
1. HTTPS证书（Let's Encrypt）
2. 真实的动态应用（替换nginx测试页面）
3. 负载均衡配置
4. 监控告警
5. CI/CD自动化部署

---

## 十二、排障记录（2026-04-15 opencode续）

### ❌ 12.1 服务调度失败 - 节点不可用

#### 故障现象
```
app-dynamic 服务状态: 0/1 或 ERROR
错误信息: "node ys8xwzb5pccw0hoy61e9sw63o is not available"
```

#### 排查过程

**步骤1：检查服务状态**
```bash
ssh -i Jiang.pem root@101.133.135.3 "docker service ls"
```
```
ID             NAME          MODE         REPLICAS   IMAGE
imfd9jgr6zb5   app-dynamic   replicated   0/1        nginx:alpine
```

**步骤2：检查服务详情**
```bash
ssh -i Jiang.pem root@101.133.135.3 "docker service ps app-dynamic --no-trunc"
```
```
ERROR: "no suitable node (1 node not available for new tasks)"
```

**步骤3：检查节点状态**
```bash
ssh -i Jiang.pem root@101.133.135.3 "docker node ls"
```
```
ID                            HOSTNAME                        STATUS    AVAILABILITY
ukwgr0gh9fxs9z29ltn05o0z9 *   iZuf6ei4xwgixld5kgkusxZ        Ready     Active
ys8xwzb5pccw0hoy61e9sw63o     jj-Jiaolong16K-Series-GM6BG0Q   Down      Active
```
**发现**: 本地节点状态显示 `Down`

#### 故障原因

| 检查项 | 结果 | 分析 |
|--------|------|------|
| 本地 Swarm 状态 | active | 本地认为自己在线 |
| 本地 Node Address | 10.0.0.2 | VPN IP |
| Manager Addresses | 172.24.44.195:2377 | 服务器内网IP |
| 内网IP可达性 | 不可达 | 不在同一网络 |
| VPN IP可达性 | 可达 | 同一VPN网络 |

**根本原因**: 本地节点加入 Swarm 时使用服务器内网IP (172.24.44.195)，但本地无法访问该IP，只能通过VPN IP (10.0.0.1) 访问。

#### 解决方案

**方案A：重新加入 Swarm（推荐）**

```bash
# 1. 本地节点离开 Swarm
docker swarm leave --force

# 2. 获取新的 worker token
ssh -i Jiang.pem root@101.133.135.3 "docker swarm join-token worker -q"

# 3. 本地通过 VPN IP 重新加入
docker swarm join --token <token> 10.0.0.1:2377
```

**方案B：修改服务器 Swarm 监听地址**

```bash
# 服务器端重新初始化（会丢失服务）
ssh -i Jiang.pem root@101.133.135.3 "docker swarm init --advertise-addr 10.0.0.1"
```

#### 排查命令记录

```bash
# 测试连通性
timeout 2 bash -c "cat < /dev/null > /dev/tcp/172.24.44.195/2377"
# 结果: 不可达

timeout 2 bash -c "cat < /dev/null > /dev/tcp/10.0.0.1/2377"
# 结果: 可达

# 服务器端口监听
ssh -i Jiang.pem root@101.133.135.3 "ss -tlnp | grep 2377"
# 结果: *:2377 (监听所有接口)

# 服务器到本地连通性
ssh -i Jiang.pem root@101.133.135.3 "ping -c 1 10.0.0.2"
# 结果: 0% packet loss
```

#### 报错1: 本地节点离开失败
```
错误: Error response from daemon: You are attempting to leave the Swarm as a manager.
原因: 本地节点是 manager 角色
解决: docker swarm leave --force (强制离开)
```

#### 报错2: 重新加入后仍是 Down
```
原因: 服务器防火墙阻止了 2377 端口的 VPN 流量
解决: 
  # 服务器端
  iptables -I INPUT -p tcp --dport 2377 -s 10.0.0.0/24 -j ACCEPT
  
  # 阿里云安全组也要开放 2377 TCP
```

#### 报错3: join-token 获取失败
```
错误: error during connect
原因: Manager 节点不可达
解决: 检查 VPN 连接和防火墙
```

**结果**: 待执行修复后更新

---

### ✅ 12.2 其他正常项

| 检查项 | 状态 | 说明 |
|--------|------|------|
| VPN 连接 | ✅ | 10.0.0.1 ↔ 10.0.0.2 互通 |
| 服务器 SSH | ✅ | 22端口可访问 |
| 服务器 2377 | ✅ | 监听所有接口 |
| 本地镜像 | ✅ | nginx:alpine 已存在 |
| app-network | ✅ | 10.0.1.0/24 已创建 |

---

### 📋 下一步操作清单

- [ ] 1. 本地节点离开 Swarm
- [ ] 2. 获取新的 worker token
- [ ] 3. 本地通过 VPN IP (10.0.0.1:2377) 重新加入
- [ ] 4. 验证节点状态变为 Ready
- [ ] 5. 检查服务自动恢复
- [ ] 6. 测试端到端访问

---

## 十三、排障续记（opencode继续排查）

### 排查时间：2026-04-15

#### 问题链条分析

```
app-dynamic 服务 0/1
    ↓
本地节点 Down
    ↓
排查发现：节点通过内网IP加入，服务器无法访问
    ↓
解决：重新通过VPN IP加入
    ↓
加入超时
    ↓
排查发现：VPN连接已断开
```

#### 检查记录

**步骤1：让本地节点离开 Swarm**
```bash
docker swarm leave --force
# 结果: ✅ 成功
```

**步骤2：获取 token**
```bash
ssh -i Jiang.pem root@101.133.135.3 "docker swarm join-token worker -q"
# 结果: ✅ SWMTKN-1-5lwg7wgokmp4x5k13dz9w2l260uba1p4wt87zqwf0id66jif1t-af7ovpp0u8dq147dwq0iatfcj
```

**步骤3：尝试通过 VPN IP 加入**
```bash
docker swarm join --token <token> 10.0.0.1:2377
# 结果: ❌ 超时 - context deadline exceeded
```

**步骤4：检查 Swarm 配置**
```bash
# 服务器端 Swarm Manager Address
docker node inspect self --pretty
# 发现: Address: 172.24.44.195 (内网IP)
```

**步骤5：检查 VPN 状态**
```bash
nmcli device status | grep wg
# 结果: wg0 已断开

ss -ulnp | grep 51820
# 结果: 51820 未监听
```

#### ❌ 根本原因

| 问题 | 状态 | 说明 |
|------|------|------|
| 本地 VPN 连接 | ❌ 断开 | 无法通过 VPN 连接到服务器 |
| 服务器 Swarm | ✅ 正常 | Manager 在 172.24.44.195:2377 |
| 节点通信 | ❌ 失败 | 服务器无法访问 10.0.0.2 |

**问题**：本地 WireGuard VPN 连接已断开，导致无法通过 VPN IP 加入 Swarm。

#### 需要手动操作

用户需要执行以下操作：

```bash
# 1. 重新连接 VPN
sudo wg-quick up wg0

# 2. 验证 VPN 连接
ping -c 2 10.0.0.1

# 3. 确认连接
nmcli device status | grep wg

# 4. 验证 UDP 51820
ss -ulnp | grep 51820

# 5. 重新加入 Swarm
docker swarm leave --force
docker swarm join --token SWMTKN-1-5lwg7wgokmp4x5k13dz9w2l260uba1p4wt87zqwf0id66jif1t-af7ovpp0u8dq147dwq0iatfcj 10.0.0.1:2377

# 6. 验证节点状态
ssh -i Jiang.pem root@101.133.135.3 "docker node ls"
```

#### 后续步骤

VPN 连接恢复后：

```bash
# 1. 确认节点在线
ssh -i Jiang.pem root@101.133.135.3 "docker node ls"
# 应显示: jj-Jiaolong16K-Series-GM6BG0Q Ready Active

# 2. 检查服务状态
ssh -i Jiang.pem root@101.133.135.3 "docker service ls"
# 应显示: app-dynamic 1/1

# 3. 端到端测试
curl http://101.133.135.3/
```

---

## 十四、排障续记（VPN修复成功，Overlay 待修复）

### 时间：2026-04-15

#### 已修复问题

| 问题 | 状态 | 说明 |
|------|------|------|
| VPN 连接 | ✅ | 公钥配置错误已修复 |
| Swarm 加入 | ✅ | 节点已重新加入 |
| 服务恢复 | ✅ | app-dynamic 1/1 Running |

#### 修复步骤

**1. 重新创建 VPN 连接**
```bash
# 导入 WG 配置
nmcli connection import type wireguard file /tmp/wg0.conf

# 启动连接
nmcli connection up wg0
```

**2. 修复服务器 WG peer 公钥**
```bash
# 服务器原配置错误
PublicKey = ohunSsvCAcZd2+efNUaUjwhrC1iw+gNwm/93u5SP+BI=  # 错误

# 修正为本地公钥
PublicKey = qodJ/NYcz4ZRfdYVbDiNQkrFConuQDCpGDPPIm0GrT0=  # 正确
```

**3. 重新加入 Swarm**
```bash
docker swarm leave --force
docker swarm join --token SWMTKN-1-5lwg7wgokmp4x5k13dz9w2l260uba1p4wt87zqwf0id66jif1t-af7ovpp0u8dq147dwq0iatfcj 10.0.0.1:2377
```

#### 当前问题

| 测试 | 结果 | 说明 |
|------|------|------|
| VPN ping | ✅ | 0% packet loss |
| 服务器→本地 10.0.0.2 | ✅ | 可达 |
| 服务器→容器 10.0.1.56 | ❌ | Destination Host Unreachable |
| 容器运行 | ✅ | app-dynamic Running |

**问题**：overlay 网络 vxlan 隧道未正确建立

**需要手动操作**：
```bash
sudo systemctl restart docker
```

重启后验证：
```bash
# 检查容器是否恢复
docker ps | grep app-dynamic

# 测试服务访问
curl http://101.133.135.3:3000
```

---

## 十五、测试结果总结（2026-04-15）

### ✅ 最终测试结果

| 测试项 | 状态 | 说明 |
|--------|------|------|
| VPN 连接 | ✅ | 10.0.0.1 ↔ 10.0.0.2 互通 |
| Swarm 加入 | ✅ | 本地节点 Ready |
| overlay 网络 | ⚠️ | 跨节点通信有问题 |
| 服务器直接部署 | ✅ | 服务正常运行 |
| 公网访问 | ✅ | http://101.133.135.3:3000 正常 |

### 当前架构

```
公网用户 → 101.133.135.3:3000 → 服务器容器 (bridge网络)
```

### 发现的问题

**Docker Swarm overlay 网络跨节点通信问题**：
- Swarm 服务调度到本地 worker 节点后，ingress 网络未正确暴露端口
- 服务器无法通过 overlay 网络 (10.0.1.0/24) 到达本地容器
- 可能原因：VPN 环境下 overlay VXLAN 隧道配置问题

**临时解决方案**：
在服务器上直接运行容器（不使用 Swarm），绕过 overlay 网络问题：
```bash
docker run -d --name app-dynamic-server \
  --network bridge \
  -p 3000:80 \
  nginx:alpine
```

### 后续建议

1. **修复 overlay 网络**：调查 Docker Swarm 在 VPN 环境下的 overlay 网络配置
2. **使用不同 VPN 方案**：考虑使用 Docker in Docker 或其他方案
3. **保持当前方案**：服务器直接部署 + 本地容器通过 VPN 通信

---

## 十六、Overlay 网络重建（方案一执行中）

### 时间：2026-04-15

### 执行步骤

#### 第一步：服务器端清理旧网络和服务

**操作命令：**
```bash
# 查看当前状态
ssh -i Jiang.pem root@101.133.135.3 "docker node ls"
ssh -i Jiang.pem root@101.133.135.3 "docker network ls"

# 清理 Down 节点
ssh -i Jiang.pem root@101.133.135.3 "docker node rm <node-id>"

# 清理 overlay 网络
ssh -i Jiang.pem root@101.133.135.3 "docker network rm app-network"
ssh -i Jiang.pem root@101.133.135.3 "docker network rm ingress"
```

**报错1：ingress 网络删除警告**
```
WARNING! Before removing the routing-mesh network, make sure all the nodes 
in your swarm run the same docker engine version.
```
**原因**：Swarm 版本不一致警告
**解决**：可忽略，继续执行

**结果：**
- ✅ 旧节点已清理
- ✅ app-network 已删除
- ✅ ingress 已删除

---

#### 第二步：重新初始化 Swarm（使用 VPN IP）

**操作命令：**
```bash
# 离开现有 Swarm
ssh -i Jiang.pem root@101.133.135.3 "docker swarm leave --force"

# 重新初始化，使用 VPN IP
ssh -i Jiang.pem root@101.133.135.3 "docker swarm init --advertise-addr 10.0.0.1"
```

**报错1：Swarm 已初始化**
```
Error response from daemon: This node is already part of a swarm
```
**原因**：之前已初始化过
**解决**：先执行 leave 再 init

**报错2：无法选择 IP 地址**
```
could not choose an IP address error
```
**原因**：服务器有多个网络接口
**解决**：使用 --advertise-addr 明确指定 IP

**结果：**
- ✅ Swarm 初始化成功
- ✅ 新 Token: SWMTKN-1-13abdll2d9zhzz3ujylm7mi7w0jgmwfclxynp6h4r5wcldxitm-aagsn93vz2is84f5yjlt2v5tj
- ✅ Node Address: 10.0.0.1

---

#### 第三步：本地重新加入 Swarm

**操作命令：**
```bash
# 本地离开
docker swarm leave --force

# 获取新 token
ssh -i Jiang.pem root@101.133.135.3 "docker swarm join-token worker -q"

# 加入
docker swarm join --token <token> 10.0.0.1:2377
```

**报错1：加入超时**
```
Error response from daemon: Timeout was reached before node joined
context deadline exceeded while waiting for connections to become ready
```
**原因**：Docker daemon 未监听 TCP 2377 端口
**排查命令：**
```bash
ss -tlnp | grep 2377  # 本地检查
ssh root@服务器 "nc -zv 10.0.0.2 2377"  # 服务器测试连接
```
**排障结果：**
- VPN 连接: ✅ 正常
- Ping: ✅ 0% packet loss
- 2377 端口: ❌ Connection refused

**根本原因：**
Docker daemon 默认只监听 Unix socket (`/var/run/docker.sock`)，不监听 TCP 端口。Swarm 节点间通信需要 TCP 端口。

---

#### 第四步：配置本地 Docker 监听 TCP（待完成）

**需要在本地执行：**

```bash
# 1. 创建配置
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json << 'EOF'
{
  "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2375"]
}

---

## 十六、Overlay 网络重建（方案一执行中）

### 时间：2026-04-15

### 执行步骤记录

#### 第一步：服务器端清理旧网络和服务
- 清理 Down 节点: ✅ 完成
- 删除 app-network: ✅ 完成
- 删除 ingress: ✅ 完成

#### 第二步：重新初始化 Swarm（使用 VPN IP）
- 执行: docker swarm init --advertise-addr 10.0.0.1
- 结果: ✅ Swarm 初始化成功
- 新 Token: SWMTKN-1-13abdll2d9zhzz3ujylm7mi7w0jgmwfclxynp6h4r5wcldxitm-aagsn93vz2is84f5yjlt2v5tj
- Node Address: ✅ 10.0.0.1

#### 第三步：本地重新加入 Swarm
- 本地离开 Swarm: ✅ 完成
- 获取新 token: ✅ 完成
- 加入 Swarm: ⏳ 超时失败

**问题排查：**
| 检查项 | 结果 |
|--------|------|
| VPN 连接 | ✅ 正常 |
| Ping 10.0.0.2 | ✅ 0% packet loss |
| 服务器连接本地 2377 | ❌ Connection refused |

**根本原因：**
Docker daemon 默认只监听 Unix socket，不监听 TCP 端口。Swarm 节点间通信需要 TCP。

---

### 解决方案

用户需要在本地执行以下命令（需要 sudo 密码）：

```bash
# 1. 创建 Docker 配置目录
sudo mkdir -p /etc/docker

# 2. 创建 daemon.json 配置
cat << 'EOF' | sudo tee /etc/docker/daemon.json
{
  "hosts": ["unix:///var/run/docker.sock", "tcp://0.0.0.0:2375"]
}
EOF

# 3. 重启 Docker 服务
sudo systemctl restart docker

# 4. 验证端口监听
ss -tlnp | grep 2375
```

执行完成后告诉 opencode 继续验证。

---

### 当前架构状态图

```
重建前:                    重建后:
服务器内网IP(172.24...)    服务器VPN IP(10.0.0.1)
        ↓                         ↓
   Swarm Manager              Swarm Manager
        ↓                         ↓
   节点通信失败               节点通信待验证

目标:
服务器(10.0.0.1) ←TCP2377→ 本地(10.0.0.2)
```

---

## 十七、项目学习记录（2026-04-15）

### 今日学习内容

#### 第一课：混合云架构原理

**为什么要学这个？**
```
传统架构：所有服务在一个机房
混合云架构：服务分布在云端和本地
- 云端：稳定、可弹性扩展
- 本地：GPU、高性能计算资源
- 需要用 VPN 把它们连成一个虚拟网络
```

**VPN 的核心作用**
```
两个节点需要直接通信，但不在同一网络
→ VPN = 在公共网络上建立加密隧道
→ 把两台机器拉进同一个私有网络
```

**WireGuard 为什么适合混合云？**
```
| 方案   | 复杂度 | 性能 | 适合场景        |
|--------|--------|------|-----------------|
| OpenVPN | 高     | 中   | 企业内网        |
| IPSec  | 高     | 中   | site-to-site   |
| WireGuard| 低    | 高   | 云+本地混合 ✅  |
```

**Docker Overlay 网络中的 VXLAN**
```
VXLAN = MAC in UDP = 封装协议
原始包: [MAC|IP|数据]
封装后: [外层IP|UDP|VXLAN头|MAC|IP|数据]
像快递：物品装进箱子，箱子再装进集装箱运输
```

**为什么 Docker 要监听 TCP？**
```
默认：Docker 只监听 Unix socket (/var/run/docker.sock)
问题：Swarm 节点间需要通过 TCP 通信
解决：配置 Docker 监听 TCP 端口
```

---

### 今日已完成任务

| 任务 | 状态 | 说明 |
|------|------|------|
| 架构原理学习 | ✅ | 理解 VPN、Overlay、VXLAN |
| 现有架构测试 | ✅ | 测试 Nginx 和静态服务 |

---

### 今日测试结果

| 测试项 | 结果 |
|--------|------|
| VPN 连接 | ✅ 正常 |
| Nginx 反向代理 | ✅ 正常 |
| 静态服务 (8080) | ✅ 已修复并正常运行 |
| 动态服务 | ⚠️ 在服务器 :3000，未在本地 |

---

### 今日排障记录

**问题**：Nginx 502 Bad Gateway

**排查过程**：
```bash
# 1. 查看 Nginx 错误日志
tail /var/log/nginx/error.log
# 发现: connect() failed (111: Connection refused) while connecting to upstream

# 2. 检查后端端口
ss -tlnp | grep 8080
# 发现: 无输出，端口未监听

# 3. 检查容器状态
docker ps --filter name=static-server
# 发现: Exited (0) 5 hours ago

# 4. 重启容器
docker restart static-server
```

**解决方案**：重启 static-server 容器

---

## 十八、项目待完成目标

### 最终目标
```
用户 → 阿里云 Nginx → 静态服务 (服务器)
                   → 动态服务 (本地容器通过 VPN)
```

### 当前架构
```
┌─────────────────┐         ┌─────────────────┐
│   阿里云服务器   │   VPN   │    本地机器     │
│   101.133.135.3 │◄──────►│   10.0.0.2      │
│                 │         │                 │
│  ┌───────────┐  │         │  ┌───────────┐  │
│  │  Nginx    │  │         │  │ Docker    │  │
│  │  :80      │  │         │  │ (未加入)  │  │
│  └───────────┘  │         │  └───────────┘  │
│  ┌───────────┐  │         │                 │
│  │ static    │  │         │                 │
│  │ :8080 ✅  │  │         │                 │
│  └───────────┘  │         │                 │
│  ┌───────────┐  │         │                 │
│  │ dynamic   │  │         │                 │
│  │ :3000 ✅  │  │         │                 │
│  └───────────┘  │         │                 │
└─────────────────┘         └─────────────────┘
```

### 待完成项

| 目标 | 优先级 | 说明 |
|------|--------|------|
| 本地 Docker 加入 Swarm | 高 | 核心网络打通 |
| 动态服务迁移到本地 | 中 | 实现混合云 |
| Ingress 网络修复 | 高 | 服务可被公网访问 |

---

## 十九、SRE 实习第二天任务预告

### 明日目标
继续完成"混合云容器子网打通"项目

### 核心契约提醒

**明天的 Code 必须遵守：**

1. **严禁自动执行**
   - 每一步必须由你（实习生）动手
   - Code 只负责指令和解释

2. **深度教学**
   - 每操作前，必须解释"为什么要这么做"
   - 说明"大厂生产环境中的目的是什么"

3. **障碍预警**
   - 执行前列出 2-3 个新手最容易踩的坑
   - 说明高级工程师的排障思路

4. **故障处理策略**
   - 遇到报错，禁止直接给修正代码
   - 引导通过 log、status、tcpdump 等手段自己找到原因

5. **强制记录**
   - 每阶段结束，记录到 /home/jj/关于云服务器和本地容器子网IP的静动态连接.md

---

### 明日学习路径

```
Day 1 ✅: 架构原理 + 现有架构测试
Day 2 📋: Docker Swarm 跨节点通信修复
         ├─ 配置 Docker TCP 监听
         ├─ 重启 Swarm
         ├─ 创建 Overlay 网络
         └─ 部署服务到本地节点
```

### 明日第一课预告

**主题**：Docker Swarm 节点通信原理

**内容**：
- 为什么 Swarm 需要 TCP 2377 端口？
- 如何安全地配置 Docker TCP 监听？
- systemd override vs daemon.json 的区别

---

## 二十、重要命令速查

### 服务器操作
```bash
# SSH 连接
ssh -i Jiang.pem root@101.133.135.3

# 查看容器
docker ps -a

# 查看日志
docker logs <容器名>

# 查看端口
ss -tlnp | grep <端口>

# 查看 Nginx 错误
tail /var/log/nginx/error.log
```

### 本地操作
```bash
# 检查 VPN
ping 10.0.0.1

# 检查 Docker
docker ps

# 查看网络
ip addr show wg0
```

### Swarm 操作
```bash
# 查看节点
docker node ls

# 查看服务
docker service ls

# 查看服务日志
docker service logs <服务名>
```

---

## 二十一、给明天 Code 的任务说明

### 核心目标

**不是**让 Code 帮我完成任务，**而是**通过项目锻炼我的能力：

| 能力 | 训练方式 |
|------|----------|
| 报错分析 | 看懂错误信息、理解根因 |
| 排障思路 | 从现象→猜测→验证→定位 |
| 命令熟练度 | 每次操作用正确的命令 |
| 网络原理 | 理解 VPN、Overlay、VXLAN 底层原理 |
| 生产经验 | 理解大厂生产环境的做法 |

---

### 训练原则

#### 1. Code 严禁做的事
```
❌ 直接给我修正代码
❌ 自动执行部署
❌ 一次性给所有步骤
❌ 跳过解释直接说"执行这个命令"
```

#### 2. Code 必须做的事
```
✅ 每一步指令前，解释"为什么要这么做"
✅ 告诉我"大厂生产环境中这么做的目的"
✅ 列出 2-3 个新手容易踩的坑
✅ 报错时，引导我通过 log/status/tcpdump 自己找原因
✅ 阶段结束时，帮我记录到 md 文档
```

#### 3. 报错时的引导方式
```
错误：xxx
我的思考：xxx (等我先分析)
你的引导：xxx (不是直接给答案，而是问问题)
    - "你觉得这个错误可能是什么原因？"
    - "你用什么命令来验证这个猜测？"
    - "查看日志有没有更多线索？"
```

#### 4. 教学示例

**错误示范**（禁止）：
```
Code: "执行这个命令修复: systemctl restart docker"
```

**正确示范**：
```
Code: "看到这个报错了吗？先告诉我你第一反应是什么？"
我: "我觉得是服务停了"
Code: "好，那用什么命令验证？"
我: "systemctl status docker"
Code: "对，去执行，然后告诉我结果"
我: "是 inactive"
Code: "那怎么启动？"
我: "systemctl restart docker"
Code: "对，去执行"
```

---

### 为什么这样训练？

**大厂 SRE 的核心能力不是会写代码，而是：**
```
1. 快速定位问题（不是猜测，是验证）
2. 看懂日志和报错（英文、技术术语）
3. 理解系统原理（网络、存储、计算）
4. 冷静排障（不是慌，是排查步骤）
```

**这些能力只能通过实践训练，不能靠看文档。**

---

### 明日实习任务

请 Code 按照以下流程带我操作：

```
1. 现状检查
   - 让我汇报今天的状态
   - 我主动发现并描述问题

2. 问题分析
   - Code 问："你觉得是什么问题？"
   - Code 问："用什么命令验证？"
   - Code 问："结果说明什么？"

3. 方案制定
   - Code 问："你觉得应该怎么解决？"
   - Code 问："有什么风险？"

4. 执行操作
   - Code 问："你来执行？"
   - Code 问："遇到报错怎么办？"

5. 验证结果
   - Code 问："怎么验证修复成功？"
   - Code 问："还有其他问题吗？"

6. 记录总结
   - Code："把这个阶段记录到 md"
   - Code："总结一下你学到了什么"
```

---

### 实习生日志模板

每次阶段结束后，我会记录：

```markdown
### [时间] [阶段名称]

**我观察到的现象**：
- 

**我的分析**：
- 

**我的解决方案**：
- 

**Code 的指导**：
- 

**我学到的**：
- tcpdump抓包基础、NFS挂载、排障思路

---

## 二十二、当前困难和障碍（2026-04-16）

### 问题描述

本地加入Swarm时一直超时：

```
Error response from daemon: Timeout was reached before node joined. 
The attempt to join the swarm will continue in the background.
```

### 排查过程

| 日期 | 排查方向 | 结果 | 说明 |
|------|----------|------|------|
| 04-16 | 网络连通性 | ✅ | VPN正常，ping通 |
| 04-16 | 端口连通性 | ✅ | nc显示2377端口可访问 |
| 04-16 | 本地tcpdump | ✅ | 看到包从wg0发出 |
| 04-16 | 服务器tcpdump | ✅ | 服务器wg0收到包，TCP三次握手成功 |
| 04-16 | 服务器日志 | ❌ | 没有join请求处理记录 |
| 04-16 | 错误日志 | ⚠️ | 发现 dispatcher is stopped |

### 发现的错误日志

```
Apr 16 10:22:56 level=error msg="invalid join token"
Apr 16 12:40:56 level=error msg="agent: session failed"
Apr 16 12:40:56 level=error msg="failed to remove node" error="dispatcher is stopped"
```

### 根因分析

1. **网络层**：✅ 完全正常（tcpdump验证）
2. **应用层**：❌ Swarm没有处理join请求
3. **可能原因**：
   - Docker 29.3.1 Swarm模块有bug
   - dispatcher异常导致无法处理join
   - Token与Swarm状态不匹配

### 已尝试的解决方案

| 方案 | 结果 | 说明 |
|------|------|------|
| 获取新token | ❌ | 仍然超时 |
| 重启本地Docker | ❌ | 仍然超时 |
| 重启服务器Docker | ❌ | 仍然超时 |
| 检查防火墙 | ✅ | 端口正常 |

### 下一步排查方向

1. 检查服务器Docker版本是否有已知bug
2. 尝试降级Docker版本
3. 重新初始化Swarm（会丢失服务）
4. 考虑使用其他方案（不通过Swarm）

### 今日学习总结

| 技能 | 学到内容 |
|------|----------|
| tcpdump抓包 | 时间、网卡、方向、源目标、Flags |
| TCP三次握手 | SYN→SYN-ACK→ACK |
| NFS挂载 | 服务器export + 本地mount |
| 日志分析 | journalctl、grep error |
| 问题分类 | 网络层vs应用层 |

---

## 二十三、公网访问修复（2026-04-16）

### 问题

公网访问 101.133.135.3:3000 返回502

### 排查结果

- 服务器端口监听：✅ 3000端口正常
- 容器运行：✅ curl localhost:3000 正常
- 原因：之前502，后恢复

### 最终结果

| 访问方式 | 状态 |
|----------|------|
| 10.0.0.1:3000 (VPN) | ✅ |
| 101.133.135.3:3000 (公网) | ✅ |

---

## 二十四、面试复习与错误回顾（2026-04-16）

### 今日犯下的错误

| 序号 | 错误 | 原因 | 教训 |
|------|------|------|------|
| 1 | daemon.json配置冲突 | systemd已有-H参数，配置文件又配 | 配置冲突先用grep检查现有参数 |
| 2 | "already part of swarm"误判 | 每次join失败都报这个 | restart docker可清除状态残留 |
| 3 | Swarm timeout误判为网络不通 | tcpdump验证后发现TCP正常 | 网络层≠应用层，必须分层排查 |
| 4 | 在服务器执行join命令 | 混淆了角色 | Manager生成token，Worker用token加入 |

### 排障中的关键发现

| 发现 | 意义 |
|------|------|
| tcpdump看到数据包但服务器无日志 | 网络通但应用层有问题 |
| journalctl看到"dispatcher is stopped" | Swarm内部异常，非网络问题 |
| Docker 29.x存在Swarm bug | 选型时要考虑版本稳定性 |

### 面试问题汇总

#### Q1：什么是混合云架构？
```
答：把"控制面"放在云上（统一入口、身份、调度、审计），
把"数据面/算力面"放在本地（GPU、敏感数据、边缘节点），
通过VPN/专线互联，形成统一管理。
```

#### Q2：Swarm join超时的排查思路？
```
答：四层定位法：
1. 现象：join timeout
2. 假设：网络不通 / 应用层问题 / Token问题
3. 验证：tcpdump抓包看三次握手 + 看服务器日志
4. 结论：TCP成功但应用无响应 → Docker 29.x bug
```

#### Q3：VPN通了但业务不通怎么排查？
```
答：先用"三问法"：
1. 请求有没有发出去？
2. 请求有没有到对端？
3. 对端收到了有没有处理成功？

再分清是"网络不通"还是"应用拒绝"：
ping通 ≠ TCP正常
TCP正常 ≠ 应用授权正常
```

#### Q4：tcpdump抓包看什么？
```
答：核心字段：
- 时间戳
- 网卡（eth0/wg0）
- 方向（In/Out）
- 源→目标IP和端口
- Flags：[S]=SYN [S.]=SYN-ACK [.]=ACK [P.]=Push [F.]=FIN
```

#### Q5：什么是Zero Trust（零信任）？
```
答：不再依赖网络位置决定信任。
核心：每次访问都要验证身份、设备、策略。
不是不要网络边界，而是身份先于网络。
```

#### Q6：什么是控制面vs数据面？
```
答：
- 控制面：登录、鉴权、调度、审计（云上）
- 数据面：实际训练流量、镜像分发（本地）
分离目的：控制面可严格收敛，数据面考虑吞吐和延迟
```

#### Q7：502错误怎么排查？
```
答：
1. 入口层是否正常（curl localhost）
2. 上游后端是否监听（ss -tlnp）
3. DNS是否解析正确
4. 后端健康检查是否通过
5. 请求体/超时/证书配置问题
不要一上来就重启服务，先分层排查。
```

#### Q8：网络分区通常分哪几个区？
```
答：6个逻辑区
- 接入区：WAF、反向代理
- 控制区：API、调度器
- 管理区：堡垒机、跳板机
- 业务区：Portal、应用
- 算力区：GPU节点
- 数据区：存储、元数据
```

### 排障框架回顾

#### 四层定位法
```
第一层：现象（看到什么？）
第二层：假设（可能是什么？）
第三层：验证（用什么证明？）
第四层：结论（哪个假设成立？）
```

#### 三问法
```
1. 请求有没有发出去？
2. 请求有没有到对端？
3. 对端收到了有没有处理成功？
```

### 常用排障命令

| 用途 | 命令 |
|------|------|
| 端口连通性 | nc -zv host port |
| 端口监听 | ss -tlnp |
| 抓包 | tcpdump -i interface port xxx |
| 查看日志 | journalctl -u service --no-pager -n xxx |
| Docker状态 | docker info |
| Swarm节点 | docker node ls |
| 路由 | ip route get xxx |
| TCP连接 | telnet host port |

---
