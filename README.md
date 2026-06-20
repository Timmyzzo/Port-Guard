<div align="center">

# Port Guard UI

**A lightweight web console for managing Linux port access with iptables, ipset, and Docker-published ports.**

![Python](https://img.shields.io/badge/Python-3.10%2B-3776AB?style=flat-square&logo=python&logoColor=white)
![iptables](https://img.shields.io/badge/firewall-iptables%20%2B%20ipset-2F855A?style=flat-square)
![Docker](https://img.shields.io/badge/Docker-aware-2496ED?style=flat-square&logo=docker&logoColor=white)
![No build](https://img.shields.io/badge/frontend-no%20build-6B7280?style=flat-square)

</div>

Port Guard UI 是一个面向个人服务器的端口可视化管理面板。它会扫描宿主机监听端口和 Docker 映射端口，把常用的「全网开放、策略组放行、禁止 CN、黑名单、关闭」整理成可编辑规则，并在应用前自动备份 iptables 规则。

> 默认公开仓库配置不会限制任何端口。真正写入防火墙前，请先确认 SSH 端口已加入 `PORT_GUARD_SAFE_INPUT_PORTS`，并保留一个可用的回滚方式。

## Features

| 能力 | 说明 |
| --- | --- |
| 端口扫描 | 读取 `ss` 和 Docker 端口映射，区分宿主机与容器发布端口 |
| 策略组 | 用来源组维护内网、个人 IP、可信来源，可多选应用到规则 |
| 访问模式 | 支持全网开放、策略组白名单、禁止 CN、黑名单、仅本机/隧道访问 |
| CN 拦截 | 通过 ipset 维护中国大陆网段，可一键对所有端口启用直连拦截 |
| 安全应用 | 应用规则前自动备份，并可为 SSH 等关键端口插入临时放行规则 |
| 备份管理 | 查看、恢复、删除 `/var/backups/port-guard-ui` 中的 iptables 备份 |
| 轻量部署 | 单个 Python 服务加静态前端，无需数据库和前端构建流程 |

## Requirements

- Linux server with root access
- Python 3.10+
- `iptables`, `iptables-save`, `iptables-restore`
- `ipset`
- `ss` from `iproute2`
- Optional: Docker CLI, if you want Docker-published port detection
- Optional: `netfilter-persistent`, if you want rules persisted automatically

Debian/Ubuntu example:

```bash
sudo apt update
sudo apt install -y python3 iptables ipset iproute2 netfilter-persistent
```

## Install

```bash
sudo mkdir -p /opt/port-guard-ui /etc/port-guard-ui /var/backups/port-guard-ui
sudo cp server.py /opt/port-guard-ui/server.py
sudo cp -r static /opt/port-guard-ui/static
sudo cp examples/port-guard-ui.env.example /etc/port-guard-ui.env
sudo cp examples/port-guard-ui.service /etc/systemd/system/port-guard-ui.service
```

Edit `/etc/port-guard-ui.env` before starting:

```bash
sudo nano /etc/port-guard-ui.env
```

Recommended defaults:

- Keep `PORT_GUARD_BIND=127.0.0.1`.
- Set a long random `PORT_GUARD_TOKEN`.
- Put your SSH port in `PORT_GUARD_SAFE_INPUT_PORTS`, for example `22222`.

Start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now port-guard-ui
sudo systemctl status port-guard-ui
```

Open the panel through an SSH tunnel:

```bash
ssh -L 8787:127.0.0.1:8787 root@YOUR_SERVER_IP -p 22222
```

Then visit:

```text
http://127.0.0.1:8787
```

If you intentionally bind the panel to `0.0.0.0`, the server requires `PORT_GUARD_TOKEN`. Exposing this panel directly to the public internet is not recommended; use a reverse proxy with authentication or an SSH tunnel.

## Usage

1. Open the panel and authenticate with `PORT_GUARD_TOKEN`.
2. Go to the port list and inspect detected listening ports.
3. Create or edit source groups under strategy settings.
4. Select a port rule mode:
   - **全网开放**: accept traffic for the selected ports.
   - **策略组放行**: only selected source groups can access.
   - **禁止中国 IP**: drop traffic from the configured CN ipset.
   - **除黑名单外开放**: allow everyone except selected blacklist groups.
   - **仅本机/隧道**: restrict direct access.
5. Preview generated iptables commands.
6. Apply rules. Port Guard UI writes a backup first, then updates managed chains.

Backups are stored in:

```text
/var/backups/port-guard-ui
```

## Configuration

Runtime environment variables:

| Variable | Default | Description |
| --- | --- | --- |
| `PORT_GUARD_HOME` | `/opt/port-guard-ui` | Application directory |
| `PORT_GUARD_CONFIG_DIR` | `/etc/port-guard-ui` | Stores `config.json` and CN zone file |
| `PORT_GUARD_BACKUP_DIR` | `/var/backups/port-guard-ui` | iptables backup directory |
| `PORT_GUARD_BIND` | `127.0.0.1` | HTTP bind address |
| `PORT_GUARD_PORT` | `8787` | HTTP port |
| `PORT_GUARD_TOKEN` | empty | Login token; required when bind is not loopback |
| `PORT_GUARD_SECRET` | token/dev fallback | Cookie signing secret |
| `PORT_GUARD_SAFE_INPUT_PORTS` | `22222` | Critical ports temporarily allowed during apply |

The main config file is:

```text
/etc/port-guard-ui/config.json
```

You can start from `examples/config.example.json`, but review it carefully before applying any firewall changes.

## Safety Notes

- Port Guard UI manages only its own chains: `PORTGUARD-INPUT` and `PORTGUARD-DOCKER`.
- Existing unmanaged firewall rules are not displayed as editable policy groups.
- Applying firewall rules always creates an iptables backup first.
- Keep a second SSH session open when testing new restrictions.
- Before restricting SSH, verify your current IP is in a selected source group.
- If you lose access, restore the newest backup from console/VNC/provider rescue mode:

```bash
sudo iptables-restore < /var/backups/port-guard-ui/iptables-YYYYMMDDTHHMMSSZ.rules
```

## Development

Run locally on a Linux test machine:

```bash
sudo PORT_GUARD_HOME="$PWD" \
  PORT_GUARD_CONFIG_DIR="$PWD/.local/config" \
  PORT_GUARD_BACKUP_DIR="$PWD/.local/backups" \
  PORT_GUARD_BIND=127.0.0.1 \
  PORT_GUARD_PORT=8787 \
  PORT_GUARD_TOKEN=dev-token \
  python3 server.py
```

Syntax checks:

```bash
python3 -m py_compile server.py
node --check static/app.js
```

## Project Layout

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

## Roadmap

- nftables backend
- import/export profiles
- rule diff history
- optional reverse-proxy deployment examples
