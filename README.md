# BitGold v4 Node Deployment (Mainnet / Testnet)

One `geth` binary in your home directory, available as `geth`, and everything node-related in `~/node`:
config, genesis, `password.txt`, keystore and the chain data.
## Requirements

### Hardware Requirements

CPU: 4 core
Memory: 8 GB
Disk: IMPORTANT 1TB, NVMe SSD, 40k IOPS, 500 MB/S throughput, read latency <1ms.
Network Bandwidth: >= 512 Mbps

### Software Requirements

Ubuntu 22.04 LTS

## Network parameters

| Item | Mainnet | Testnet |
| --- | --- | --- |
| chainId / networkId | **618** | **617** |
| Release asset | `geth-1.8.1-mainnet-linux-amd64.zip` | `geth-1.8.1-testnet-linux-amd64.zip` |
| Config file | `~/node/config.toml` | `~/node/config-testnet.toml` |
| Genesis file | `~/node/genesis.json` | `~/node/genesis-testnet.json` |
| Also in the zip | `geth` + start scripts | same |
| P2P | `30666` (TCP + UDP) | same |
| HTTP RPC | `8545` | same |
| WebSocket | `8546` | same |

⚠️ One network per host: `~/node` can hold the data of a single network. To run both on one host, give the
second one its own directory with `NODE_DIR` (see the note at the end of section 1).

## 1. Deploy

### Testnet

```bash
cd ~

# 1) unzip the release: geth, config-testnet.toml, genesis-testnet.json, start scripts
curl -LO https://github.com/bitgold-network/bitgold-geth-deploy/releases/download/1.8.1/geth-1.8.1-testnet-linux-amd64.zip
unzip -o geth-1.8.1-testnet-linux-amd64.zip
chmod +x geth *.sh

# 2) node directory: config, genesis, password and the chain data live here
mkdir -p ~/node
mv -f config-testnet.toml genesis-testnet.json ~/node/

# 3) put geth on PATH once (then `geth version` simply works)
echo 'export PATH="$HOME:$PATH"' >> ~/.bashrc
source ~/.bashrc
geth version

# 4) initialise the datadir, once
geth --datadir ~/node init ~/node/genesis-testnet.json
```

### Mainnet

Same steps, with `config.toml` and `genesis.json`:

```bash
cd ~

curl -LO https://github.com/bitgold-network/bitgold-geth-deploy/releases/download/1.8.1/geth-1.8.1-mainnet-linux-amd64.zip
unzip -o geth-1.8.1-mainnet-linux-amd64.zip
chmod +x geth *.sh

mkdir -p ~/node
mv -f config.toml genesis.json ~/node/

# only if ~ is not on PATH yet:
# echo 'export PATH="$HOME:$PATH"' >> ~/.bashrc && source ~/.bashrc

geth --datadir ~/node init ~/node/genesis.json
```

Sanity checks:

```bash
geth version

grep -m1 '"chainId"' ~/node/genesis-testnet.json    # 617
grep -m1 '"chainId"' ~/node/genesis.json            # 618
```

> - `unzip` is not installed everywhere: `sudo apt install -y unzip` once.
> - If the zip unpacks into a nested directory, move its files up into `~` first.
> - `source ~/.bashrc` only affects the current shell; new shells and `ssh` sessions pick the change up automatically.
> - Running both networks on one host: keep the second one out of `~/node`, e.g.
>   `NODE_DIR=$HOME/node-mainnet ./start-validator.sh 0x<address>`. All three scripts honour `NODE_DIR`.

### Directory layout

```
~/
├── geth                     # binary, on PATH
├── start-validator.sh
├── start-rpc.sh
├── validator-info.sh
└── node/
    ├── config-testnet.toml  # mainnet: config.toml
    ├── genesis-testnet.json # mainnet: genesis.json
    ├── password.txt         # keystore + BLS password
    ├── keystore/            # validator account
    ├── bls/                 # BLS key and proof
    └── geth/chaindata/      # chain data
```

## 2. Edit the config file (required before starting)

The config is `~/node/config-testnet.toml` on testnet and `~/node/config.toml` on mainnet. Only two things
need changing.

### 2.1 StaticNodes (required)

The config sets `NoDiscovery = true`, so the node does **not** look for peers on its own. With an empty
`StaticNodes` it will stay at zero peers and never sync.

```toml
[Node.P2P]
NoDiscovery = true
StaticNodes = [
  "enode://<pubkey>@<ip-or-host>:30666",
  "enode://<pubkey>@<ip-or-host>:30666",
]
```

- Ask the network operator for seed enodes; list at least two (ideally from different operators).
- Your own enode, once the node is running:
  `geth attach --exec 'admin.nodeInfo.enode' ~/node/geth.ipc`.
- Changes only take effect after a restart.

