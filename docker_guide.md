# Docker 入门指南

## 一、容器比直接 Ubuntu 运行更安全

```
容器 = 沙箱隔离
```

- **进程隔离**：容器内的进程看不到宿主机上其他进程
- **文件系统隔离**：容器内的修改不影响宿主机（除非用 volume 挂载）
- **网络隔离**：容器有独立的网络栈
- **资源限制**：可限制 CPU、内存使用
- **最小权限**：可使用非 root 用户运行

简单说：**破坏容器只影响容器本身，不会搞垮整台服务器。**

---

## 二、查看网络连接需要安装什么

基础工具通常已包含，但完整版需要：

```dockerfile
RUN apt-get update && apt-get install -y \
    iproute2 \
    net-tools \
    iputils-ping \
    curl \
    && rm -rf /var/lib/apt/lists/*
```

核心工具：

| 工具 | 用途 |
|------|------|
| `iproute2` | 包含 `ss` 命令 |
| `net-tools` | 包含 `netstat` |

不过注意：**容器内看的是容器的网络，不是宿主机的。**

---

## 三、docker build vs docker run 本质区别

| 命令 | 类比 | 做什么 |
|------|------|--------|
| `docker build` | **编译** | 读取 Dockerfile → 生成镜像（只执行一次） |
| `docker run` | **执行** | 读取镜像 → 创建容器 → 运行程序（每次都执行） |

```
docker build   → 产出: 镜像 (Image)    ← 类似 "编译出 .exe"
docker run     → 产出: 容器 (Container) ← 类似 "运行进程"
```

**类比汽车**：build 是造车，run 是开车。造一次可以开多次。

---

## 四、Dockerfile 每行解析

```dockerfile
FROM python:3.11-slim          # ①
WORKDIR /app                   # ②

RUN apt-get update && apt-get install -y \   # ③
    iproute2 \
    net-tools \
    iputils-ping \
    curl \
    procps \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .        # ④
RUN pip install --no-cache-dir -r requirements.txt  # ⑤

COPY . .                       # ⑥

CMD ["python", "app.py"]       # ⑦
```

| 行号 | 指令 | 作用 | 类比 |
|------|------|------|------|
| ① | `FROM` | 选择基础镜像（Python 运行环境） | 买毛坯房 |
| ② | `WORKDIR` | 设置工作目录 | `cd /app` |
| ③ | `RUN` | 在构建时执行命令（安装网络工具） | 装修时装宽带 |
| ④ | `COPY` | 复制文件到镜像 | 搬家时搬文件 |
| ⑤ | `RUN` | 安装 Python 依赖 | 搬家后安装家具 |
| ⑥ | `COPY` | 复制源代码 | 搬入源代码 |
| ⑦ | `CMD` | 容器启动时执行的命令 | 开门营业 |

**两种格式区别：**

```dockerfile
# shell 格式（会启动 shell）
CMD python app.py

# exec 格式（直接执行，更稳定）
CMD ["python", "app.py"]
```

**构建阶段 vs 运行时：**

```
docker build 时执行 → FROM, WORKDIR, RUN, COPY
docker run  时执行   → CMD
```

---

## 五、关于 `rm -rf /var/lib/apt/lists/*`

**`/var/lib/apt/lists/` 存的不是软件包本身，而是软件的"索引目录"。**

```
/var/lib/apt/lists/  ← 存的是 .deb 包的"索引信息"（名称、大小、依赖关系）

/var/cache/apt/archives/  ← 才存的是 .deb 实际安装包
```

**类比：**

- 索引目录 = 图书馆的"目录卡片"
- 实际包 = 书架上的"真正的书"

**为什么删除索引？**

```
apt-get update 后，索引大概 10-50MB
构建时下载，运行时不需要 → 删除减小镜像体积
```

**对比两种写法：**

```dockerfile
# 推荐写法
RUN apt-get update && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*

# 不推荐的写法（如果需要分步）
RUN apt-get update
RUN apt-get install -y curl
# 索引没删，镜像变大
```

**执行顺序很重要：**

```
错误写法：
RUN apt-get install -y curl    ← 此时还没 update，索引是空的
RUN apt-get update             ← 这条白跑了

正确写法：
RUN apt-get update && apt-get install -y curl
```

简单记：**先查目录，再装软件，最后删目录**

---

## 六、安装网络工具是在镜像里，不是在容器里

**安装到"镜像"里，不是容器里。**

```
docker build   → 生成镜像（安装动作在这里发生）
docker run     → 从镜像创建容器（工具已经在里面了）
```

**时间线：**

```
1. 你执行 docker build
   └─ RUN apt-get install iproute2 ...   ← 此时安装到镜像

2. 构建完成，镜像已包含工具

3. 你执行 docker run myapp:latest
   └─ 容器启动，工具已经在了，不需要再安装
```

**验证方式：**

```bash
docker build -t myapp .           # 构建时安装
docker run -it myapp ss -tuln      # 运行容器，直接用 ss
```

所以每次运行容器不需要重新安装，镜像做好就永久带这些工具了。

---

## 七、实战操作

### 文件结构

```
/home/jj/
├── Dockerfile       # 镜像构建配方
├── requirements.txt # Python 依赖
└── app.py           # 应用代码
```

### 常用命令

```bash
# 1. 构建镜像
docker build -t myapp:latest .

# 2. 运行容器（后台）
docker run -d --name myapp -p 5000:5000 myapp:latest

# 3. 进入容器查看网络
docker exec -it myapp bash

# 4. 在容器内测试网络工具
ss -tuln      # 查看端口
netstat -tuln # 同上
ping -c 2 8.8.8.8
curl localhost:5000
ps aux        # 查看进程
```

### Dockerfile 文件内容

**Dockerfile:**

```dockerfile
FROM python:3.11-slim

WORKDIR /app

RUN apt-get update && apt-get install -y \
    iproute2 \
    net-tools \
    iputils-ping \
    curl \
    procps \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["python", "app.py"]
```

**requirements.txt:**

```text
flask==3.0.0
requests==2.31.0
```

**app.py:**

```python
#!/usr/bin/env python3
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return 'Hello from container!'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

---

*整理自 AI 对话，2026-04-12*
