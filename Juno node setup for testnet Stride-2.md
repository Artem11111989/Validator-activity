## Official documentation:
https://github.com/juno-Labs/testnet

## Explorer:
https://testnet.juno.explorers.guru/


# Hardware Requirements

## Minimum Hardware Requirements
4x CPUs; the faster clock speed the better
8GB RAM
100GB of storage (SSD or NVME)
Permanent Internet connection (traffic will be minimal during testnet; 10Mbps will be plenty - for production at least 100Mbps is expected)

## Recommended Hardware Requirements
4x CPUs; the faster clock speed the better
32GB RAM
200GB of storage (SSD or NVME)
Permanent Internet connection (traffic will be minimal during testnet; 10Mbps will be plenty - for production at least 100Mbps is expected)


# Set up your juno fullnode

## Setting up vars

Put here name of your moniker (validator) that will be visible in explorer

```
NODENAME=<YOUR_MONIKER>
```

## Save and import variables into system

```
JUNO_PORT=22
echo "export NODENAME=$NODENAME" >> $HOME/.bash_profile
if [ ! $WALLET ]; then
	echo "export WALLET=wallet" >> $HOME/.bash_profile
fi
echo "export JUNO_CHAIN_ID=uni-3" >> $HOME/.bash_profile
echo "export JUNO_PORT=${JUNO_PORT}" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

## Update packages

```
sudo apt update && sudo apt upgrade -y
```

## Install dependencies

```
sudo apt install curl build-essential git wget jq make gcc tmux chrony -y
```

## Install go

```
if ! [ -x "$(command -v go)" ]; then
  ver="1.18.2"
  cd $HOME
  wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
  sudo rm -rf /usr/local/go
  sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
  rm "go$ver.linux-amd64.tar.gz"
  echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> ~/.bash_profile
  source ~/.bash_profile
fi
```

## Download and build binaries

```
cd $HOME
git clone https://github.com/CosmosContracts/juno
cd juno
git checkout v7.0.0-beta.2
make build && make install
```

## Config app

```
junod config chain-id $JUNO_CHAIN_ID
junod config keyring-backend test
junod config node tcp://localhost:${JUNO_PORT}657
```

## Init app

```
junod init $NODENAME --chain-id $JUNO_CHAIN_ID
```

## Download genesis and addrbook

```
wget -qO $HOME/.juno/config/genesis.json "https://raw.githubusercontent.com/CosmosContracts/testnets/main/uni-3/genesis.json"
```

## Set seeds and peers

```
SEEDS=""
PEERS="26709b3d89548a865ccfd2efac34ef3e9a5b2bc4@135.181.59.162:26656"
sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.juno/config/config.toml
```

## Enable state sync

```
SNAP_RPC="https://juno-testnet-rpc.polkachu.com:443"
LATEST_HEIGHT=$(curl -s $SNAP_RPC/block | jq -r .result.block.header.height); \
BLOCK_HEIGHT=$((LATEST_HEIGHT - 2000)); \
TRUST_HASH=$(curl -s "$SNAP_RPC/block?height=$BLOCK_HEIGHT" | jq -r .result.block_id.hash)
sed -i.bak -E "s|^(enable[[:space:]]+=[[:space:]]+).*$|\1true| ; \
s|^(rpc_servers[[:space:]]+=[[:space:]]+).*$|\1\"$SNAP_RPC,$SNAP_RPC\"| ; \
s|^(trust_height[[:space:]]+=[[:space:]]+).*$|\1$BLOCK_HEIGHT| ; \
s|^(trust_hash[[:space:]]+=[[:space:]]+).*$|\1\"$TRUST_HASH\"| ; \
s|^(seeds[[:space:]]+=[[:space:]]+).*$|\1\"\"|" $HOME/.juno/config/config.toml
```

## Set custom ports

```
sed -i.bak -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:${JUNO_PORT}658\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:${JUNO_PORT}657\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:${JUNO_PORT}060\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:${JUNO_PORT}656\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":${JUNO_PORT}660\"%" $HOME/.juno/config/config.toml
sed -i.bak -e "s%^address = \"tcp://0.0.0.0:1317\"%address = \"tcp://0.0.0.0:${JUNO_PORT}317\"%; s%^address = \":8080\"%address = \":${JUNO_PORT}080\"%; s%^address = \"0.0.0.0:9090\"%address = \"0.0.0.0:${JUNO_PORT}090\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:${JUNO_PORT}091\"%" $HOME/.juno/config/app.toml
```

## Config pruning

```
pruning="custom"
pruning_keep_recent="100"
pruning_keep_every="0"
pruning_interval="50"
sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.juno/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" $HOME/.juno/config/app.toml
sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/" $HOME/.juno/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $HOME/.juno/config/app.toml
```

## Set minimum gas price and timeout commit

```
sed -i -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0ujunox\"/" $HOME/.juno/config/app.toml
```

## Enable prometheus

```
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.juno/config/config.toml
```

## Reset chain data

```
junod tendermint unsafe-reset-all --home $HOME/.juno
```

## Create service

```
sudo tee /etc/systemd/system/junod.service > /dev/null <<EOF
[Unit]
Description=juno
After=network-online.target

[Service]
User=$USER
ExecStart=$(which junod) start --home $HOME/.juno
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

## Register and start service

```
sudo systemctl daemon-reload
sudo systemctl enable junod
sudo systemctl restart junod && sudo journalctl -u junod -f -o cat
```

# Post installation

When installation is finished you need to load variables into system

```
source $HOME/.bash_profile
```

Next you have to make sure your validator is syncing blocks. Use command below to check synchronization status

```
junod status 2>&1 | jq .SyncInfo
```

## Create wallet

Don’t forget to save the mnemonic
```
junod keys add $WALLET
```

(OPTIONAL) To recover your wallet using seed phrase

```
junod keys add $WALLET --recover
```

## Save wallet info

Add wallet and valoper address into variables
```
JUNO_WALLET_ADDRESS=$(junod keys show $WALLET -a)
JUNO_VALOPER_ADDRESS=$(junod keys show $WALLET --bech val -a)
echo 'export JUNO_WALLET_ADDRESS='${JUNO_WALLET_ADDRESS} >> $HOME/.bash_profile
echo 'export JUNO_VALOPER_ADDRESS='${JUNO_VALOPER_ADDRESS} >> $HOME/.bash_profile
source $HOME/.bash_profile
```

## Fund your wallet

In order to create validator first you need to fund your wallet with testnet tokens. To top up your wallet join juno discord server and navigate to #faucet to request test tokens

To request a faucet grant:

```
$request <YOUR_WALLET_ADDRESS>
```

## Create validator

Please make sure that you have at least 1 junox (1 junox is equal to 1000000 ujunox) and your node is synchronized

To check your wallet balance:

```
junod query bank balances $JUNO_WALLET_ADDRESS
```

If your wallet does not show any balance than probably your node is still syncing. Please wait until it finish to synchronize and then continue

Create validator

```
junod tx staking create-validator \
  --amount 10000000ujunox \
  --from $WALLET \
  --commission-max-change-rate "0.01" \
  --commission-max-rate "0.2" \
  --commission-rate "0.07" \
  --min-self-delegation "1" \
  --pubkey  $(junod tendermint show-validator) \
  --moniker $NODENAME \
  --chain-id $JUNO_CHAIN_ID
  ```
  
  
