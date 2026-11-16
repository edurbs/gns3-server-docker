# GNS3 Server in Docker Compose

Run the [GNS3 server](https://github.com/GNS3/gns3-server) as a Docker Compose
stack and drive it entirely from the **browser Web UI** — no desktop client
needed. Built and used on Windows + WSL2 (Docker Desktop), works anywhere
Docker runs on x86_64.

## What you get

- `gns3/gns3-server:2.2.61` (pinned) running privileged with `/dev/kvm` passthrough, so
  QEMU-based appliances run at native speed
- Web UI on **http://localhost:3080** — node consoles (telnet/VNC) are proxied over
  WebSocket on the same port, so nothing else is exposed
- Credentials rendered from `.env` at container start — never committed
- Projects, appliance images and user templates persist across `docker compose down`

## Requirements

- Docker Engine + Docker Compose v2 (`docker compose`), x86_64
- `/dev/kvm` available on the host (optional but strongly recommended — without it
  QEMU falls back to slow software emulation)
- WSL2 note: `/dev/kvm` must exist inside the WSL distro / Docker VM

## Quickstart

```bash
cp .env.example .env
# edit .env and set your credentials
docker compose up -d
```

Open http://localhost:3080 and log in with the credentials from `.env`.

To change credentials later: edit `.env`, then `docker compose up -d`.

## Files

| Path | Role |
|------|------|
| `compose.yaml` | the stack (privileged, KVM, loopback-only port 3080) |
| `.env` / `.env.example` | GNS3 HTTP auth credentials (untracked / template) |
| `config/gns3_server.conf.template` | server config template with credential tokens |
| `config/gns3_server.conf` | rendered at container start (untracked) |
| `config/gns3_controller.conf` | GNS3 controller state incl. your templates (untracked) |
| `data/` | projects, appliance images, symbols (untracked) |

From Windows, browse your labs at `\\wsl.localhost\<distro>\home\<you>\...\data`
(writing from a WSL shell needs `sudo` — the container runs as root).

## Security notes

- The port is bound to **127.0.0.1 only**. The container runs privileged, which is
  close to root on the host — do not widen the binding without changing the default
  credentials first.
- Auth is required (`auth = True` in the template) because the Web UI arrives via
  Docker's NAT and GNS3 refuses unauthenticated non-local clients.
- Basic auth over plain HTTP — fine for localhost use; put a TLS proxy in front if
  you ever expose it beyond loopback.

## Limitations

- **Docker-based GNS3 nodes don't run**: the official image does not ship the
  Python `docker` library. QEMU, Dynamips (IOS routers), VPCS and the built-in
  switch/hub all work.
- x86_64 only (Dynamips/uBridge have no ARM builds).
- No TLS.

## How it works

The GNS3 server reads its auth settings only from its config file — there is no
environment-variable support. Compose therefore mounts the tracked template into
the container and renders it to the config path at startup using `python3`
literal string replacement (safe for any special characters in your password),
then `exec`s the server so it stays PID 1 and receives signals.

The template is mounted into GNS3's versioned config directory
(`/root/.config/GNS3/2.2/`), which is also where the controller writes
`gns3_controller.conf` (your user templates) — so one bind mount keeps both
persistent.

## Credits

Everything comes from the [GNS3 project](https://www.gns3.org/) — this repo only
packages it for compose-based, browser-only use. Also see the
[design notes](docs/superpowers/specs/2026-11-16-gns3-docker-compose-design.md)
for the decisions behind this setup.

## License

[MIT](LICENSE)
