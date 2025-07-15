# TAC Chain Mainnet - Complete Setup Guide

## Table of Contents
- [System Requirements](#system-requirements)
- [System Update](#system-update)
- [Go Installation](#go-installation)
- [TAC Chain Binary Installation](#tac-chain-binary-installation)
- [Cosmovisor Installation](#cosmovisor-installation)
- [Service Creation](#service-creation)
- [Node Configuration](#node-configuration)
- [Genesis and Addrbook Download](#genesis-and-addrbook-download)
- [Peers and Pruning Settings](#peers-and-pruning-settings)
- [Port Configuration](#port-configuration)
- [Snapshot Download](#snapshot-download)
- [Node Start](#node-start)
- [Sync Check](#sync-check)
- [Wallet Creation](#wallet-creation)
- [Balance Check](#balance-check)
- [Validator Setup](#validator-setup)
- [Validator Check](#validator-check)
- [Useful Commands](#useful-commands)
- [Update Process](#update-process)
- [Important Links](#important-links)
- [Important Notes](#important-notes)

## System Requirements

| Component | Minimum Requirements |
|-----------|---------------------|
| CPU       | 8+ Core             |
| RAM       | 16+ GB              |
| Storage   | 500GB+ SSD          |
| Network   | 100+ Mbps           |

## System Update

```bash
sudo apt update
sudo apt install curl git jq lz4 build-essential make gcc snapd chrony tmux unzip bc -y
sudo apt upgrade -y
```

## Go Installation

> **Note:** If you already have Go 1.19+ installed, skip this step

```bash
# Check current Go version
go version

# If Go is missing or version is below 1.19, run the following commands:
rm -rf $HOME/go
sudo rm -rf /usr/local/go
cd $HOME
curl https://dl.google.com/go/go1.24.5.linux-amd64.tar.gz | sudo tar -C/usr/local -zxvf -
cat <<'EOF' >>$HOME/.profile
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export GO111MODULE=on
export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin
EOF
source $HOME/.profile
go version
```

## TAC Chain Binary Installation

```bash
cd $HOME
rm -rf tacchain
git clone https://github.com/TacBuild/tacchain.git
cd tacchain
git checkout v1.0.1
make build
```

## Cosmovisor Installation

```bash
# Install Cosmovisor
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.5.0

# Create genesis binary directory
mkdir -p $HOME/.tacchaind/cosmovisor/genesis/bin
mkdir -p /root/.tacchaind/cosmovisor/upgrades/v1.0.1/bin
cp build/tacchaind /root/.tacchaind/cosmovisor/upgrades/v1.0.1/bin/
cp build/tacchaind $HOME/.tacchaind/cosmovisor/genesis/bin/

# Create system symlinks
sudo ln -s $HOME/.tacchaind/cosmovisor/genesis $HOME/.tacchaind/cosmovisor/current -f
sudo ln -s $HOME/.tacchaind/cosmovisor/current/bin/tacchaind /usr/local/bin/tacchaind -f
```

## Service Creation

```bash
sudo tee /etc/systemd/system/tacchaind.service > /dev/null << EOF
[Unit]
Description=tacchaind node service
After=network-online.target

[Service]
User=$USER
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.tacchaind"
Environment="DAEMON_NAME=tacchaind"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:$HOME/.tacchaind/cosmovisor/current/bin"

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable tacchaind.service
```

## Node Configuration

```bash
# Initialize node (replace NODE_NAME with your desired name)
tacchaind init NODE_NAME --chain-id tacchain_239-1

# Set port configuration (optional - default is 26xxx, here we use 59xxx)
echo 'export TAC_PORT="59"' >> $HOME/.bash_profile
source $HOME/.bash_profile
```

## Genesis and Addrbook Download

```bash
# Download genesis file
curl -Ls https://ss.tac.nodestake.org/genesis.json > $HOME/.tacchaind/config/genesis.json

# Download addrbook file
curl -Ls https://ss.tac.nodestake.org/addrbook.json > $HOME/.tacchaind/config/addrbook.json
```

## Peers and Pruning Settings

```bash
# Peers configuration (seeds left empty)
SEEDS=""
PEERS="d146b7727aa3a91404c23faa149c35896ebd82f1@141.95.97.21:60256,a460c869b5132849dc761d862a728350a8712fba@63.251.232.230:45110,6a16de4127bb0881e7fb7bb72e5083dbd6015f07@109.123.108.141:45110,10550a03e4f7fa487c78fbd07e0770e2b0f085c7@64.46.115.78:58960,b047e2dd6b068d8190b574264c93bc3e270dcf76@107.6.89.134:55130,509286977a3c644b6fc36b1a5e0f3927bb3f1266@84.32.186.70:60256,0327e180e47c30c47b6a69bdd862fd249c701212@23.106.238.97:60256,338eca43dc3d17a5ec7bf0d4ad38883452849569@63.251.232.121:55130,0efae9d157f0ef60ad7d25507d6939799f832e34@69.4.239.26:58960"

sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.tacchaind/config/config.toml

# Pruning settings
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-keep-every *=.*|pruning-keep-every = "0"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "19"|' \
  $HOME/.tacchaind/config/app.toml

# Timeout settings
sed -i 's/timeout_commit = "5s"/timeout_commit = "2s"/' $HOME/.tacchaind/config/config.toml
```

## Port Configuration

> **Optional:** If you want to change ports from default 26xxx to 59xxx

```bash
# Configure ports (if you want to change from default 26xxx to 59xxx)
sed -i.bak -e "s%:1317%:${TAC_PORT}317%g;
s%:8080%:${TAC_PORT}080%g;
s%:9090%:${TAC_PORT}090%g;
s%:9091%:${TAC_PORT}091%g;
s%:8545%:${TAC_PORT}545%g;
s%:8546%:${TAC_PORT}546%g;
s%:6065%:${TAC_PORT}065%g" $HOME/.tacchaind/config/app.toml

sed -i.bak -e "s%:26658%:${TAC_PORT}658%g;
s%:26657%:${TAC_PORT}657%g;
s%:6060%:${TAC_PORT}060%g;
s%:26656%:${TAC_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${TAC_PORT}656\"%;
s%:26660%:${TAC_PORT}660%g" $HOME/.tacchaind/config/config.toml
```

## Snapshot Download

> **Recommended:** This significantly speeds up synchronization

```bash
# Download snapshot for fast synchronization
SNAP_NAME=$(curl -s https://ss.tac.nodestake.org/ | egrep -o ">20.*\.tar.lz4" | tr -d ">")
curl -o - -L https://ss.tac.nodestake.org/${SNAP_NAME} | lz4 -c -d - | tar -x -C $HOME/.tacchaind
```

## Node Start

```bash
# Start the node
sudo systemctl start tacchaind.service

# Follow logs
sudo journalctl -u tacchaind.service -f --no-hostname -o cat
```

## Sync Check

```bash
# Check status (add --node if using custom port)
tacchaind status --node http://localhost:59657

# Check sync status
tacchaind status --node http://localhost:59657 2>&1 | jq .sync_info.catching_up

# If returns "false", node is fully synced
```

## Wallet Creation

```bash
# Create new wallet
tacchaind keys add wallet_name

# List wallets
tacchaind keys list

# Show wallet address
tacchaind keys show wallet_name -a

# Show EVM address
echo "0x$(tacchaind debug addr $(tacchaind keys show wallet_name -a) | grep hex | awk '{print $3}')"
```

## Balance Check

```bash
# Check balance
tacchaind query bank balances $(tacchaind keys show wallet_name -a) --node http://localhost:59657
```

## Validator Setup

> **Important:** Send TAC tokens to your wallet first! Minimum: 1 TAC + gas fees (~0.15 TAC)

```bash
# Get validator pubkey
tacchaind tendermint show-validator

# Create validator transaction file
cat > validatortx.json << 'EOF'
{
    "pubkey": {
        "@type": "/cosmos.crypto.ed25519.PubKey",
        "key": "VALIDATOR_PUBKEY_HERE"
    },
    "amount": "1000000000000000000utac",
    "moniker": "YOUR_MONIKER",
    "identity": "YOUR_IDENTITY",
    "website": "YOUR_WEBSITE",
    "security": "YOUR_EMAIL",
    "details": "YOUR_DESCRIPTION",
    "commission-rate": "0.1",
    "commission-max-rate": "0.2",
    "commission-max-change-rate": "0.01",
    "min-self-delegation": "1"
}
EOF

# Add pubkey to file
PUBKEY=$(tacchaind tendermint show-validator | jq -r .key)
sed -i "s/VALIDATOR_PUBKEY_HERE/$PUBKEY/g" validatortx.json

# Create validator
tacchaind tx staking create-validator validatortx.json \
    --from wallet_name \
    --chain-id tacchain_239-1 \
    --node http://localhost:59657 \
    --gas auto --gas-adjustment 1.4 \
    --gas-prices 400000000000000utac -y
```

## Validator Check

```bash
# Search for validator in list
tacchaind query staking validators --node http://localhost:59657 | grep "YOUR_MONIKER"

# Check validator status
tacchaind query staking validator $(tacchaind tendermint show-validator) --node http://localhost:59657
```

## Useful Commands

```bash
# Node status
sudo systemctl status tacchaind

# Restart node
sudo systemctl restart tacchaind

# View logs
sudo journalctl -u tacchaind -f --no-pager

# Stop node
sudo systemctl stop tacchaind

# Check disk usage
du -sh $HOME/.tacchaind/
```

## Update Process

> **Future use:** When new versions are released

```bash
# Update to new version
cd $HOME/tacchain
git fetch --all
git checkout NEW_VERSION
make build

# Create new version directory
mkdir -p $HOME/.tacchaind/cosmovisor/upgrades/NEW_VERSION/bin
cp build/tacchaind $HOME/.tacchaind/cosmovisor/upgrades/NEW_VERSION/bin/

# Create upgrade info file
cat <<EOF > $HOME/.tacchaind/cosmovisor/upgrades/NEW_VERSION/upgrade-info.json
{
  "name": "NEW_VERSION",
  "height": UPGRADE_HEIGHT,
  "info": "upgrade description"
}
EOF
```

## Important Links

- **Website:** https://tac.build
- **Explorer:** https://explorer.tac.build
- **RPC:** https://rpc.tac.build
- **Documentation:** https://docs.tac.build

## Important Notes

- **Mainnet Chain ID:** `tacchain_239-1`
- **Current Version:** `v1.0.1` (July 15, 2025 mainnet launch)
- **Save your mnemonic phrase securely!**
- **Take backups regularly!**
- **Minimum 1 TAC + gas fees required for validator**
- **Wait for full synchronization before creating validator**
- **Use snapshot for fast synchronization**

---

**This guide is based on TAC mainnet launch experience. Following step by step will ensure successful setup!**

## Contributing

If you encounter issues or have improvements, please:

1. Check the [troubleshooting section](#useful-commands)
2. Review the [official documentation](https://docs.tac.build)
3. Join the community channels for support

## License

This guide is provided as-is for educational purposes. Please refer to the official TAC documentation for the most up-to-date information.
