# Cloudflare Tunnel Stack

Cloudflare Tunnel, nginx reverse proxy, and optional cAdvisor — chạy qua Docker Compose trong WSL (Windows) hoặc trên VPS.

## Hai chế độ deploy

| | Single | HA Replicate |
|---|--------|--------------|
| Tunnel | Mỗi máy 1 tunnel riêng | 1 tunnel, 2 connector |
| `.env` | Token khác nhau trên từng máy | Cùng `TUNNEL_TOKEN` trên cả 2 máy |
| Máy chạy | 1 hoặc nhiều máy độc lập | Windows + VPS cùng lúc |
| Domain | Mỗi tunnel route domain riêng | Cùng domain, Cloudflare load-balance |

### Single — mỗi máy tunnel riêng

1. Tạo tunnel trên [Cloudflare Dashboard](https://one.dash.cloudflare.com/) → Zero Trust → Networks → Tunnels
2. Cấu hình Public Hostname → `http://nginx:80`
3. Copy token vào `.env` trên **máy đó**
4. Chạy docker compose (xem [Run](#run) bên dưới)

### HA Replicate — cùng token, 2 máy

1. Dùng **cùng** `TUNNEL_TOKEN` trong `.env` trên cả VPS và Windows
2. Đảm bảo nginx config (`nginx/conf.d/`) và app containers giống nhau trên cả 2 máy
3. Chạy compose trên cả 2 máy
4. Kiểm tra Dashboard → Tunnel hiện **2 connectors** healthy

## Architecture

```
Internet → Cloudflare → cloudflared → nginx → upstream app containers (web network)
                                              ↓
                                         cadvisor :8080 (VPS only)
```

| Service | Role |
|---------|------|
| `cloudflared` | Cloudflare Tunnel, dùng `TUNNEL_TOKEN` từ `.env` |
| `nginx` | Reverse proxy; route theo hostname qua `nginx/conf.d/*.conf` |
| `cadvisor` | Metrics UI — mặc định trên VPS (`:8080`), optional trên Windows (`--profile monitor`, `:8081`) |

## File compose

| File | Dùng cho |
|------|----------|
| [`docker-compose.vps2.yml`](docker-compose.vps2.yml) | VPS — cloudflared + nginx + cAdvisor |
| [`docker-compose.windows.yml`](docker-compose.windows.yml) | Windows WSL — cloudflared + nginx, cAdvisor optional |

## Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) với WSL2 (Windows) hoặc Docker Engine (VPS)
- External Docker network `web`:

```bash
docker network create web
```

## Configuration

```bash
cp .env.example .env
# Điền TUNNEL_TOKEN
```

**Không commit `.env` hoặc hardcode token trong compose files.**

## nginx upstreams

| Config file | Hostname | Upstream container |
|-------------|----------|-------------------|
| `000-default.conf` | `_` (default) | `dntr-la13_web` |
| `stnn.conf` | `stnn.congnghetintien.vn` | `stnn_web` |
| `dntrdongthap.conf` | `store.congnghetintien.vn` | `dntr-la13_web` |
| `dulich-myngai.conf` | `dulich-myngai.congnghetintien.vn` | `dulich-myngai_web` |
| `ketnoitanlong.conf` | `kntl.congnghetintien.vn` | `kntl_web` |

App stacks phải join network `web`:

```yaml
networks:
  web:
    external: true

services:
  web:
    networks:
      - default
      - web
```

## Run

```bash
cd /home/ghunter/cloudflared

# VPS
docker compose -f docker-compose.vps2.yml up -d

# Windows WSL
docker compose -f docker-compose.windows.yml up -d
```

Windows cAdvisor (optional):

```bash
docker compose -f docker-compose.windows.yml --profile monitor up -d
# UI: http://localhost:8081
```

## Stop / restart

```bash
# VPS
docker compose -f docker-compose.vps2.yml down
docker compose -f docker-compose.vps2.yml up -d --force-recreate

# Windows
docker compose -f docker-compose.windows.yml down
```

## Troubleshooting

| Problem | Fix |
|---------|-----|
| `network web not found` | `docker network create web` |
| nginx 502 | Upstream container chưa join `web` hoặc chưa chạy |
| cloudflared errors | Kiểm tra `TUNNEL_TOKEN` trong `.env` |
| HA chỉ 1 connector | Chạy compose trên cả 2 máy, cùng token |
| Port 8080 in use | Windows dùng profile monitor (port 8081) |

```bash
docker compose -f docker-compose.vps2.yml logs cloudflared --tail 20
```

Expected cAdvisor: `curl -s -o /dev/null -w "%{http_code}" http://localhost:8080` → `307` hoặc `200`.
