# Secrets to .env — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** GNS3 auth credentials live only in an untracked `.env`; the tracked repo carries placeholders only (`.env.example`, config template).

**Architecture:** GNS3 2.2 reads auth only from its config file, so compose overrides the container command to render `config/gns3_server.conf` (untracked) from `config/gns3_server.conf.template` (tracked, `@GNS3_USER@`/`@GNS3_PASSWORD@` tokens) into the already-mounted versioned config dir, then `exec`s the server. Render uses Python literal replace — any password characters are safe.

**Tech Stack:** Docker Compose v5.7.0 (`${VAR:?}` interpolation), gns3/gns3-server:2.2.61 (python3 in image), git.

**Spec:** Approved design in conversation 2026-11-16 ("Design: secrets via .env"); supersedes the port/auth details of `docs/superpowers/specs/2026-11-16-gns3-docker-compose-design.md` where they conflict (secrets handling only; port stays loopback-only per commit 9f3116a).

## Global Constraints

- Working directory: `/home/eduardo/vscode/gns3`.
- Credential values start as `GNS3_USER=gns3`, `GNS3_PASSWORD=gns3` (behavior-preserving; user changes later in `.env` only).
- `.env` must NEVER be tracked; `config/gns3_server.conf` (rendered) must NEVER be tracked.
- Port binding stays `127.0.0.1:3080:3080`.
- Container command must end with `exec python3 -m gns3server --port 3080` (PID 1 keeps signal handling).
- TDD config-file exception applies (ruling carried from previous cycle).

## Review Focus

1. No secret material in tracked files or history additions → Task 1 Step 3 + Task 3 Step 4 (`git ls-files` grep).
2. Missing `.env` must fail fast with a clear message, not silently run with defaults → Task 2 Step 3 (`docker compose config` errors).
3. Rendered config must contain no leftover `@GNS3_` tokens → Task 3 Step 2.
4. Auth must still work after the command override, from both WSL and Windows → Task 3 Steps 3–5.
5. Controller-template persistence must survive the restructure (`gns3_controller.conf` still in `config/`) → Task 3 Step 5.

---

### Task 1: Secret files, template, ignore rules

**Files:**
- Create: `.gitignore`
- Create: `.env.example`
- Create: `.env` (untracked, by design)
- Create: `config/gns3_server.conf.template`
- Modify (untrack): `config/gns3_server.conf`

**Interfaces:**
- Produces: token template consumed by Task 2's render command; `.env` variables `GNS3_USER`, `GNS3_PASSWORD` consumed by compose interpolation.

- [ ] **Step 1: Write `.gitignore`**

```gitignore
# secrets & runtime state (never commit)
.env
data/
config/gns3_server.conf
config/gns3_controller.conf
```

- [ ] **Step 2: Write `.env.example` and `.env`**

`.env.example` (tracked):

```bash
# Copy to .env and set real values before `docker compose up -d`.
# Used by compose to render config/gns3_server.conf at container start.
GNS3_USER=gns3
GNS3_PASSWORD=change-me
```

`.env` (untracked): same two variables with values `gns3` / `gns3`.

- [ ] **Step 3: Write template, untrack rendered conf**

Create `config/gns3_server.conf.template`: exactly the current committed `config/gns3_server.conf` with `user = gns3` → `user = @GNS3_USER@` and `password = gns3` → `password = @GNS3_PASSWORD@`. Then `git rm --cached config/gns3_server.conf` (file stays on disk; it becomes the rendered artifact).

- [ ] **Step 4: Verify ignore rules and tracking**

Run: `git check-ignore .env data/ config/gns3_server.conf config/gns3_controller.conf && git ls-files | grep -E '\.env$|gns3_server\.conf$|data/' ; git ls-files | grep -E 'template|example'`
Expected: check-ignore names all four paths; first grep exits 1 (nothing tracked); second grep lists `.env.example` and `config/gns3_server.conf.template`.

### Task 2: Compose render command

**Files:**
- Modify: `compose.yaml`

**Interfaces:**
- Consumes: `.env` interpolation vars and the template mount from Task 1.
- Produces: rendered `config/gns3_server.conf` (Task 3 verification target); server process as PID 1.

- [ ] **Step 1: Edit `compose.yaml`**

Add to the `gns3-server` service (keep all existing keys):

```yaml
    environment:
      GNS3_USER: ${GNS3_USER:?copy .env.example to .env and set real values}
      GNS3_PASSWORD: ${GNS3_PASSWORD:?copy .env.example to .env and set real values}
    volumes:
      - ./config/gns3_server.conf.template:/srv/gns3_server.conf.template:ro
      - ./config:/root/.config/GNS3/2.2
      - ./data:/root/GNS3
    entrypoint: ["/bin/sh", "-c"]
    command:
      - |
        set -e
        python3 -c "import os, pathlib; src = pathlib.Path('/srv/gns3_server.conf.template').read_text(); out = src.replace('@GNS3_USER@', os.environ['GNS3_USER']).replace('@GNS3_PASSWORD@', os.environ['GNS3_PASSWORD']); pathlib.Path('/root/.config/GNS3/2.2/gns3_server.conf').write_text(out)"
        exec python3 -m gns3server --port 3080
```

- [ ] **Step 2: Validate resolved config**

Run: `docker compose config --quiet && echo VALID && docker compose config | grep -E 'GNS3_(USER|PASSWORD):'`
Expected: `VALID`; both variables resolved to `.env` values.

- [ ] **Step 3: Verify fail-fast without .env**

Run: `docker compose --env-file /nonexistent config --quiet 2>&1 | tail -1`
Expected: error mentioning `GNS3_USER` and the guidance text (compose refuses to start with defaults).

- [ ] **Step 4: Commit**

```bash
git add .gitignore .env.example config/gns3_server.conf.template compose.yaml
git commit -m "feat: render GNS3 credentials from .env at container start"
```

### Task 3: Restart and verify end-to-end

**Files:** none created.

- [ ] **Step 1: Recreate the stack**

Run: `docker compose up -d` (recreates container with new command; if `ports are not available` appears, run once more — known transient).

- [ ] **Step 2: Rendered config is token-free**

Run: `grep -c '@GNS3_' config/gns3_server.conf; grep 'password = ' config/gns3_server.conf`
Expected: `0`; password line shows the `.env` value.

- [ ] **Step 3: Auth live on WSL path**

Run: `curl -s -u "$GNS3_USER:$GNS3_PASSWORD" -o /dev/null -w '%{http_code}\n' http://localhost:3080/v2/version` (source .env values) and logs `docker logs gns3-server 2>&1 | grep -i 'authentication'`
Expected: `200`; log shows `HTTP authentication is enabled with username '<GNS3_USER value>'`.

- [ ] **Step 4: No secrets in tracked files**

Run: `git grep -l 'password' -- . | grep -v template; git ls-files | grep -E '\.env$'`
Expected: only `config/gns3_server.conf.template` and `.env.example` may reference passwords; first grep exits 1 (no other tracked file contains one); second exits 1.

- [ ] **Step 5: Windows path + persistence intact**

Run: `/mnt/c/Windows/System32/curl.exe -s -o NUL -w '%{http_code}\n' http://localhost:3080/v2/version; ls config/`
Expected: `401` (reachable, auth gate); `config/` still holds `gns3_controller.conf`.

- [ ] **Step 6: Report**

Summarize for the user: what moved where, how to change credentials now, restart idempotency note.
