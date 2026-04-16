# SSH 密钥对配置指南

## 1. 生成密钥对（如果还没有）

```bash
# 生成 RSA 密钥（兼容性好）
ssh-keygen -t rsa -b 4096 -C "ansible@your-email.com" -f ~/.ssh/ansible_key

# 或生成 Ed25519 密钥（更安全、更快）
ssh-keygen -t ed25519 -C "ansible@your-email.com" -f ~/.ssh/ansible_key

# 设置密钥权限
chmod 600 ~/.ssh/ansible_key
chmod 644 ~/.ssh/ansible_key.pub
```

## 2. 上传公钥到服务器

```bash
# 方式1: 使用 ssh-copy-id（推荐）
ssh-copy-id -i ~/.ssh/ansible_key.pub root@你的服务器IP

# 方式2: 手动复制
cat ~/.ssh/ansible_key.pub
# 复制公钥内容，添加到服务器 ~/.ssh/authorized_keys
```

## 3. 测试连接

```bash
# 测试 SSH 连接
ssh -i ~/.ssh/ansible_key root@你的服务器IP

# 测试 Ansible 连接
ansible -i hosts server1 -m ping
```

## 4. 运行 Playbook

```bash
# 运行（使用默认密钥 ~/.ssh/id_rsa）
ansible-playbook -i hosts nginx-setup.yml

# 或指定密钥
ansible-playbook -i hosts nginx-setup.yml --private-key=~/.ssh/ansible_key
```

## 5. 阿里云服务器配置公钥

```
阿里云控制台 → ECS → 密钥对 → 创建密钥对
↓ 
绑定到你的 ECS 实例
↓ 
下载私钥文件（如 ansible_key.pem）
```

## 安全建议

| 操作 | 命令 |
|------|------|
| 禁用密码登录 | `PasswordAuthentication no` 在 `/etc/ssh/sshd_config` |
| 禁用 root 直接登录 | `PermitRootLogin no` |
| 修改默认 SSH 端口 | `Port 2222` |
| 防火墙只允许你的 IP | `ufw allow from 你的IP to any port 22` |
