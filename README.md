#!/bin/bash

read -r -p "Enter node moniker: " NODE_MONIKER

CHAIN_ID="humans_3000-1"
CHAIN_DENOM="uheart"
BINARY_NAME="humansd"
BINARY_VERSION_TAG="v0.1.0"

echo -e "Node moniker:       $NODE_MONIKER"
echo -e "Chain id:           $CHAIN_ID"
echo -e "Chain demon:        $CHAIN_DENOM"
echo -e "Binary version tag: $BINARY_VERSION_TAG"
sleep 2

sudo apt update
sudo apt install -y make gcc jq curl git lz4 build-essential chrony unzip
curl -L -# -O "https://golang.org/dl/go$version.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$version.linux-amd64.tar.gz"
rm "go$version.linux-amd64.tar.gz"
touch $HOME/.bash_profile
source $HOME/.bash_profile
PATH_INCLUDES_GO=$(grep "$HOME/go/bin" $HOME/.bash_profile)
if [ -z "$PATH_INCLUDES_GO" ]; then
  echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> $HOME/.bash_profile
  echo "export GOPATH=$HOME/go" >> $HOME/.bash_profile
fi
source .bash_profile
go version
sleep 2

cd $HOME
git clone https://github.com/humansdotai/humans
cd humans
git checkout v0.1.0
make install
humansd version # must be 0.1.0

humansd config keyring-backend os
humansd config chain-id $CHAIN_ID
humansd init "$NODE_MONIKER" --chain-id $CHAIN_ID

curl -s https://raw.githubusercontent.com/humansdotai/testnets/master/friction/genesis-M1-P2.json > $HOME/.humansd/config/genesis.json

SEEDS="6ce9a9acc23594ec75516617647286fe546f83ca@humans-testnet-seed.itrocket.net:17656"
PEERS=""
sed -i 's|^seeds *=.*|seeds = "'$SEEDS'"|; s|^persistent_peers *=.*|persistent_peers = "'$PEERS'"|' $HOME/.humansd/config/config.toml

sed -i 's|^pruning *=.*|pruning = "custom"|g' $HOME/.humansd/config/app.toml
sed -i 's|^pruning-keep-recent  *=.*|pruning-keep-recent = "1000"|g' $HOME/.humansd/config/app.toml
sed -i 's|^pruning-interval *=.*|pruning-interval = "10"|g' $HOME/.humansd/config/app.toml
sed -i 's|^snapshot-interval *=.*|snapshot-interval = 0|g' $HOME/.humansd/config/app.toml

sed -i 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0.001aheart"|g' $HOME/.humansd/config/app.toml
sed -i 's|^prometheus *=.*|prometheus = true|' $HOME/.humansd/config/config.toml
sed -i 's|^create_empty_blocks_interval *=.*|create_empty_blocks_interval = "30s"|' $HOME/.humansd/config/config.toml
sed -i 's|^timeout_propose *=.*|timeout_propose = "30s"|' $HOME/.humansd/config/config.toml
sed -i 's|^timeout_propose_delta *=.*|timeout_propose_delta = "5s"|' $HOME/.humansd/config/config.toml
sed -i 's|^timeout_prevote *=.*|timeout_prevote = "10s"|' $HOME/.humansd/config/config.toml
sed -i 's|^timeout_prevote_delta *=.*|timeout_prevote_delta = "5s"|' $HOME/.humansd/config/config.toml
sleep 2

sudo tee /etc/systemd/system/humansd.service > /dev/null << EOF
[Unit]
Description=Humans AI Node
After=network-online.target
[Service]
User=$USER
ExecStart=$(which humansd) start
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable humansd
sudo systemctl start humansd

echo -e "Check logs:            sudo journalctl -u $BINARY_NAME -f --no-hostname -o cat"
echo -e "Check synchronization: $BINARY_NAME status 2>&1 | jq .SyncInfo.catching_up"
echo -e "Now create a your own validator"
sleep 2
