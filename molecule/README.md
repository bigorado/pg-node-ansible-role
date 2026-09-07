# Molecule tests

Two scenarios share one converge/verify playbook (`molecule/shared/`).

| Scenario | Driver | Target | Use |
|---|---|---|---|
| `default` | docker | throwaway `geerlingguy/docker-ubuntu2404-ansible` container (privileged, docker-in-docker) | local / CI |
| `delegated` | default | a real, already-provisioned host (default `deploy@192.168.50.126`) | integration on real hardware |

Both cover `pg_node_deploy_mode=docker` (default) and `pg_node_deploy_mode=systemd`
(set `PG_NODE_DEPLOY_MODE=systemd`).

## Requirements

```bash
pip install "molecule>=6" "molecule-plugins[docker]" ansible-lint
```

## Run

```bash
# containerised, full lifecycle (create → converge → idempotence → verify → destroy)
molecule test -s default
PG_NODE_DEPLOY_MODE=systemd molecule test -s default

# against the real host — applies the role for real, does NOT auto-destroy
molecule converge    -s delegated
molecule idempotence -s delegated
molecule verify      -s delegated
molecule destroy     -s delegated          # rolls the node back off the host

# override the target
PG_NODE_TEST_HOST=10.0.0.5 PG_NODE_TEST_USER=root molecule converge -s delegated
```

## What `verify` asserts

- `/opt/pg-node/.env` exists and has a valid `SERVICE_PORT`, UUID `API_KEY`, `SSL_CERT_FILE`
- node certificate + key exist, non-empty, carry `localhost` / `127.0.0.1` SAN
- `geoip.dat` + `geosite.dat` present in `/var/lib/pg-node/xray-assets` and > 100 KiB
- `.env` sets `XRAY_ASSETS_PATH` to the geodata dir
- node port `62050` is listening
- **docker mode:** `docker-compose.yml` present, container `node` `running`, `/usr/local/bin/pg-node` executable
- **systemd mode:** `pg-node.service` unit + `/opt/pg-node/pasarguard-node` present, `systemctl is-active pg-node` == `active`, `/usr/local/bin/xray` installed
