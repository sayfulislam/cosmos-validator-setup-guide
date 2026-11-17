# Cosmos Validator Infrastructure with Docker — One Role per Server

A step-by-step guide to build a full Cosmos SDK validator setup where every part runs on its own server, inside Docker.

The examples use **Cosmos Hub** (`gaiad`, chain `cosmoshub-4`). For another Cosmos SDK chain, change the values in the `.env` file (binary name, chain ID, token, download links). The steps stay the same.

> **Practice first.** Run the whole guide on a testnet before you touch mainnet. A mistake with the signing key on mainnet can cost real money (see "Double signing" in Part 12).

---

## Contents

0. The plan: servers, IPs, ports
1. Prepare every server
2. Private network (WireGuard)
3. Firewall
4. Build the node Docker image
5. Sentry nodes (sentry-1, sentry-2)
6. Validator node
7. Remote signer (TMKMS)
8. Create the validator on chain
9. Public RPC / API node
10. Monitoring server
11. Daily work: upgrades, backups, checks
12. Safety rules and common problems

---

## 0. The plan

### 0.1 Servers

| # | Name | Job | Public internet? | Private IP (WireGuard) |
|---|------|-----|------------------|------------------------|
| 1 | `mon` | Prometheus, Grafana, Alertmanager, Tenderduty | Only Grafana (HTTPS) | `10.10.0.1` |
| 2 | `signer` | TMKMS — holds the validator signing key | No | `10.10.0.2` |
| 3 | `val` | Validator node — makes and signs blocks | No | `10.10.0.3` |
| 4 | `sentry1` | Full node — talks to the public network | Yes, P2P only | `10.10.0.4` |
| 5 | `sentry2` | Full node — second sentry | Yes, P2P only | `10.10.0.5` |
| 6 | `rpc` | Full node with public RPC / API / gRPC | Yes, HTTPS | `10.10.0.6` |

### 0.2 How they talk

```
                     Internet (other chain nodes, users)
                        |                 |              |
                   P2P 26656         P2P 26656       HTTPS 443
                        |                 |              |
                   +---------+       +---------+     +-------+
                   | sentry1 |<----->| sentry2 |<--->|  rpc  |
                   +---------+       +---------+     +-------+
                        \               /
                         \  WireGuard  /       (private network 10.10.0.0/24)
                          +-----------+
                          |    val    |   <- never open to the internet
                          +-----------+
                                ^
                                | signs blocks (port 26659)
                          +-----------+
                          |  signer   |   <- never open to the internet
                          +-----------+

        mon (10.10.0.1) reads metrics from every server over WireGuard
```

Why this shape:

- The **validator** only knows the two sentries. Attackers on the internet cannot see its IP, so they cannot flood it.
- The **signing key** is not on the validator. If the validator server is broken into, the key is still safe on `signer`.
- The **RPC node** takes all public traffic, so heavy users cannot slow down the validator.

### 0.3 Ports

| Port | Used for | Open on |
|------|----------|---------|
| 22/tcp | SSH | all servers (limit to your own IP if you can) |
| 51820/udp | WireGuard | all servers |
| 26656/tcp | P2P | public on `sentry1`, `sentry2`, `rpc`; private only on `val` |
| 26657/tcp | RPC | private on sentries; `127.0.0.1` on `rpc` (behind Caddy) |
| 26659/tcp | Remote signer | `val` only, from `signer` only |
| 26660/tcp | Node metrics | private, from `mon` only |
| 9100/tcp | Server metrics (node_exporter) | private, from `mon` only |
| 1317, 9090 | REST API, gRPC | `127.0.0.1` on `rpc` (behind Caddy) |
| 80, 443/tcp | HTTPS | `rpc` and `mon` |

### 0.4 Server size (rough guide)

These are starting points, not official numbers. Check the chain's own docs before you buy.

| Server | CPU | RAM | Disk |
|--------|-----|-----|------|
| `val`, `sentry1`, `sentry2` | 8 cores | 32 GB | 1 TB NVMe SSD |
| `rpc` | 8 cores | 32–64 GB | 1–2 TB NVMe SSD |
| `signer` | 2 cores | 2–4 GB | 20 GB |
| `mon` | 2–4 cores | 8 GB | 100 GB |

OS for all: Ubuntu 22.04 or 24.04 LTS.

### 0.5 Order of work

1. Parts 1–3 on **all six** servers.
2. Part 4 + 5 on the sentries. Wait until they are synced.
3. Part 6 on the validator. Wait until it is synced.
4. Part 7 on the signer. Move the key.
5. Part 8: create the validator on chain.
6. Part 9 (RPC) and Part 10 (monitoring) can be done at any time after Part 3.

---

## 1. Prepare every server

Do this on all six servers.

### 1.1 Update and set the name

```bash
sudo apt update && sudo apt -y upgrade
sudo apt -y install curl jq nano ufw wireguard chrony lz4 git
sudo hostnamectl set-hostname val        # use: mon, signer, val, sentry1, sentry2, rpc
sudo timedatectl set-timezone UTC
sudo systemctl enable --now chrony       # correct clock is a must for a validator
```

### 1.2 Make a work user and lock down SSH

```bash
sudo adduser --disabled-password --gecos "" cosmos
sudo usermod -aG sudo cosmos
sudo mkdir -p /home/cosmos/.ssh
sudo cp ~/.ssh/authorized_keys /home/cosmos/.ssh/
sudo chown -R cosmos:cosmos /home/cosmos/.ssh && sudo chmod 700 /home/cosmos/.ssh
echo "cosmos ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/cosmos
```

Test that you can log in as `cosmos` with your SSH key **before** the next step. Then turn off password login:

```bash
sudo nano /etc/ssh/sshd_config.d/99-hardening.conf
```

```
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
```

```bash
sudo systemctl restart ssh
```

