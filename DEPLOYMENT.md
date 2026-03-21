# Deployment Instructions

## Overview

This repository deploys a Docker Swarm stack with Traefik as the reverse proxy, along with Prometheus node-exporter for host metrics. It is designed to work both with a real public domain (using Let's Encrypt certificates) and without one (see below).

A second stack — the Randovania server — is deployed separately from `randovania/tools/server-docker/docker-compose.yml` and depends on the Traefik stack being up first.

Deployments are run **locally** using `docker stack deploy` pointed at the remote host via a Docker SSH context. No preprocessing is needed on the target machine.

---

## Prerequisites

### Packages on the target host

```sh
sudo apt-get install -y docker.io
sudo usermod -aG docker $USER
```

### Local machine

Docker must be installed. Verify with `docker --version`.

Add the target host to `~/.ssh/config` so Docker's SSH transport can authenticate:

```
Host 192.168.5.125
    User randovania
    IdentityFile /path/to/admin.key
    StrictHostKeyChecking no
```

Create a Docker context pointing at the remote swarm (one-time):

```sh
docker context create randovania-server \
  --description "Randovania Docker Swarm" \
  --docker "host=ssh://randovania@192.168.5.125"
```

---

## One-time swarm setup on the target host

```sh
ssh randovania@192.168.5.125

# Initialise the swarm
sudo docker swarm init --advertise-addr 192.168.5.125

# Label the node
NODE_ID=$(sudo docker node ls -q)
sudo docker node update --label-add traefik-public.traefik-public-certificates=true $NODE_ID
sudo docker node update --label-add randovania-allow-production=true $NODE_ID

# Create the shared overlay network
sudo docker network create --driver overlay traefik-public
```

---

## Deploying the Traefik stack

### 1. Generate a dashboard password hash

```sh
htpasswd -nbB admin "your-password" | cut -d: -f2
# Outputs something like: $2y$05$...
```

### 2. Create `.env`

```
DOMAIN=<your-domain>
DASHBOARD_PASSWORD='<hash from above>'
LETSENCRYPT_EMAIL=admin@example.com
```

> **Important:** wrap `DASHBOARD_PASSWORD` in **single quotes** in the `.env` file. The bcrypt hash contains `$` characters that bash would otherwise expand when sourcing.

### 3. Deploy

```sh
set -a && source .env && set +a
docker --context randovania-server stack deploy -c docker-compose.yml traefik
```

Variable substitution happens locally; the resolved stack definition is sent to the remote daemon over SSH.

### 4. Verify

```sh
docker --context randovania-server stack services traefik
# Both services should show 1/1 replicas

curl -sk -u admin:your-password -H "Host: <DOMAIN>" \
  -o /dev/null -w "%{http_code}" https://192.168.5.125/dashboard/
# Expect: 200
```

---

## Deploying the Randovania stack

### 1. Create the Docker secret on the target host

```sh
ssh randovania@192.168.5.125

python3 -c "
import json, base64, os
config = {
    'discord_client_id': 618134325921316864,
    'server_address': 'https://<DOMAIN>/randovania',
    'socketio_path': '/randovania/socket.io',
    'server_config': {
        'fernet_key': base64.urlsafe_b64encode(os.urandom(32)).decode(),
        'secret_key': base64.urlsafe_b64encode(os.urandom(32)).decode(),
        'discord_client_secret': '',
        'database_path': '/data/data.db',
        'client_version_checking': 'ignore'
    },
    'discord_bot': {
        'token': 'REPLACE_WITH_REAL_DISCORD_BOT_TOKEN'
    }
}
print(json.dumps(config))
" | sudo docker secret create randovania_production_configuration -
```

> Replace `discord_client_secret` and `discord_bot.token` with real values when available. Without a valid bot token the `bot` service will restart-loop with a Discord auth error — this is expected.

### 2. Create `.env` in the `server-docker` directory

```
VERSION=v10.5.0
SERVER_ENVIRONMENT=production
DOMAIN=<your-domain>
DOMAIN2=rdv.<your-domain>
PATH_PREFIX=randovania
OBFUSCATOR_SECRET=
```

### 3. Deploy

```sh
cd path/to/randovania/tools/server-docker
set -a && source .env && set +a
docker --context randovania-server stack deploy -c docker-compose.yml randovania
```

### 4. Verify

```sh
docker --context randovania-server stack services randovania
# randovania and sqlite-web should show 1/1 replicas

curl -sk -H "Host: <DOMAIN>" -o /dev/null -w "%{http_code}" \
  https://192.168.5.125/randovania/
# Expect: 200 (returns server version string)
```

---

## Deploying without a publicly registered domain

**No changes to any configuration file are required.**

### Why it works

The stacks use `certresolver=le` on all HTTPS routers, which instructs Traefik to obtain certificates from Let's Encrypt. When the ACME TLS challenge cannot be completed — because the host is on a private IP not reachable from the internet — Traefik fails the challenge, retries in the background, and **automatically falls back to its built-in self-signed certificate** for all affected routes. HTTPS continues to work normally; clients will see a certificate warning that can be bypassed.

The only requirement is that `LETSENCRYPT_EMAIL` is set to any value (it satisfies the `?Variable not set` guard in the compose file; the address is never actually contacted).

### Choosing a domain without registration

Use [nip.io](https://nip.io): `<anything>.<IP>.nip.io` resolves to `<IP>` via public DNS. No account or registration needed.

```
192.168.5.125.nip.io      →  192.168.5.125
rdv.192.168.5.125.nip.io  →  192.168.5.125
```

### `.env` example for a private deployment

```
# server-traefik/.env
DOMAIN=192.168.5.125.nip.io
DASHBOARD_PASSWORD='$2y$05$...'
LETSENCRYPT_EMAIL=admin@example.com

# server-docker/.env
VERSION=v10.5.0
SERVER_ENVIRONMENT=production
DOMAIN=192.168.5.125.nip.io
DOMAIN2=rdv.192.168.5.125.nip.io
PATH_PREFIX=randovania
OBFUSCATOR_SECRET=
```

Once a real public domain is available, update `DOMAIN` and `LETSENCRYPT_EMAIL` and redeploy. Traefik will obtain a valid Let's Encrypt certificate automatically.
