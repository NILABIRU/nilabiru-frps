# Nilabiru FRPS

A Docker Compose setup for the FRP server (`frps`), the public-facing end of the Nilabiru tunnel. It accepts connections from FRP clients (`frpc`) and forwards public traffic through them to services running behind NAT or a private network.

---

## Overview

**Nilabiru FRPS** runs a single container, [`frps`](https://github.com/fatedier/frp), on a server with a public IP address. FRP clients (such as the `nilabiru-frpc` container in the Nilabiru Data Hub) connect to it on port `7000` and authenticate with a shared token. Traffic arriving on the server's public ports `80` and `443` is then tunneled through the client to the target service. Deployment is handled by a single `deploy.sh` script.

---

## Services

| Service           | Image                   | Port(s)                     | Description                                                                                            |
| ----------------- | ----------------------- | --------------------------- | ------------------------------------------------------------------------------------------------------ |
| **nilabiru-frps** | `fatedier/frps:v0.69.1` | `7000`, `80`, `443`, `7500` | FRP server that accepts frpc clients and exposes their tunneled services; web dashboard on port `7500` |

| Port   | Purpose                                                                   |
| ------ | ------------------------------------------------------------------------- |
| `7000` | Bind port — FRP clients (`frpc`) connect here                             |
| `80`   | Public HTTP traffic, tunneled to the client's proxy (`remotePort = 80`)   |
| `443`  | Public HTTPS traffic, tunneled to the client's proxy (`remotePort = 443`) |
| `7500` | frps web dashboard                                                        |

The container uses `restart: unless-stopped` and runs on the default Docker Compose network. Its configuration is read from `frps.toml`, mounted read-only at `/etc/frp/frps.toml`.

> **Warning:** All four ports are published on **all network interfaces**. Ports `80` and `443` are meant to be public, and `7000` must be reachable by your FRP clients. The dashboard on port `7500` is served over plain HTTP with basic authentication — restrict it with a firewall rule (for example to your own IP) rather than leaving it open to the internet.

---

## Requirements

- Docker Engine `20.10+`
- Docker Compose `v2+`
- A server with a public IP address (for example a VPS)
- Ports `7000`, `80`, `443`, and `7500` open on the host firewall, and not already used by another service (such as a web server)
- An FRP client configured with the same token (e.g. `nilabiru-frpc`)

---

## Getting Started

### 1. Clone the repository

```bash
git clone https://github.com/NILABIRU/nilabiru-frps.git
cd nilabiru-frps
```

### 2. Configure environment variables

Copy the provided `env` file and fill in all values:

```bash
cp env .env
```

Then edit `.env`:

```env
# FRP
FRP_TOKEN=
FRP_USER=
FRP_PASSWORD=
```

| Variable       | Description                                                                      |
| -------------- | -------------------------------------------------------------------------------- |
| `FRP_TOKEN`    | Shared secret that FRP clients must present to connect (use a long random value) |
| `FRP_USER`     | Username for the frps web dashboard                                              |
| `FRP_PASSWORD` | Password for the frps web dashboard                                              |

> **Note:** Never commit `.env` to version control. It is already listed in `.gitignore`.

> **Note:** `FRP_TOKEN` must be identical on the server and on every FRP client. In the Nilabiru Data Hub, set the same value as `FRP_TOKEN`, and set `FRP_SERVER_ADDR` to this server's address.

### 3. Prepare the frps configuration

Ensure `frps.toml` exists in the repository root. The environment variables are passed into the container and should be referenced inside `frps.toml` using FRP's environment variable expansion syntax:

```toml
bindPort = 7000

[auth]
method = "token"
token = "{{ .Envs.FRP_TOKEN }}"

[webServer]
addr = "0.0.0.0"
port = 7500
user = "{{ .Envs.FRP_USER }}"
password = "{{ .Envs.FRP_PASSWORD }}"
```

> **Note:** `bindPort` must match the port FRP clients connect to (`serverPort = 7000` in `frpc.toml`). Ports `80` and `443` do not need to be declared in `frps.toml`: they are opened on demand when a client registers a proxy with `remotePort = 80` or `remotePort = 443`, and they are published by Docker through the `ports` section of `docker-compose.yml`.

### 4. Start the server

The recommended way is the provided deploy script:

```bash
chmod +x deploy.sh
./deploy.sh
```

`deploy.sh` stops on the first error (`set -e`) and does the following:

1. Validates the Compose configuration with `docker compose config --quiet`.
2. Deploys/redeploys the service with `docker compose up -d --remove-orphans --build`.
3. Always runs a cleanup on exit (even if a step fails) that removes dangling images with `docker image prune -f`.

Alternatively, you can start the server directly:

```bash
docker compose up -d
```

To verify it is running and check connected clients in the logs:

```bash
docker compose ps
docker compose logs -f nilabiru-frps
```

---

## Service Access

| Service              | URL / Address             |
| -------------------- | ------------------------- |
| FRP client bind port | `<SERVER_IP>:7000`        |
| frps Web Dashboard   | `http://<SERVER_IP>:7500` |
| Tunneled HTTP        | `http://<SERVER_IP>:80`   |
| Tunneled HTTPS       | `https://<SERVER_IP>:443` |

Log in to the dashboard with `FRP_USER` and `FRP_PASSWORD`. It shows server status, connected clients, and active proxies.

---

## Volumes & Mounts

This stack uses no named volumes. It needs a single bind mount:

| Mount                               | Type                   | Purpose                 |
| ----------------------------------- | ---------------------- | ----------------------- |
| `./frps.toml:/etc/frp/frps.toml:ro` | Bind mount (read-only) | frps configuration file |

> **Note:** frps is stateless — no data needs to be backed up. After editing `frps.toml`, apply the change with `docker compose restart nilabiru-frps` (or re-run `./deploy.sh`).

---

## License

This project is licensed under the [MIT License](LICENSE).
Copyright © 2026 Andry Pebrianto