### 1.3 Install Docker

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker cosmos
```

Limit log size so logs do not fill the disk:

```bash
sudo nano /etc/docker/daemon.json
```

```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "100m", "max-file": "5" }
}
```

```bash
sudo systemctl restart docker
```

Log out and in again, then check:

```bash
docker run --rm hello-world
docker compose version
```

### 1.4 Folders

```bash
sudo mkdir -p /opt/cosmos && sudo chown cosmos:cosmos /opt/cosmos
```

Everything for this guide lives under `/opt/cosmos`.

### 1.5 System limits (node servers only: val, sentry1, sentry2, rpc)

```bash
echo "cosmos soft nofile 65535
cosmos hard nofile 65535" | sudo tee /etc/security/limits.d/cosmos.conf
```

---

## 2. Private network (WireGuard)

All servers join one private network, `10.10.0.0/24`. Private traffic (validator ↔ sentries, signer ↔ validator, metrics) only goes through it.

### 2.1 Make keys (on every server)

```bash
wg genkey | sudo tee /etc/wireguard/private.key | wg pubkey | sudo tee /etc/wireguard/public.key
sudo chmod 600 /etc/wireguard/private.key
sudo cat /etc/wireguard/public.key
```

Write down each server's **public key** and **public IP** in a table like this:

| Name | Private IP | Public IP | WireGuard public key |
|------|-----------|-----------|----------------------|
| mon | 10.10.0.1 | `<MON_PUBLIC_IP>` | `<MON_PUBKEY>` |
| signer | 10.10.0.2 | `<SIGNER_PUBLIC_IP>` | `<SIGNER_PUBKEY>` |
| val | 10.10.0.3 | `<VAL_PUBLIC_IP>` | `<VAL_PUBKEY>` |
| sentry1 | 10.10.0.4 | `<S1_PUBLIC_IP>` | `<S1_PUBKEY>` |
| sentry2 | 10.10.0.5 | `<S2_PUBLIC_IP>` | `<S2_PUBKEY>` |
| rpc | 10.10.0.6 | `<RPC_PUBLIC_IP>` | `<RPC_PUBKEY>` |

### 2.2 Config file (on every server)

```bash
sudo nano /etc/wireguard/wg0.conf
```

Example for **val**. On each server: change `Address` to its own private IP, put its own private key, and **remove its own `[Peer]` block**.

```ini
[Interface]
Address = 10.10.0.3/24
ListenPort = 51820
PrivateKey = <THIS_SERVER_PRIVATE_KEY>

[Peer]
# mon
PublicKey = <MON_PUBKEY>
Endpoint = <MON_PUBLIC_IP>:51820
AllowedIPs = 10.10.0.1/32
PersistentKeepalive = 25

[Peer]
# signer
PublicKey = <SIGNER_PUBKEY>
Endpoint = <SIGNER_PUBLIC_IP>:51820
AllowedIPs = 10.10.0.2/32
PersistentKeepalive = 25

[Peer]
# val
PublicKey = <VAL_PUBKEY>
Endpoint = <VAL_PUBLIC_IP>:51820
AllowedIPs = 10.10.0.3/32
PersistentKeepalive = 25

[Peer]
# sentry1
PublicKey = <S1_PUBKEY>
Endpoint = <S1_PUBLIC_IP>:51820
AllowedIPs = 10.10.0.4/32
PersistentKeepalive = 25

[Peer]
# sentry2
PublicKey = <S2_PUBKEY>
Endpoint = <S2_PUBLIC_IP>:51820
AllowedIPs = 10.10.0.5/32
PersistentKeepalive = 25

[Peer]
# rpc
PublicKey = <RPC_PUBKEY>
Endpoint = <RPC_PUBLIC_IP>:51820
AllowedIPs = 10.10.0.6/32
PersistentKeepalive = 25
```

### 2.3 Start and test

```bash
sudo chmod 600 /etc/wireguard/wg0.conf
sudo systemctl enable --now wg-quick@wg0
sudo wg show                      # each peer should show "latest handshake"
for i in 1 2 3 4 5 6; do ping -c1 -W2 10.10.0.$i >/dev/null && echo "10.10.0.$i ok" || echo "10.10.0.$i FAIL"; done
```

Do not go on until every server can ping every other server.

---

## 3. Firewall

> **Important Docker point.** Ports opened with Docker's `ports:` setting skip UFW rules. In this guide every container uses `network_mode: host`, so the container uses the server's own network and **UFW rules do apply**. Keep it that way.

### 3.1 Same on every server

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 51820/udp
sudo ufw allow from 10.10.0.1 to any port 9100 proto tcp     # server metrics, from mon
```

### 3.2 Extra rules per server

**sentry1 and sentry2**

```bash
sudo ufw allow 26656/tcp                                        # public P2P
sudo ufw allow from 10.10.0.0/24 to any port 26657 proto tcp    # private RPC
sudo ufw allow from 10.10.0.1 to any port 26660 proto tcp       # metrics
```

**val**

```bash
sudo ufw allow from 10.10.0.4 to any port 26656 proto tcp       # sentry1
sudo ufw allow from 10.10.0.5 to any port 26656 proto tcp       # sentry2
sudo ufw allow from 10.10.0.2 to any port 26659 proto tcp       # signer
sudo ufw allow from 10.10.0.1 to any port 26657 proto tcp       # tenderduty on mon
sudo ufw allow from 10.10.0.1 to any port 26660 proto tcp       # metrics
```

**signer** — nothing extra. It only makes outgoing connections.

**rpc**

```bash
sudo ufw allow 26656/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow from 10.10.0.1 to any port 26660 proto tcp
```