### 2.2 RPC listener (per role)

| Role | `HTTPHost` | `HTTPModules` |
| --- | --- | --- |
| Validator | `127.0.0.1` | leave as shipped |
| RPC behind a reverse proxy | `127.0.0.1` | drop `admin` |
| RPC directly exposed | `0.0.0.0` | `["eth","net","web3","txpool","parlia"]` |

HTTP and WebSocket are enabled by `HTTPHost` / `WSPort` in the config; the scripts do not pass any
extra `--http` flags.

## 3. Validator node

### 3.1 Password file

Create it first. The same file is the password of the validator keystore **and** of the BLS key, and it is
what `--password` / `--blspassword` read at startup, so no password is ever typed on the command line.

```bash
printf '%s' 'your-password' > ~/node/password.txt
chmod 600 ~/node/password.txt
```

Use `printf` rather than `echo`: a trailing newline makes the unlock fail.

### 3.2 Create the validator account

`--password` reads the password from the file, so the command runs non-interactively:

```bash
geth --datadir ~/node account new --password ~/node/password.txt
```

Note the `Public address of the key: 0x...` line. This is the validator address; it signs blocks.
The keystore lives in `~/node/keystore/`; **back it up offline immediately**.

### 3.3 BLS account

Fast finality in Parlia uses a BLS key. `--blspassword` reads the same file, so this is non-interactive too:

```bash
geth bls account new --datadir ~/node --blspassword ~/node/password.txt --show-private-key
```

`--show-private-key` prints the BLS private key: store it offline together with the keystore, and clear it
from your terminal scrollback and shell history afterwards.

No separate BLS password file is needed: `--password` and `--blspassword` both point at
`~/node/password.txt`.

### 3.4 Inspect the keys / generate the proof

```bash
./validator-info.sh --gen-proof
```

It prints the validator account, the BLS account (including its public key) and the BLS proof, then the
values to hand over. Once the proof exists, plain `./validator-info.sh` is enough. The node does not need
to be running.

The underlying call takes **two positional arguments**: the operator (validator) account and the BLS
public key. The script fills both in from the keystore and from `geth bls account list`:

```bash
geth bls account generate-proof --datadir ~/node --blspassword ~/node/password.txt \
  0x<validator-address> 0x<bls-public-key>
```

Without them geth stops with `Fatal: Operator account must be given as argument.` or
`Fatal: BLS pubkey must be given as argument.` The BLS public key is the `[BLS public key] 0x...` line in
the `bls account list` output; override it with `--bls-pubkey` if needed.

The proof file (`proofOfPossession.json`) is written to the current directory on some builds and to the
datadir on others; the script looks in both, so run it from `~`. If the keystore holds several accounts,
pick one with `--validator 0x<address>`. `--gen-proof` exits non-zero when no proof could be produced.

### 3.5 Start

```bash
./start-validator.sh 0x<validator-address>
```

Without an argument it picks the first account in the keystore: `./start-validator.sh`.

Equivalent command (testnet; mainnet uses `~/node/config.toml`):

```bash
geth --config ~/node/config-testnet.toml --datadir ~/node --networkid 617 \
  --password ~/node/password.txt \
  --miner.etherbase=0x<validator-address> \
  --unlock=0x<validator-address> \
  --blspassword ~/node/password.txt \
  --mine --vote
```

Run it in the background with:

```bash
nohup ./start-validator.sh 0x<validator-address> > validator.log 2>&1 &
```

For long-running nodes prefer systemd (section 6).

### 3.6 Join the validator set

A locally running node is not yet producing blocks: the validator set is maintained on chain, so the
following has to be submitted to the operator:

| Item | Where to get it |
| --- | --- |
| Validator address | `validator-info.sh` |
| BLS public key / proof | `validator-info.sh` |
| enode | `geth attach --exec 'admin.nodeInfo.enode' ~/node/geth.ipc` |
| Public IP + P2P port | your host (port 30666 must be reachable from the internet) |

It takes effect at the next epoch boundary (Parlia uses 200-block epochs by default), not immediately.
The staking/governance entry point is defined by the operator; this repository contains no contract calls.

### 3.7 Confirm block production

```bash
# is the latest block signed by you?
geth attach --exec 'eth.getBlock("latest").miner' ~/node/geth.ipc

# sealing log (when running in the background with output redirected to validator.log)
grep 'Successfully sealed new block' validator.log
```

## 4. RPC node

### 4.1 Start

```bash
./start-rpc.sh
```

Equivalent to `geth --config ~/node/config-testnet.toml --datadir ~/node --networkid 617` (mainnet:
`~/node/config.toml`); the listen address and the exposed APIs come from the config (see 2.2).

### 4.2 Verify

