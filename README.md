<div align="center">

# Port Guard UI

**Linux 服务器端口防火墙可视化管理面板**

![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?style=flat-square&logo=python&logoColor=white)
![iptables](https://img.shields.io/badge/%E9%98%B2%E7%81%AB%E5%A2%99-iptables%20%2B%20ipset-2F855A?style=flat-square)
![Docker](https://img.shields.io/badge/Docker-%E7%AB%AF%E5%8F%A3%E8%AF%86%E5%88%AB-2496ED?style=flat-square&logo=docker&logoColor=white)
![No Build](https://img.shields.io/badge/%E5%89%8D%E7%AB%AF-%E6%97%A0%E9%9C%80%E6%9E%84%E5%BB%BA-6B7280?style=flat-square)

一个轻量、直观、适合个人 VPS 和云服务器的端口开放管理工具。

</div>

---

## 默认策略

> **首次安装默认打开全部正在使用的端口，无任何来源限制。**
>
> 首次启动时，Port Guard UI 会扫描当前正在监听的宿主机端口和 Docker 已发布端口，自动生成“全网开放”规则并写入防火墙。
> **这次首次初始化不会备份原有 iptables 策略，也不会清空原有策略。**

| 项目 | 默认行为 |
| --- | --- |
| 首次安装 | 自动托管当前正在监听的端口 |
| 端口策略 | 全网开放，无白名单、黑名单、CN 拦截 |
| 原有防火墙策略 | 不备份、不清空，只插入 Port Guard 自己的放行链 |
| 后续手动应用 | 会正常创建备份，方便回滚 |
| 面板访问 | 默认只监听 `127.0.0.1:8787`，通过 SSH 隧道访问 |
| 云厂商安全组 | 仍需要在云控制台单独放行，脚本无法修改云平台安全组 |

如果你想让首次安装保持空配置，请把环境变量改成：

```text
PORT_GUARD_INIT_OPEN_LISTENING=0
```

---

## 功能概览

| 能力 | 说明 |
| --- | --- |
| 端口扫描 | 读取 `ss` 和 Docker 映射，区分宿主机端口和容器发布端口 |
| 首次自动开放 | 第一次启动自动把正在监听的端口设为全网开放 |
| 托管全部监听 | 面板内可一键把新监听端口加入管理 |
| 访问策略 | 支持全网开放、策略组放行、禁止中国 IP、除黑名单外开放、仅本机/隧道 |
| 策略组 | 维护内网 IP、个人 IP、可信来源，并复用到多个端口 |
| Docker 识别 | 自动展示 Docker 已发布端口，支持托管容器入口 |
| 备份回滚 | 后续手动应用规则时自动创建 iptables 备份 |
| 轻量部署 | 单个 Python 服务加静态前端，无数据库、无前端构建 |

---

## 适用环境

| 环境 | 要求 |
| --- | --- |
| 系统 | Debian / Ubuntu / CentOS / Rocky / AlmaLinux / Fedora 等 systemd Linux |
| 权限 | root 或具备 `sudo` 权限 |
| Python | Python 3.10+ |
| 依赖 | `iptables`、`ipset`、`iproute2` |
| 可选 | Docker CLI、`netfilter-persistent` |

---

## 从零开始部署

### 1. 登录服务器

```bash
ssh root@你的服务器公网IP
```

如果 SSH 不是 22 端口：

```bash
ssh root@你的服务器公网IP -p 22222
```

### 2. 拉取项目

Debian / Ubuntu：

```bash
sudo apt update
sudo apt install -y git
git clone https://github.com/Timmyzzo/port-guard-ui.git
cd port-guard-ui
```

CentOS / Rocky / AlmaLinux / Fedora：

```bash
sudo dnf install -y git || sudo yum install -y git
git clone https://github.com/Timmyzzo/port-guard-ui.git
cd port-guard-ui
```

### 3. 一键检测、安装、启动

在项目根目录执行：

```bash
sudo env PG_SSH_PORT=22 PG_BIND=127.0.0.1 PG_PORT=8787 bash <<'EOF'
set -e

APP=/opt/port-guard-ui
CONF=/etc/port-guard-ui
BACKUP=/var/backups/port-guard-ui
ENV_FILE=/etc/port-guard-ui.env

[ -f server.py ] && [ -d static ] || { echo "请在项目根目录运行"; exit 1; }
command -v systemctl >/dev/null || { echo "需要 systemd"; exit 1; }

if command -v apt-get >/dev/null; then
  apt-get update
  DEBIAN_FRONTEND=noninteractive apt-get install -y python3 iptables ipset iproute2 ca-certificates
  DEBIAN_FRONTEND=noninteractive apt-get install -y netfilter-persistent || true
elif command -v dnf >/dev/null; then
  dnf install -y python3 iptables ipset iproute ca-certificates
elif command -v yum >/dev/null; then
  yum install -y python3 iptables ipset iproute ca-certificates
else
  echo "未识别包管理器，请手动安装 python3、iptables、ipset、iproute2"
  exit 1
fi

python3 - <<'PY'
import sys
if sys.version_info < (3, 10):
    raise SystemExit("需要 Python 3.10 或更高版本")
PY

install -d "$APP" "$APP/static" "$CONF" "$BACKUP"
install -m 0644 server.py "$APP/server.py"
cp -a static/. "$APP/static/"
install -m 0644 examples/port-guard-ui.service /etc/systemd/system/port-guard-ui.service

TOKEN="$(python3 -c 'import secrets; print(secrets.token_hex(24))')"
SECRET="$(python3 -c 'import secrets; print(secrets.token_hex(24))')"

cat > "$ENV_FILE" <<ENV
PORT_GUARD_HOME=$APP
PORT_GUARD_CONFIG_DIR=$CONF
PORT_GUARD_BACKUP_DIR=$BACKUP
PORT_GUARD_BIND=${PG_BIND:-127.0.0.1}
PORT_GUARD_PORT=${PG_PORT:-8787}
PORT_GUARD_TOKEN=$TOKEN
PORT_GUARD_SECRET=$SECRET
PORT_GUARD_SAFE_INPUT_PORTS=${PG_SSH_PORT:-22}
PORT_GUARD_INIT_OPEN_LISTENING=1
ENV
chmod 600 "$ENV_FILE"

systemctl daemon-reload
systemctl enable --now port-guard-ui

echo
echo "部署完成"
echo "面板地址: http://127.0.0.1:${PG_PORT:-8787}"
echo "SSH 隧道: ssh -L ${PG_PORT:-8787}:127.0.0.1:${PG_PORT:-8787} root@你的服务器公网IP -p ${PG_SSH_PORT:-22}"
echo "访问令牌: $TOKEN"
EOF
```

这段脚本会自动完成：

- 检测包管理器并安装依赖。
- 安装项目到 `/opt/port-guard-ui`。
- 生成 `/etc/port-guard-ui.env`。
- 启动 `port-guard-ui` 服务。
- 首次启动时自动开放当前正在监听的端口。
- **首次开放不创建 iptables 备份，不清空原有策略。**

### 4. 打开面板

在本地电脑执行 SSH 隧道：

```bash
ssh -L 8787:127.0.0.1:8787 root@你的服务器公网IP -p 22
```

浏览器访问：

```text
http://127.0.0.1:8787
```

访问令牌可以重新查看：

```bash
sudo grep PORT_GUARD_TOKEN /etc/port-guard-ui.env
```

### 5. 放行云厂商安全组

如果业务端口需要公网访问，还要到云厂商控制台放行入站规则。服务器本机放行不等于云安全组已放行。

通用开放规则：

| 协议 | 端口 | 来源 |
| --- | --- | --- |
| TCP | `1-65535` | `0.0.0.0/0` |
| UDP | `1-65535` | `0.0.0.0/0` |
| ICMP | 全部 | `0.0.0.0/0` |

如果使用 IPv6，也同步放行 `::/0`。

---

## 手动部署

```bash
sudo mkdir -p /opt/port-guard-ui /etc/port-guard-ui /var/backups/port-guard-ui
sudo cp server.py /opt/port-guard-ui/server.py
sudo cp -r static /opt/port-guard-ui/static
sudo cp examples/port-guard-ui.env.example /etc/port-guard-ui.env
sudo cp examples/port-guard-ui.service /etc/systemd/system/port-guard-ui.service
sudo nano /etc/port-guard-ui.env
sudo systemctl daemon-reload
sudo systemctl enable --now port-guard-ui
```

重点检查：

```text
PORT_GUARD_TOKEN=换成足够长的随机字符串
PORT_GUARD_SECRET=换成另一个足够长的随机字符串
PORT_GUARD_SAFE_INPUT_PORTS=你的 SSH 端口
PORT_GUARD_INIT_OPEN_LISTENING=1
```

`PORT_GUARD_INIT_OPEN_LISTENING=1` 只在 `/etc/port-guard-ui/config.json` 不存在时生效。已有配置时重启服务不会重复初始化。

---

## 面板使用

### 托管新端口

1. 打开面板。
2. 进入端口列表。
3. 点击“托管全部监听”。
4. 新加入端口默认是“全网开放”。
5. 点击“应用到防火墙”。

### 清空所有限制

如果你已经启用了白名单、黑名单或 CN 拦截，想恢复全开放：

1. 进入“设置”。
2. 点击“清空所有限制”。
3. 二次确认。
4. 所有托管端口会恢复为全网开放。

### 策略模式

| 模式 | 效果 |
| --- | --- |
| 全网开放 | 允许所有来源访问指定端口 |
| 策略组放行 | 只允许选中的来源组访问 |
| 禁止中国 IP | 丢弃中国大陆来源 IP，其他来源放行 |
| 除黑名单外开放 | 默认开放，但拒绝黑名单来源组 |
| 仅本机/隧道 | 不开放公网直连，适合只走本机或 SSH 隧道 |

---

## 配置说明

环境变量文件：

```text
/etc/port-guard-ui.env
```

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `PORT_GUARD_HOME` | `/opt/port-guard-ui` | 应用目录 |
| `PORT_GUARD_CONFIG_DIR` | `/etc/port-guard-ui` | 配置目录 |
| `PORT_GUARD_BACKUP_DIR` | `/var/backups/port-guard-ui` | 后续手动应用规则的备份目录 |
| `PORT_GUARD_BIND` | `127.0.0.1` | 面板监听地址 |
| `PORT_GUARD_PORT` | `8787` | 面板端口 |
| `PORT_GUARD_TOKEN` | 空 | 登录令牌 |
| `PORT_GUARD_SECRET` | token 或开发默认值 | Cookie 签名密钥 |
| `PORT_GUARD_SAFE_INPUT_PORTS` | `22222` | 应用规则时临时保护的 SSH 端口 |
| `PORT_GUARD_INIT_OPEN_LISTENING` | `0` | 首次无配置时，是否自动开放当前监听端口且不备份 |

主配置文件：

```text
/etc/port-guard-ui/config.json
```

---

## 防火墙行为

Port Guard UI 只管理自己的链：

```text
PORTGUARD-INPUT
PORTGUARD-DOCKER
```

首次安装且 `PORT_GUARD_INIT_OPEN_LISTENING=1` 时：

- 扫描当前监听端口。
- 生成全网开放规则。
- 插入 Port Guard 管理链。
- 不备份原有策略。
- 不清空原有策略。

后续在面板点击“应用到防火墙”时：

- 会先创建 iptables 备份。
- 然后刷新 Port Guard 自己管理的链。
- 如果安装了 `netfilter-persistent`，会保存为开机规则。

回滚备份：

```bash
sudo iptables-restore < /var/backups/port-guard-ui/iptables-YYYYMMDDTHHMMSSZ.rules
```

---

## 常用维护命令

```bash
sudo systemctl status port-guard-ui --no-pager
sudo journalctl -u port-guard-ui -f
sudo systemctl restart port-guard-ui
sudo ss -lntup
sudo iptables -S PORTGUARD-INPUT
sudo iptables -S PORTGUARD-DOCKER
```

如果想重新触发“首次自动开放当前监听端口”，可以删除配置后重启：

```bash
sudo rm -f /etc/port-guard-ui/config.json
sudo systemctl restart port-guard-ui
```

---

## 对外开放面板

默认推荐 SSH 隧道访问，不建议直接把管理面板暴露到公网。

如果确实要公网访问面板：

```bash
sudo sed -i 's/^PORT_GUARD_BIND=.*/PORT_GUARD_BIND=0.0.0.0/' /etc/port-guard-ui.env
sudo systemctl restart port-guard-ui
```

同时确保：

- `PORT_GUARD_TOKEN` 是足够长的随机字符串。
- 云安全组放行 `8787/tcp`。
- 你清楚管理面板公网暴露的风险。

---

## 本地开发

```bash
sudo PORT_GUARD_HOME="$PWD" \
  PORT_GUARD_CONFIG_DIR="$PWD/.local/config" \
  PORT_GUARD_BACKUP_DIR="$PWD/.local/backups" \
  PORT_GUARD_BIND=127.0.0.1 \
  PORT_GUARD_PORT=8787 \
  PORT_GUARD_TOKEN=dev-token \
  python3 server.py
```

语法检查：

```bash
python3 -m py_compile server.py
node --check static/app.js
```

---

## 项目结构

```text
.
├── server.py
├── static/
│   ├── app.js
│   ├── index.html
│   └── styles.css
└── examples/
    ├── config.example.json
    ├── port-guard-ui.env.example
    └── port-guard-ui.service
```

---

<div align="center">

**首次安装自动开放当前正在使用的端口；后续限制策略由你在面板中主动开启。**

</div>