**mon**

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
```

### 3.3 Turn it on (every server)

```bash
sudo ufw --force enable
sudo ufw status verbose
```

---

## 4. Build the node Docker image

Do this on the four node servers: `sentry1`, `sentry2`, `val`, `rpc`.

We build a small image from the **official release binary**. This works for any chain, and you always know which version is inside.

> If the chain team publishes a ready-made image you trust, you can use it instead: set `image:` in `docker-compose.yml` and delete the `build:` block. Check which user and home folder that image uses, because they may differ from this guide.

### 4.1 Find the right version

The version must match what the network runs **now**. Ask a public node:

```bash
curl -s https://cosmos-rpc.polkachu.com/abci_info | jq -r .result.response.version
```

Compare with the chain's GitHub releases page (`github.com/cosmos/gaia/releases`). I did not put a version number in this guide on purpose, because it changes with every chain upgrade.

### 4.2 Settings file

```bash
mkdir -p /opt/cosmos/node && cd /opt/cosmos/node
nano .env
```

```bash
# ---- chain settings (change these for another chain) ----
CHAIN_ID=cosmoshub-4
DAEMON=gaiad
VERSION=vX.Y.Z                      # <- put the version from step 4.1
BINARY_URL=https://github.com/cosmos/gaia/releases/download/vX.Y.Z/gaiad-vX.Y.Z-linux-amd64
DENOM=uatom
MIN_GAS=0.005uatom
# ---- this server ----
MONIKER=my-sentry-1                 # a name for this node
```

Open the releases page and copy the exact link of the `linux-amd64` file into `BINARY_URL`. Also note its SHA256 value from the checksum file on the same page.

### 4.3 Dockerfile

```bash
nano Dockerfile
```

```dockerfile
FROM debian:bookworm-slim

ARG BINARY_URL
ARG DAEMON

RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates curl jq lz4 \
 && rm -rf /var/lib/apt/lists/*

RUN curl -fL -o /usr/local/bin/${DAEMON} "${BINARY_URL}" \
 && chmod +x /usr/local/bin/${DAEMON}

# same user id as the "cosmos" user on the server, so file owners match
RUN useradd -u 1000 -m -d /node node
USER node
WORKDIR /node
```

Check the user id on the server with `id -u cosmos`. If it is not `1000`, change `-u 1000` to that number.

### 4.4 Compose file

```bash
nano docker-compose.yml
```

```yaml
services:
  node:
    build:
      context: .
      args:
        BINARY_URL: ${BINARY_URL}
        DAEMON: ${DAEMON}
    image: local/${DAEMON}:${VERSION}
    container_name: node
    network_mode: host
    restart: unless-stopped
    stop_grace_period: 1m
    volumes:
      - ./data:/node/data
    command: ["${DAEMON}", "start", "--home", "/node/data"]
    ulimits:
      nofile: { soft: 65535, hard: 65535 }
```

### 4.5 Build and check

```bash
mkdir -p data
docker compose build
docker compose run --rm node sh -c '$0 version --long | head -5' gaiad
docker compose run --rm node sha256sum /usr/local/bin/gaiad
```

- The version must match step 4.1.
- The SHA256 must match the value on the releases page. If it does not, stop and download again.
- If `version` fails with a "not found" or library error, the release file is not a self-contained binary. Use the chain's official image or build from source instead.

### 4.6 A short command

Typing the full command each time is slow. Add a helper:

```bash
echo "alias nd='docker compose -f /opt/cosmos/node/docker-compose.yml --env-file /opt/cosmos/node/.env run --rm node gaiad --home /node/data'" >> ~/.bashrc
source ~/.bashrc
nd version
```

`nd` runs any `gaiad` command in a short-lived container that uses the same data folder.

---

## 5. Sentry nodes

Do all of Part 5 on `sentry1`, then again on `sentry2`.

### 5.1 Create the node files

```bash
cd /opt/cosmos/node
source .env
nd init "$MONIKER" --chain-id "$CHAIN_ID"
```

This makes `data/config/` (settings) and `data/data/` (chain data).

### 5.2 Genesis file

```bash
curl -fL https://github.com/cosmos/mainnet/raw/master/genesis/genesis.cosmoshub-4.json.gz | gunzip > data/config/genesis.json
jq -r .chain_id data/config/genesis.json        # must print cosmoshub-4
```

For another chain or a testnet, take the genesis link from that chain's official repo.

### 5.3 Get this node's ID

```bash
nd comet show-node-id
```

Run this on **every** node server as you set it up and save the result. You will need these IDs:

| Name | Node ID |
|------|---------|
| sentry1 | `<S1_ID>` |
| sentry2 | `<S2_ID>` |
| val | `<VAL_ID>` (you get this in Part 6) |
| rpc | `<RPC_ID>` (you get this in Part 9) |

Tip: set up `val` up to step 6.1 early, just to get `<VAL_ID>`, then come back.

### 5.4 Edit `config.toml`

```bash
nano data/config/config.toml
```

Change these lines (example for **sentry1**; on sentry2 swap the IPs and IDs):

```toml
# --- [rpc] ---
laddr = "tcp://10.10.0.4:26657"            # private RPC only (sentry2: 10.10.0.5)

# --- [p2p] ---
laddr = "tcp://0.0.0.0:26656"
external_address = "<S1_PUBLIC_IP>:26656"
seeds = "<SEEDS_FROM_CHAIN_REGISTRY>"
persistent_peers = "<VAL_ID>@10.10.0.3:26656,<S2_ID>@10.10.0.5:26656"
pex = true
addr_book_strict = false                   # needed because we use private IPs
private_peer_ids = "<VAL_ID>"              # never tell others about the validator
unconditional_peer_ids = "<VAL_ID>"        # always keep a slot for the validator
max_num_inbound_peers = 80
max_num_outbound_peers = 40

# --- [instrumentation] ---
prometheus = true
prometheus_listen_addr = "10.10.0.4:26660" # sentry2: 10.10.0.5
```

Where to find seeds and public peers: the chain registry file `github.com/cosmos/chain-registry/blob/master/cosmoshub/chain.json`, under `peers`.

The two most important lines for a sentry are `private_peer_ids` and `pex = true`. The first hides the validator. The second lets the sentry find many public peers.

### 5.5 Edit `app.toml`

```bash
nano data/config/app.toml
```

```toml
minimum-gas-prices = "0.005uatom"

pruning = "custom"
pruning-keep-recent = "1000"
pruning-interval = "10"

[api]
enable = false

[grpc]
enable = false

[state-sync]
snapshot-interval = 1000        # sentries can serve snapshots to your other nodes
snapshot-keep-recent = 2
```

### 5.6 Fast sync with state sync

Without this, the node must replay the chain from the first block, which takes a very long time. State sync downloads a recent copy of the state instead.

```bash
cd /opt/cosmos/node
SNAP_RPC="https://cosmos-rpc.polkachu.com:443"     # any trusted RPC that supports state sync

LATEST=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height)
TRUST_HEIGHT=$((LATEST - 2000))
TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$TRUST_HEIGHT" | jq -r .result.block_id.hash)
echo "$LATEST $TRUST_HEIGHT $TRUST_HASH"           # all three must have a value

sed -i -E "/^\[statesync\]/,/^\[/ {
  s|^(enable[[:space:]]*=[[:space:]]*).*|\1true|
  s|^(rpc_servers[[:space:]]*=[[:space:]]*).*|\1\"$SNAP_RPC,$SNAP_RPC\"|
  s|^(trust_height[[:space:]]*=[[:space:]]*).*|\1$TRUST_HEIGHT|
  s|^(trust_hash[[:space:]]*=[[:space:]]*).*|\1\"$TRUST_HASH\"|
}" data/config/config.toml

grep -A12 '^\[statesync\]' data/config/config.toml | grep -E '^(enable|rpc_servers|trust_)'
```

The public RPC above is run by a third party. It is an example. Use one you trust, and later use **your own sentries** as the state sync source for `val` and `rpc`.

Other way: download a snapshot file from a snapshot provider and unpack it into `data/data/`. Use this if state sync cannot find a snapshot.

### 5.7 Start

```bash
docker compose up -d
docker compose logs -f --tail 50
```

You should see "Discovered new snapshot", then "Snapshot restored", then block heights going up.

### 5.8 Check sync

```bash
curl -s http://10.10.0.4:26657/status | jq '.result.sync_info | {latest_block_height, catching_up}'
curl -s http://10.10.0.4:26657/net_info | jq -r .result.n_peers
```

- `catching_up: false` means the node is fully synced.
- Peers should be more than 5.

After it is synced, turn state sync off so a restart does not try it again:

```bash
sed -i -E '/^\[statesync\]/,/^\[/ s|^(enable[[:space:]]*=[[:space:]]*).*|\1false|' data/config/config.toml
```

**Repeat Part 5 on sentry2** with its own IPs (`10.10.0.5`, `<S2_PUBLIC_IP>`) and with `<S1_ID>@10.10.0.4:26656` in `persistent_peers`.

---

## 6. Validator node

Do this on `val`. First do Part 4 there (set `MONIKER` to your public validator name).

At this stage the server is just a normal full node. It becomes a real validator only in Part 8.

### 6.1 Create the node files

```bash
cd /opt/cosmos/node
source .env
nd init "$MONIKER" --chain-id "$CHAIN_ID"
curl -fL https://github.com/cosmos/mainnet/raw/master/genesis/genesis.cosmoshub-4.json.gz | gunzip > data/config/genesis.json
nd comet show-node-id            # this is <VAL_ID> - put it in both sentries' config.toml
```

If you added `<VAL_ID>` to the sentries only now, restart them: `docker compose restart` on each sentry.

### 6.2 Edit `config.toml`

```bash
nano data/config/config.toml
```

```toml
# --- [rpc] ---
laddr = "tcp://10.10.0.3:26657"

# --- [p2p] ---
laddr = "tcp://10.10.0.3:26656"            # listen on the private network ONLY
external_address = ""
seeds = ""
persistent_peers = "<S1_ID>@10.10.0.4:26656,<S2_ID>@10.10.0.5:26656"
pex = false                                # do not look for other peers
addr_book_strict = false

# --- [mempool] / tx_index ---
# under [tx_index]:
indexer = "null"                           # validator does not need to index

# --- [instrumentation] ---
prometheus = true
prometheus_listen_addr = "10.10.0.3:26660"
```

`pex = false` plus only the two sentries in `persistent_peers` is what keeps the validator hidden.

### 6.3 Edit `app.toml`

```toml
minimum-gas-prices = "0.005uatom"

pruning = "custom"
pruning-keep-recent = "100"
pruning-interval = "10"

[api]
enable = false

[grpc]
enable = false

[state-sync]
snapshot-interval = 0
```

### 6.4 Sync from your own sentries

Use the same script as step 5.6, but with your sentries as the source:

```bash
SNAP_RPC="http://10.10.0.4:26657"
SNAP_RPC2="http://10.10.0.5:26657"
```

and in the `sed` line use `\"$SNAP_RPC,$SNAP_RPC2\"`. Your sentries make a snapshot every 1000 blocks (step 5.5), so wait until at least one exists:

```bash
# on a sentry
nd snapshots list
```

If your sentries have no snapshot yet, use the public RPC from step 5.6 for this first sync.

### 6.5 Start and check

```bash
docker compose up -d
docker compose logs -f --tail 50
curl -s http://10.10.0.3:26657/status | jq '.result.sync_info | {latest_block_height, catching_up}'
curl -s http://10.10.0.3:26657/net_info | jq -r '.result.peers[].remote_ip'    # must show only 10.10.0.4 and 10.10.0.5
```

Wait for `catching_up: false`, then turn state sync off (same command as in 5.8).

### 6.6 Save the validator public key

```bash
nd comet show-validator
```

It prints something like `{"@type":"/cosmos.crypto.ed25519.PubKey","key":"AbCd..."}`. Save it. You need it in Part 8.

---

## 7. Remote signer (TMKMS)

Do this on `signer`. TMKMS keeps the signing key and signs blocks for the validator over the private network. It also remembers the last block it signed, which protects you from signing the same block twice.

This guide uses **softsign** (key stored as a file on the signer server). For stronger safety use a hardware key (YubiHSM2): build with `--features=yubihsm` and pass the USB device into the container with `devices:`. The rest is the same.

### 7.1 Build the TMKMS image

```bash
mkdir -p /opt/cosmos/tmkms/data && cd /opt/cosmos/tmkms
nano Dockerfile
```

```dockerfile
FROM rust:1-bookworm AS build
RUN apt-get update && apt-get install -y --no-install-recommends libusb-1.0-0-dev pkg-config
RUN cargo install tmkms --features=softsign --locked

FROM debian:bookworm-slim
RUN apt-get update \
 && apt-get install -y --no-install-recommends ca-certificates libusb-1.0-0 \
 && rm -rf /var/lib/apt/lists/*
COPY --from=build /usr/local/cargo/bin/tmkms /usr/local/bin/tmkms
RUN useradd -u 1000 -m -d /tmkms tmkms
USER tmkms
WORKDIR /tmkms
ENTRYPOINT ["tmkms"]
```

```bash
nano docker-compose.yml
```

```yaml
services:
  tmkms:
    build: .
    image: local/tmkms:latest
    container_name: tmkms
    network_mode: host
    restart: unless-stopped
    volumes:
      - ./data:/tmkms/data
    command: ["start", "-c", "/tmkms/data/tmkms.toml"]
```

```bash
docker compose build                   # takes 10-20 minutes the first time
docker compose run --rm tmkms version
```

### 7.2 Create the TMKMS files

```bash
docker compose run --rm tmkms init /tmkms/data
ls data            # tmkms.toml  secrets/  state/  schema/
```

### 7.3 Move the validator key to the signer

The key file is `priv_validator_key.json` on `val`. Copy it over the private network.

On **val**:

```bash
scp /opt/cosmos/node/data/config/priv_validator_key.json cosmos@10.10.0.2:/opt/cosmos/tmkms/data/secrets/
```

On **signer**, turn it into the TMKMS format:

```bash
cd /opt/cosmos/tmkms
docker compose run --rm tmkms softsign import \
  /tmkms/data/secrets/priv_validator_key.json \
  /tmkms/data/secrets/cosmoshub-4-consensus.key
chmod 600 data/secrets/*
```

Now make a safe copy of `priv_validator_key.json` **offline** (encrypted USB drive, two copies, two places). Then remove it from the signer:

```bash
shred -u data/secrets/priv_validator_key.json
```

### 7.4 Edit `tmkms.toml`

```bash
nano data/tmkms.toml
```

Replace the whole file with:

```toml
[[chain]]
id = "cosmoshub-4"
key_format = { type = "bech32", account_key_prefix = "cosmospub", consensus_key_prefix = "cosmosvalconspub" }
state_file = "/tmkms/data/state/cosmoshub-4-consensus.json"

[[providers.softsign]]
chain_ids = ["cosmoshub-4"]
key_type = "consensus"
path = "/tmkms/data/secrets/cosmoshub-4-consensus.key"

[[validator]]
chain_id = "cosmoshub-4"
addr = "tcp://10.10.0.3:26659"
secret_key = "/tmkms/data/secrets/kms-identity.key"
protocol_version = "v0.34"
reconnect = true
```

`protocol_version = "v0.34"` is the value used for current CometBFT chains too. If TMKMS logs show a protocol error, check the TMKMS README for the value that fits your chain.

### 7.5 Tell the validator to use the signer

On **val**:

```bash
cd /opt/cosmos/node
docker compose down
nano data/config/config.toml
```

```toml
priv_validator_laddr = "tcp://10.10.0.3:26659"
```

Now remove the key from the validator, so it exists **only** on the signer (and in your offline backup):

```bash
shred -u data/config/priv_validator_key.json
```

Keep `data/data/priv_validator_state.json`. Do not delete it.

### 7.6 Start both

On **val**:

```bash
docker compose up -d
docker compose logs -f --tail 20       # it waits for the signer to connect
```

On **signer**:

```bash
cd /opt/cosmos/tmkms
docker compose up -d
docker compose logs -f --tail 20
```

Good signs:

- TMKMS log: `[cosmoshub-4@tcp://10.10.0.3:26659] connected to validator successfully`
- Validator log: blocks keep coming.

Check the key is the same as before:

```bash
# on val
nd comet show-validator        # must be the same key you saved in step 6.6
```

After Part 8, the TMKMS log will also show `signed PreVote` and `signed PreCommit` lines for every block.

---

## 8. Create the validator on chain

You need a **wallet** with tokens. This is a different key from the signing key in Part 7.

- Signing key (Part 7): signs blocks. Lives on `signer`.
- Wallet key (this part): holds your tokens, pays fees, sends the "create validator" message. **Do not keep it on any of the six servers.** Use a Ledger device or your own secured PC.

### 8.1 Make or import the wallet (on your own PC)

Install Docker on your PC and build the same image as in Part 4, or install `gaiad` directly. Then:

```bash
gaiad keys add wallet                    # new wallet: WRITE DOWN the 24 words on paper
# or
gaiad keys add wallet --recover          # bring back a wallet from 24 words
# or
gaiad keys add wallet --ledger           # use a Ledger device

gaiad keys show wallet -a                # your address: cosmos1...
```

Send tokens to this address. Then check:

```bash
gaiad query bank balances $(gaiad keys show wallet -a) --node https://rpc.YOURDOMAIN.com:443
```

(Until your own RPC from Part 9 is ready, use any public RPC for `--node`.)

### 8.2 Make sure everything is ready

- `val` shows `catching_up: false`.
- TMKMS is connected.
- You have the public key from step 6.6.

### 8.3 Write the validator file

```bash
nano validator.json
```

```json
{
  "pubkey": {"@type":"/cosmos.crypto.ed25519.PubKey","key":"<KEY_FROM_STEP_6.6>"},
  "amount": "1000000uatom",
  "moniker": "My Validator",
  "identity": "",
  "website": "https://example.com",
  "security": "security@example.com",
  "details": "Short text about your validator",
  "commission-rate": "0.10",
  "commission-max-rate": "0.20",
  "commission-max-change-rate": "0.01",
  "min-self-delegation": "1"
}
```

- `amount` is your own stake. `1000000uatom` = 1 ATOM.
- `commission-max-rate` and `commission-max-change-rate` **can never be changed later**. Choose with care.

### 8.4 Send it

```bash
gaiad tx staking create-validator validator.json \
  --from wallet \
  --chain-id cosmoshub-4 \
  --gas auto --gas-adjustment 1.4 \
  --gas-prices 0.005uatom \
  --node https://rpc.YOURDOMAIN.com:443
```

### 8.5 Check

```bash
gaiad query staking validator $(gaiad keys show wallet --bech val -a) --node https://rpc.YOURDOMAIN.com:443
```

- `status: BOND_STATUS_BONDED` means you are in the active set and signing.
- `BOND_STATUS_UNBONDED` means you are registered but your stake is too small to be in the active set. On Cosmos Hub mainnet the active set is full, so you need more stake than the lowest active validator. On a testnet this is easy.

On `signer`, the log should now show signed votes:

```bash
docker compose logs --tail 20 | grep signed
```

---

## 9. Public RPC / API node

Do this on `rpc`. First do Part 4 there.

### 9.1 Node setup

Follow steps 5.1–5.3 (init, genesis, node ID). Then edit the settings.

`data/config/config.toml`:

```toml
# --- [rpc] ---
laddr = "tcp://127.0.0.1:26657"            # only Caddy can reach it
cors_allowed_origins = ["*"]
max_open_connections = 900

# --- [p2p] ---
laddr = "tcp://0.0.0.0:26656"
external_address = "<RPC_PUBLIC_IP>:26656"
seeds = "<SEEDS_FROM_CHAIN_REGISTRY>"
persistent_peers = "<S1_ID>@10.10.0.4:26656,<S2_ID>@10.10.0.5:26656"
pex = true
addr_book_strict = false

# --- [tx_index] ---
indexer = "kv"                             # needed so users can search transactions

# --- [instrumentation] ---
prometheus = true
prometheus_listen_addr = "10.10.0.6:26660"
```

The RPC node does **not** know the validator. Do not put `<VAL_ID>` here.

`data/config/app.toml`:

```toml
minimum-gas-prices = "0.005uatom"

pruning = "custom"
pruning-keep-recent = "100000"      # keep more history for users (about one week)
pruning-interval = "100"

[api]
enable = true
address = "tcp://127.0.0.1:1317"
enabled-unsafe-cors = true

[grpc]
enable = true
address = "127.0.0.1:9090"
```

Sync with state sync from your sentries (step 6.4), then start:

```bash
docker compose up -d
curl -s http://127.0.0.1:26657/status | jq .result.sync_info.catching_up
```

### 9.2 HTTPS with Caddy

Caddy gets free HTTPS certificates by itself. First point three DNS names at `<RPC_PUBLIC_IP>`:

- `rpc.YOURDOMAIN.com`
- `api.YOURDOMAIN.com`
- `grpc.YOURDOMAIN.com`

```bash
mkdir -p /opt/cosmos/caddy && cd /opt/cosmos/caddy
nano Caddyfile
```

```
rpc.YOURDOMAIN.com {
    reverse_proxy 127.0.0.1:26657
}

api.YOURDOMAIN.com {
    reverse_proxy 127.0.0.1:1317
}

grpc.YOURDOMAIN.com {
    reverse_proxy h2c://127.0.0.1:9090
}
```

```bash
nano docker-compose.yml
```

```yaml
services:
  caddy:
    image: caddy:2
    container_name: caddy
    network_mode: host
    restart: unless-stopped
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./data:/data
      - ./config:/config
```

```bash
docker compose up -d
docker compose logs --tail 20
```

### 9.3 Test from your PC

```bash
curl -s https://rpc.YOURDOMAIN.com/status | jq .result.node_info.network
curl -s https://api.YOURDOMAIN.com/cosmos/base/tendermint/v1beta1/node_info | jq .default_node_info.network
```

Both should print `cosmoshub-4`.

---

## 10. Monitoring server

### 10.1 Server metrics on every server (node_exporter)

Run this on **all six** servers. Change the IP to that server's private IP.

```bash
mkdir -p /opt/cosmos/exporter && cd /opt/cosmos/exporter
nano docker-compose.yml
```

```yaml
services:
  node_exporter:
    image: prom/node-exporter:latest
    container_name: node_exporter
    network_mode: host
    pid: host
    restart: unless-stopped
    volumes:
      - /:/host:ro,rslave
    command:
      - --path.rootfs=/host
      - --web.listen-address=10.10.0.3:9100      # <- this server's private IP
```

```bash
docker compose up -d
```

### 10.2 Monitoring stack on `mon`

```bash
mkdir -p /opt/cosmos/monitoring/{prometheus,alertmanager,tenderduty} && cd /opt/cosmos/monitoring
```

**Prometheus settings**

```bash
nano prometheus/prometheus.yml
```

```yaml
global:
  scrape_interval: 15s

rule_files:
  - /etc/prometheus/alerts.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets: ["127.0.0.1:9093"]

scrape_configs:
  - job_name: servers
    static_configs:
      - targets: ["10.10.0.1:9100"]
        labels: { host: mon }
      - targets: ["10.10.0.2:9100"]
        labels: { host: signer }
      - targets: ["10.10.0.3:9100"]
        labels: { host: val }
      - targets: ["10.10.0.4:9100"]
        labels: { host: sentry1 }
      - targets: ["10.10.0.5:9100"]
        labels: { host: sentry2 }
      - targets: ["10.10.0.6:9100"]
        labels: { host: rpc }

  - job_name: cosmos
    static_configs:
      - targets: ["10.10.0.3:26660"]
        labels: { host: val }
      - targets: ["10.10.0.4:26660"]
        labels: { host: sentry1 }
      - targets: ["10.10.0.5:26660"]
        labels: { host: sentry2 }
      - targets: ["10.10.0.6:26660"]
        labels: { host: rpc }

  - job_name: tenderduty
    static_configs:
      - targets: ["127.0.0.1:28686"]
```

**Alert rules**

```bash
nano prometheus/alerts.yml
```

```yaml
groups:
  - name: basics
    rules:
      - alert: ServerDown
        expr: up == 0
        for: 2m
        labels: { severity: critical }
        annotations: { summary: "{{ $labels.host }} ({{ $labels.job }}) is down" }

      - alert: DiskAlmostFull
        expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) < 0.15
        for: 10m
        labels: { severity: warning }
        annotations: { summary: "{{ $labels.host }} has less than 15% disk left" }

      - alert: ChainNotMoving
        expr: increase(cometbft_consensus_height[5m]) == 0
        for: 5m
        labels: { severity: critical }
        annotations: { summary: "{{ $labels.host }} block height is not going up" }

      - alert: FewPeers
        expr: cometbft_p2p_peers < 2
        for: 10m
        labels: { severity: warning }
        annotations: { summary: "{{ $labels.host }} has fewer than 2 peers" }
```

Metric names start with `cometbft_` on new chains and `tendermint_` on old ones. Check with `curl -s http://10.10.0.4:26660/metrics | grep consensus_height` and fix the rules to match.

**Alertmanager (sends alerts to Telegram)**

```bash
nano alertmanager/alertmanager.yml
```

```yaml
route:
  receiver: telegram
  group_wait: 30s
  repeat_interval: 2h

receivers:
  - name: telegram
    telegram_configs:
      - bot_token: "<TELEGRAM_BOT_TOKEN>"
        chat_id: <TELEGRAM_CHAT_ID>
```

**Tenderduty (watches your validator's signing)**

Make the example settings file, then edit it:

```bash
docker run --rm ghcr.io/blockpane/tenderduty:latest -example-config > tenderduty/config.yml
nano tenderduty/config.yml
```

Set these values (leave the rest as it is):

```yaml
enable_dashboard: yes
listen_port: 8888
prometheus_enabled: yes
prometheus_listen_port: 28686

chains:
  "Cosmos Hub":
    chain_id: cosmoshub-4
    valoper_address: cosmosvaloper1...        # your validator operator address
    public_fallback: no
    nodes:
      - url: tcp://10.10.0.4:26657
        alert_if_down: yes
      - url: tcp://10.10.0.5:26657
        alert_if_down: yes
```

Also fill in the Telegram or Discord block in that file to get "missed blocks" alerts.

**Compose file**

```bash
nano docker-compose.yml
```

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    network_mode: host
    restart: unless-stopped
    user: "1000"
    volumes:
      - ./prometheus:/etc/prometheus:ro
      - ./prom-data:/prometheus
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --storage.tsdb.retention.time=30d
      - --web.listen-address=127.0.0.1:9090

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    network_mode: host
    restart: unless-stopped
    volumes:
      - ./alertmanager:/etc/alertmanager:ro
    command:
      - --config.file=/etc/alertmanager/alertmanager.yml
      - --web.listen-address=127.0.0.1:9093

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    network_mode: host
    restart: unless-stopped
    user: "1000"
    environment:
      GF_SERVER_HTTP_ADDR: 127.0.0.1
      GF_SERVER_HTTP_PORT: "3000"
      GF_SECURITY_ADMIN_PASSWORD: "<CHANGE_THIS_PASSWORD>"
      GF_USERS_ALLOW_SIGN_UP: "false"
    volumes:
      - ./grafana-data:/var/lib/grafana

  tenderduty:
    image: ghcr.io/blockpane/tenderduty:latest
    container_name: tenderduty
    network_mode: host
    restart: unless-stopped
    volumes:
      - ./tenderduty/config.yml:/var/lib/tenderduty/config.yml

  caddy:
    image: caddy:2
    container_name: caddy
    network_mode: host
    restart: unless-stopped
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./caddy-data:/data
```

```bash
nano Caddyfile
```

```
grafana.YOURDOMAIN.com {
    reverse_proxy 127.0.0.1:3000
}
```

Start it:

```bash
mkdir -p prom-data grafana-data caddy-data
docker compose up -d
docker compose ps
```

### 10.3 Check

```bash
curl -s http://127.0.0.1:9090/api/v1/targets | jq -r '.data.activeTargets[] | "\(.labels.host // .labels.job)  \(.labels.job)  \(.health)"'
```

Every line should end with `up`.

### 10.4 Grafana

1. Open `https://grafana.YOURDOMAIN.com`, log in as `admin` with your password.
2. **Connections → Data sources → Add → Prometheus**, URL `http://127.0.0.1:9090`, Save.
3. **Dashboards → New → Import**. Import a "Node Exporter Full" dashboard for the servers and a "Cosmos Validator" / "CometBFT" dashboard for the chain. Search for them on grafana.com/dashboards and paste the ID.

Tenderduty has its own page on port `8888`. It is not open to the internet. View it through an SSH tunnel from your PC:

```bash
ssh -L 8888:127.0.0.1:8888 cosmos@<MON_PUBLIC_IP>
# then open http://localhost:8888
```

Optional later step: add Loki with Grafana Alloy on each server to collect container logs in Grafana.

---

## 11. Daily work

### 11.1 Useful commands

```bash
# any node server
cd /opt/cosmos/node
docker compose ps
docker compose logs -f --tail 100
docker compose restart
curl -s http://<PRIVATE_IP>:26657/status | jq '.result.sync_info | {latest_block_height, catching_up}'

# signer
cd /opt/cosmos/tmkms && docker compose logs -f --tail 50
```

### 11.2 Chain upgrade (new binary version)

Cosmos chains upgrade at a fixed block height that is voted on chain. With Docker the steps are:

1. **Before the height:** on each node server, build the new image so it is ready.
   ```bash
   cd /opt/cosmos/node
   cp .env .env.new
   nano .env.new               # set the new VERSION and BINARY_URL
   docker compose --env-file .env.new build
   ```
2. **At the height:** the node stops by itself and the log says `UPGRADE "vNN" NEEDED at height ...`. The container will keep restarting and stopping. That is normal.
3. **Switch:**
   ```bash
   mv .env.new .env
   docker compose up -d
   docker compose logs -f --tail 50
   ```
4. **Order:** sentries first, then `rpc`, then `val` last. Blocks start again when two thirds of the network has upgraded.

Never start the new version before the upgrade height. The node would fail with a wrong app hash.

### 11.3 Backups

| What | Where | How often |
|------|-------|-----------|
| `priv_validator_key.json` | Offline, encrypted, 2 copies | Once (Part 7) |
| Wallet 24 words | On paper, offline | Once (Part 8) |
| `node_key.json` of each node | `/opt/cosmos/node/data/config/` | Once |
| `.env`, `docker-compose.yml`, `config.toml`, `app.toml`, `tmkms.toml` | Private git repo (no keys inside!) | After each change |
| Grafana data | `/opt/cosmos/monitoring/grafana-data` | Weekly |

Chain data does not need a backup. You can always sync again.

### 11.4 Disk space

```bash
df -h /
du -sh /opt/cosmos/node/data/data/*
docker system df
docker image prune -af --filter "until=720h"      # remove old images
```

### 11.5 Unjail

If your validator misses too many blocks it is "jailed" (paused). Fix the cause first, wait until `val` is synced, then from your PC:

```bash
gaiad tx slashing unjail --from wallet --chain-id cosmoshub-4 \
  --gas auto --gas-adjustment 1.4 --gas-prices 0.005uatom --node https://rpc.YOURDOMAIN.com:443
```

---

## 12. Safety rules and common problems

### 12.1 Double signing — the one mistake you cannot undo

If the same signing key signs two different blocks at the same height, the chain **slashes your stake and removes the validator for good** ("tombstoned").

Rules:

1. The signing key runs in **one place only**. Never run two validators or two TMKMS copies with the same key at the same time.
2. Never delete TMKMS `state/` files or the validator's `priv_validator_state.json` on a validator that has signed before.
3. When you move the validator or signer to a new server: stop the old one, **make sure it is really stopped and cannot start again**, copy the state file, then start the new one.
4. Do not restore an old backup of a state file onto a live validator.

### 12.2 Other rules

- No wallet keys on any server.
- `val` and `signer` have no public ports except SSH and WireGuard. Better still: allow SSH only from your own IP.
- Change settings on one server at a time, and check after each change.
- Keep keys and passwords out of git.

### 12.3 Common problems

| Problem | Likely cause | Fix |
|---------|--------------|-----|
| Node has 0 peers | Bad seeds/peers, port 26656 closed | Check `ufw status`, take fresh peers from the chain registry |
| Validator has 0 peers | Wrong node IDs, WireGuard down | `wg show`, compare IDs with `nd comet show-node-id`, check `addr_book_strict = false` |
| State sync stuck on "Discovering snapshots" | No snapshot on the source | Wait and retry, try another RPC, or use a snapshot file |
| `wrong Block.Header.AppHash` | Wrong binary version | Check `VERSION` against the network (step 4.1) |
| Validator log: waiting for signer / `connection refused` on 26659 | TMKMS not running or firewall | Start TMKMS, check UFW rule for `10.10.0.2 → 26659` |
| TMKMS: `chain ID mismatch` or key errors | Wrong `id` / `chain_ids` in `tmkms.toml` | Make them equal to `CHAIN_ID` |
| `permission denied` in `data/` | User id in the image is not the owner | `sudo chown -R 1000:1000 /opt/cosmos/node/data` |
| Validator misses blocks | Slow disk, clock drift, sentries behind | Check Grafana, `chronyc tracking`, sentry sync |
| Container restarts in a loop | Upgrade height reached, or bad config | `docker compose logs --tail 50` and read the last error |

### 12.4 Final checklist

- [ ] All six servers ping each other on `10.10.0.x`
- [ ] UFW on and only the planned ports open on every server
- [ ] `sentry1`, `sentry2`, `rpc`, `val` show `catching_up: false`
- [ ] `val` has exactly two peers (the sentries)
- [ ] Sentries have `private_peer_ids = <VAL_ID>`
- [ ] `priv_validator_key.json` is **not** on `val`
- [ ] TMKMS log shows `signed PreVote` / `signed PreCommit`
- [ ] Offline backup of the signing key and the wallet words exists
- [ ] All Prometheus targets are `up`
- [ ] A test alert reaches Telegram
- [ ] Tenderduty shows the validator as signing

---

## Notes on this guide

- Commands are written for Ubuntu 22.04 / 24.04 and Docker Compose v2.
- Version numbers, download links, seeds and peers change over time. Always take them from the chain's official repo and the chain registry at the time you install.
- The public RPC endpoint (`cosmos-rpc.polkachu.com`) is a third-party service used as an example.
- For higher uptime later: replace the single TMKMS server with a Horcrux signer cluster (three or more signer servers), and add a third sentry in another data center.