```bash
curl -s -X POST -H 'Content-Type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"eth_chainId","params":[]}' \
  http://127.0.0.1:8545
# testnet returns 0x269 (617), mainnet 0x26a (618)

curl -s -X POST -H 'Content-Type: application/json' \
  --data '{"jsonrpc":"2.0","id":1,"method":"eth_blockNumber","params":[]}' \
  http://127.0.0.1:8545
```

### 4.3 Exposing it

- Only open 30666 in the firewall; 8545 / 8546 should be reachable from the reverse proxy or the LAN only.
- Public RPC must sit behind Nginx with rate limiting, at minimum:

```nginx
limit_req_zone $binary_remote_addr zone=rpc:10m rate=20r/s;

location / {
    limit_req zone=rpc burst=40 nodelay;
    proxy_pass http://127.0.0.1:8545;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
}
```

- Never enable `admin`, `personal` or `debug` in the API whitelist.
- Transaction indexes and full state dominate disk usage; plan for capacity, or run a separate archive node.

## 5. Common commands

```bash
geth version                              # binary in use
geth attach ~/node/geth.ipc               # interactive console

eth.blockNumber                           # local height
eth.syncing                               # false means fully synced
net.peerCount                             # peers; stuck at 0 means StaticNodes is wrong
admin.nodeInfo.enode                      # your enode
eth.getBlock("latest").miner              # signer of the latest block
```

One-shot form: `geth attach --exec 'net.peerCount' ~/node/geth.ipc`

## 6. Run in background (systemd, optional)

`/etc/systemd/system/bitgold-validator.service`:

```ini
[Unit]
Description=BitGold validator
After=network-online.target

[Service]
User=bitgold
WorkingDirectory=/home/bitgold
ExecStart=/home/bitgold/start-validator.sh 0x<validator-address>
Restart=always
RestartSec=10
LimitNOFILE=65535
TimeoutStopSec=300

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now bitgold-validator
journalctl -u bitgold-validator -f
```

For an RPC node, point `ExecStart` at `start-rpc.sh` and rename the unit.

The scripts look for `geth` on `PATH` and fall back to `$HOME/geth`, so the service does not need the
`.bashrc` PATH entry. Do not lower `TimeoutStopSec=300`: geth needs time to flush the database on
SIGTERM, and a hard kill makes the next start much slower.

## 7. Troubleshooting

| Symptom | Cause / fix |
| --- | --- |
| `geth: command not found` | `echo 'export PATH="$HOME:$PATH"' >> ~/.bashrc && source ~/.bashrc`, or call `~/geth` |
| `no config-testnet.toml or config.toml in ...` | config is not in `~/node`; move it there or set `NODE_DIR` |
| Exits immediately with `incompatible genesis` | `~/node` was initialised with a different genesis; start over in a fresh directory |
| `peers = 0`, repeated `Looking for peers` | empty `StaticNodes` / wrong enode / port 30666 closed / networkId mismatch |
| `bind: address already in use` | port taken; change `ListenAddr` when running several nodes on one host |
| `Failed to unlock account` | wrong `~/node/password.txt` content (extra space or newline) |
| `Fatal: Operator account must be given as argument` / `Fatal: BLS pubkey must be given as argument` | `generate-proof` needs both positional arguments; use `./validator-info.sh --gen-proof`, which appends them |
| Synced but no blocks | the address is not in the validator set, or `--miner.etherbase` differs from `--unlock` |
| Blocks produced but finality stalls | the BLS key is locked: `--blspassword` must point at `password.txt` and that password must match the BLS key |
| RPC unreachable from outside | `HTTPHost` still `localhost`, firewall, or missing reverse proxy |
| Disk fills up quickly | full transaction index plus full state; add capacity or split off an archive node |

## 8. Security

- Private keys stay in `~/node/keystore`; never put them on a command line, in a script, or in chat.
- `~/node/password.txt` must be mode 600; keep an encrypted offline copy of the keystore and the password.
- Run validators and RPC nodes on separate hosts; validators should not serve public RPC.
- Run geth as an unprivileged user, never as root.
- Use different keys for mainnet and testnet.
- Upgrade promptly and read the release notes for hard-fork heights.

## Appendix: scripts

All three scripts ship inside the release zip and use `$HOME/node` by default; set `NODE_DIR` to point them
elsewhere. Each resolves `geth` from `PATH`, then `$HOME/geth`.

| Script | Purpose |
| --- | --- |
| `start-validator.sh [address]` | start a validator; uses `config-testnet.toml` or `config.toml` (whichever is in the node directory), picks the first keystore account when no address is given, and passes `password.txt` to both `--password` and `--blspassword` |
| `start-rpc.sh` | start an RPC node |
| `validator-info.sh [--gen-proof]` | print validator account, BLS account and BLS proof; `--gen-proof` generates the proof when missing |
