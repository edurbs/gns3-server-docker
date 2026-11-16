# GNS3 Server in Docker Compose — Design

Date: 2026-11-16
Status: Approved

## Intent and why

Run the GNS3 server (`gns3/gns3-server`) as a Docker Compose stack inside WSL2
Ubuntu, driven from the Windows side via the built-in browser Web UI. The user
works on Windows; the agent session runs in WSL. Docker Engine 29.9.3 (Docker
Desktop WSL2 backend, linux/amd64) and Docker Compose v5.7.0 are available;
`/dev/kvm` exists in the WSL distro (24 CPUs, vmx present).

## Goals

- `docker compose up -d` brings up a working GNS3 server.
- Web UI reachable from the Windows browser at `http://localhost:3080`.
- Projects, appliance images and user templates survive `docker compose down`.
- QEMU (KVM-accelerated), Dynamips (IOS), VPCS and built-in switch/hub nodes work.

## Non-goals

- Desktop GUI remote-server support (would require publishing console port ranges).
- Docker-based GNS3 nodes: the official image does not ship the Python `docker`
  library (verified against `requirements.txt`), so container nodes can't run
  without a custom image build. Revisit only if needed.
- SSL/TLS termination.

## Anti-goals

- Exposing the API beyond localhost without auth (privileged container + root-equivalent access).

## Constraints

- Image is x86_64 only (Dynamips/uBridge); environment is linux/amd64. OK.
- The server refuses unauthenticated non-local clients: the browser arrives via
  Docker's NAT gateway, so HTTP auth must be enabled in the server config.
- Config file search order (gns3server/config.py, 2.2 branch): CWD →
  `~/.config/GNS3/<major.minor>/gns3_server.conf` → `~/.config/GNS3.conf` →
  `/etc/gns3/gns3_server.conf` → `/etc/xdg/...`. We mount into the versioned dir
  for the pinned 2.2.x tag; the controller writes `gns3_controller.conf` (user
  templates) next to the server config, so that directory is bind-mounted to
  persist templates too.

## Approaches considered

1. **Official Hub image `gns3/gns3-server:2.2.61` + mounted config (CHOSEN)** —
   pinned, reproducible, no build step; uBridge needs `privileged`, QEMU needs
   `/dev/kvm`; only port 3080 published because the Web UI proxies telnet/VNC
   consoles over WebSocket on 3080.
2. **Build the repo's dev image** — for development only (unprivileged, ad-hoc
   port mapping); rejected.
3. **Publish console port ranges (5000–6000 tcp, VNC range)** — needed only for
   desktop GUI clients or direct telnet from Windows; rejected for browser-only
   use (keeps the port surface minimal).

## Decided details (user-approved)

- Image: `gns3/gns3-server:2.2.61` (Docker Hub `latest` == 2.2.61 at approval time).
- `privileged: true`; device `/dev/kvm`; `restart: unless-stopped`.
- Port: `3080:3080`.
- `config/gns3_server.conf` → `/root/.config/GNS3/2.2/gns3_server.conf` (bind
  mount of the whole `config/` dir so `gns3_controller.conf` persists there):
  - `host = 0.0.0.0`, `port = 3080`, `auth = True`, `user = gns3`, `password = gns3`
  - images/projects/appliances paths pinned under `/root/GNS3`
- `data/` → `/root/GNS3` (bind mount; browsable from Windows via `\\wsl.localhost\...`).

## Left to the builder

- Exact config file contents (based on upstream `conf/gns3_server.conf`, trimmed).
- Verification steps: `docker compose up -d`, `curl -u gns3:gns3 http://localhost:3080/v2/version`,
  Web UI served, container healthy with no error logs.
