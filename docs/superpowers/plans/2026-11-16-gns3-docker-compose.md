# GNS3 Server Docker Compose — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A `docker compose up -d` stack running `gns3/gns3-server:2.2.61`, Web UI reachable from Windows at `http://localhost:3080`, projects/images/templates persisted.

**Architecture:** Two files: a pinned compose service (privileged, `/dev/kvm`, port 3080 only) and a server config bind-mounted into the image's versioned config dir (`/root/.config/GNS3/2.2/`), which is also where the controller persists `gns3_controller.conf` (user templates). A `data/` bind mount holds `/root/GNS3`.

**Tech Stack:** Docker Compose v5.7.0, Docker Engine 29.9.3 (Docker Desktop WSL2 backend, linux/amd64), `gns3/gns3-server:2.2.61`.

**Spec:** `docs/superpowers/specs/2026-11-16-gns3-docker-compose-design.md`

## Global Constraints

- Working directory: `/home/eduardo/vscode/gns3` (NOT a git repo — no commits; verification steps are the task gates).
- Image tag pinned verbatim: `gns3/gns3-server:2.2.61`.
- Auth verbatim: `auth = True`, `user = gns3`, `password = gns3` (spec-approved; user may change later).
- Only port 3080 published. No console/VNC ranges.
- `privileged: true` and device `/dev/kvm:/dev/kvm` (verified present in WSL).
- Config search order demands the mount target `/root/.config/GNS3/2.2` (2.2.x versioned dir).
- Paths pinned verbatim: `images_path = /root/GNS3/images`, `projects_path = /root/GNS3/projects`, `appliances_path = /root/GNS3/appliances`, `symbols_path = /root/GNS3/symbols`.

## Review Focus

1. Unauthenticated non-local client must be refused; authenticated must pass → Task 3 Step 4 (`curl` without auth expects 401, with auth expects 200).
2. Server must load OUR config, not image defaults (silently disables auth otherwise) → Task 3 Step 3 (log line `Load configuration file /root/.config/GNS3/2.2/gns3_server.conf`).
3. `/dev/kvm` must reach the container or QEMU falls back to TCG (very slow) / errors → Task 3 Step 2 (no device-mapping error in logs).
4. Controller state must persist (`gns3_controller.conf` with user templates appears in `config/`) → Task 3 Step 5 (file exists after first run).
5. Server version must be 2.2.61 exactly → Task 3 Step 4 (JSON `"version": "2.2.61"`).

---

### Task 1: Server config file

**Files:**
- Create: `config/gns3_server.conf`

**Interfaces:**
- Produces: the config file consumed by the container at `/root/.config/GNS3/2.2/gns3_server.conf` (Task 2 mount), with the exact keys pinned in Global Constraints.

- [ ] **Step 1: Write `config/gns3_server.conf`**

Trimmed from upstream `conf/gns3_server.conf`; only keys that differ from code defaults or pin the spec:

```ini
[Server]
host = 0.0.0.0
port = 3080

auth = True
user = gns3
password = gns3

images_path = /root/GNS3/images
projects_path = /root/GNS3/projects
appliances_path = /root/GNS3/appliances
symbols_path = /root/GNS3/symbols

console_start_port_range = 5000
console_end_port_range = 10000
vnc_console_start_port_range = 5900
vnc_console_end_port_range = 10000
udp_start_port_range = 20000
udp_end_port_range = 30000

enable_builtin_templates = True
install_builtin_appliances = True

[Qemu]
enable_kvm = True
enable_hardware_acceleration = True
```

(Deliberately omitted: `hardware_virtualization_check` and `report_errors` — code defaults avoid spurious failures inside the docker-desktop VM; `allowed_interfaces`/`default_nat_interface` — no libvirt NAT cloud in this container.)

- [ ] **Step 2: Verify required keys are present**

Run: `grep -E '^(host|port|auth|user|password|projects_path) = ' config/gns3_server.conf`
Expected: 6 lines matching, including `auth = True` and `projects_path = /root/GNS3/projects`.

### Task 2: Compose file

**Files:**
- Create: `compose.yaml`

**Interfaces:**
- Consumes: `config/gns3_server.conf` from Task 1 (must exist; bind-mounted whole-dir).
- Produces: service name `gns3-server`, published port `3080`, mounts `./config` and `./data`.

- [ ] **Step 1: Write `compose.yaml`**

```yaml
services:
  gns3-server:
    image: gns3/gns3-server:2.2.61
    container_name: gns3-server
    restart: unless-stopped
    privileged: true
    devices:
      - /dev/kvm:/dev/kvm
    ports:
      - "3080:3080"
    volumes:
      - ./config:/root/.config/GNS3/2.2
      - ./data:/root/GNS3
```

- [ ] **Step 2: Validate**

Run: `docker compose config --quiet && echo VALID`
Expected: `VALID` (no errors; `./data` auto-created by the bind mount).

### Task 3: Launch and verify

**Files:** none created (operates on Tasks 1–2 outputs).

**Interfaces:**
- Consumes: the stack from Task 2; auth pair `gns3` / `gns3` from Task 1.

- [ ] **Step 1: Pull and start**

Run: `docker compose up -d`
Expected: image pulled, container `gns3-server` started.

- [ ] **Step 2: Container boots cleanly**

Run: `docker logs gns3-server 2>&1 | tail -30`
Expected: no `Permission denied` on `/dev/kvm`, no device-mapping error, no Python traceback.

- [ ] **Step 3: Config actually loaded**

Run: `docker logs gns3-server 2>&1 | grep 'Load configuration file'`
Expected: `Load configuration file /root/.config/GNS3/2.2/gns3_server.conf`

- [ ] **Step 4: API auth and version (Review Focus 1, 5)**

Run: `curl -s -o /dev/null -w '%{http_code}\n' http://localhost:3080/v2/version && curl -s -u gns3:gns3 http://localhost:3080/v2/version`
Expected: first command `401`; second a JSON object containing `"version": "2.2.61"`.

- [ ] **Step 5: Web UI served and controller state persisted (Review Focus 4)**

Run: `curl -s -o /dev/null -w '%{http_code}\n' http://localhost:3080/static/web-ui/web-ui/index.html; ls config/`
Expected: `200` (if 404, fetch `/` to locate the UI path and report — do not silently skip); `config/` now contains `gns3_controller.conf` alongside `gns3_server.conf`.

- [ ] **Step 6: Report to user**

Report: Web UI URL `http://localhost:3080` (auth `gns3`/`gns3`), how to stop (`docker compose down`), where projects live on the Windows side (`\\wsl.localhost\<distro>\home\eduardo\vscode\gns3\data`), and the Docker-nodes limitation from the spec.
