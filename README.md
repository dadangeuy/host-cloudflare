# Cloudflare

Self-hosted [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) (`cloudflared`) running in Docker on the `cloudflare-1` Proxmox host.

The tunnel shares the network namespace of an existing `tailscale` container, so services reachable over the tailnet can be exposed through Cloudflare without opening inbound ports.

## Repository layout

```
.
├── .github/workflows/
│   ├── gitguardian.yml       # Secret scanning on every push / PR
│   ├── proxmox-docker.yml    # Redeploy the stack on push to main
│   └── proxmox-system.yml    # Weekly OS upgrade + reboot of the host
└── docker/
    ├── .env                  # Compose project name and compose file
    ├── Makefile              # Bootstrap, lifecycle and secret helpers
    ├── compose.tunnel.yml    # cloudflared service definition
    ├── environment/          # Non-secret env files (default + per stage)
    └── secret/               # GPG-encrypted env files (*.env.gpg)
```

## Requirements

- Docker with the Compose plugin
- A running container named `tailscale` on the same host (the tunnel uses `network_mode: container:tailscale`)
- `gpg` and `openssl` for secret management
- [`ggshield`](https://github.com/GitGuardian/ggshield) (optional, for `make scan`)

## Configuration

Environment files are loaded in this order, later files overriding earlier ones:

1. `environment/tunnel.default.env`
2. `environment/tunnel.${STAGE}.env`
3. `storage/secret/tunnel.default.env` (decrypted)
4. `storage/secret/tunnel.${STAGE}.env` (decrypted)

`STAGE` defaults to `default`; set `STAGE=production` on the production host.

Secrets (e.g. `TUNNEL_TOKEN`) live only in the encrypted files under `docker/secret/`. They are decrypted into `docker/storage/secret/` using a symmetric key at `/etc/default/secret.key` (override with `SECRET_KEY=...`).

## Setup

All commands are run from the `docker/` directory.

```sh
cd docker

# 1. Install the shared secret key at /etc/default/secret.key
#    (or generate a new one for a fresh setup)
make create-secret-key

# 2. Create the storage directory, symlink it to ./storage and decrypt secrets
STORAGE_PATH=/path/to/storage make bootstrap

# 3. Start the tunnel
make start
```

## Make targets

| Target              | Description                                                                 |
| ------------------- | --------------------------------------------------------------------------- |
| `bootstrap`         | Create `$STORAGE_PATH`, symlink it to `./storage`, and decrypt secrets      |
| `start`             | `docker compose up -d --wait`                                               |
| `stop`              | `docker compose down`                                                       |
| `update`            | Stop, pull latest git + images, decrypt secrets, start, and prune Docker    |
| `listen`            | Follow container logs                                                       |
| `create-secret-key` | Generate a random key at `$SECRET_KEY`                                      |
| `encrypt-secret`    | Encrypt every file in `storage/secret/` into `secret/*.gpg` (AES256)        |
| `decrypt-secret`    | Decrypt every `secret/*.gpg` into `storage/secret/`                         |
| `scan`              | Scan the repository for leaked secrets with `ggshield`                      |

## Managing secrets

```sh
cd docker
make decrypt-secret              # refresh plaintext copies
$EDITOR storage/secret/tunnel.production.env
make encrypt-secret              # write back to secret/*.gpg
git add secret/ && git commit
```

Never commit anything under `storage/` — it is git-ignored.

## CI/CD

- **GitGuardian** — scans every push and pull request for leaked secrets.
- **Proxmox (Docker)** — on push to `main`, connects to the tailnet and runs `make update` on `cloudflare-1`.
- **Proxmox (System)** — every Tuesday at 00:00 UTC (or manually), runs `apt full-upgrade` and reboots `cloudflare-1`.

The deploy workflows use the `proxmox` GitHub environment and require these secrets: `TS_OAUTH_CLIENT_ID`, `TS_AUDIENCE`, `SSH_PASSWORD`. Secret scanning requires `GITGUARDIAN_API_KEY`.
