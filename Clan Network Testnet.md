Official link is here https://github.com/ClanNetwork/testnets

Explorer: https://testnet.explorer.testnet.run/Clan%20Network or https://secretnodes.com/clan/chains/playstation-2

## Minimum Hardware Requirements:

- 4GB RAM
- 50GB+ of disk space
- 2 Cores (modern CPU's)


# Instructions


## Install cland
Download binary from github

1. Download the binary for your platform: releases (https://github.com/ClanNetwork/clan-network/releases/tag/v1.0.4-alpha).
```
wget https://github.com/ClanNetwork/clan-network/releases/download/v1.0.4-alpha/clan-network_v1.0.4-alpha_linux_amd64.tar.gz
```

2. Copy it to a location in your PATH, i.e: /usr/local/bin or $HOME/bin.
```
sudo tar -C /usr/local/bin -zxvf clan-network_v1.0.4-alpha_linux_amd64.tar.gz
```

## Verify installation

Enter command: 
```
cland version --long
```
and check this out that everything is OK:

name: Clan-Network  
server_name: clan-networkd  
version: 1.0.4-alpha  
commit: 7a6a92d782c978ac730e337b28d2bc927e809739  
build_tags: ""  
go: go version go1.18 darwin/amd64  


## Let's add variables

Add chainName. Enter by one command.

``` 
CHAIN_ID=playstation-2 ; \
echo $CHAIN_ID ; \
echo 'export chainName='\"${CHAIN_ID}\" >> $HOME/.bash_profile
```

## Set your moniker name
Add your moniker instead of < moniker-name >. Enter by one command.

```
MONIKER_NAME=<moniker-name> ; \
echo $MONIKER_NAME ; \
echo 'export MONIKER_NAME='\"${MONIKER_NAME}\" >> $HOME/.bash_profile
```

## Set persistent peers

Persistent peers will be required to tell your node where to connect to other nodes and join the network. To retrieve the peers for the chosen testnet:

Set the base repo URL for the testnet & retrieve peers
```
CHAIN_REPO="https://raw.githubusercontent.com/ClanNetwork/testnets/main/$CHAIN_ID" && \
export PEERS="$(curl -s "$CHAIN_REPO/persistent-peers.txt")"
```

check it worked
```
echo $PEERS
```

## Set 0 gas prices
```
sed -i.bak -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0uclan\"/" ~/.clan/config/app.toml
```  

## Setting up the Node


* Initialize the chain
```
cland init $MONIKER_NAME --chain-id=$CHAIN_ID
```  

This will generate the following files in ~/.clan/config/
genesis.json
node_key.json
priv_validator_key.json

* Download the genesis file
```
curl https://raw.githubusercontent.com/ClanNetwork/testnets/main/$CHAIN_ID/genesis.json > ~/.clan/config/genesis.json
```  

* Set persistent peers

Using the peers variable we set earlier, we can set the persistent_peers in ~/.clan/config/config.toml:
```
sed -i.bak -e "s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" ~/.clan/config/config.toml
```
* Create a local key pair
```
cland keys add <key-name>
```
* Query the keystore for your public address
```
cland keys show <key-name> -a
```
Replace <key-name> with a key name of your choosing.
If you already have a key from a previous testnet, you can recover it using the mnemonic:
```
cland keys add <key-name> --recover
```
After creating a new key, the key information and seed phrase will be shown. It is essential to write this seed phrase down and keep it in a safe place. The seed phrase is the only way to restore your keys.


* Get some testnet tokens

Testnet tokens can be requested from the Faucet: https://faucet-testnet.clan.network/.

## Start your node
  
  Now that everything is setup and ready to go, you can start your node.
  
  ```
  cland start
  ```
You will need some way to keep the process always running. If you're on linux, you can do this by creating a service. 
Note: First run ```which cland ``` and replace /usr/local/bin/cland with the output if you find any differences 

```
sudo tee /etc/systemd/system/cland.service > /dev/null <<'EOF'
[Unit]
Description=Clan daemon
After=network-online.target

[Service]
User=<your-username>
ExecStart=/usr/local/bin/cland start
Restart=on-failure
RestartSec=3
LimitNOFILE=4096

[Install]
WantedBy=multi-user.target
EOF
``` 
Then update and start the node
  
```
sudo -S systemctl daemon-reload
sudo -S systemctl enable cland
sudo -S systemctl start cland
```
You can check the status with:
```
systemctl status cland
```  

## Syncing the node
After starting the cland daemon, the chain will begin to sync to the network. The time to sync to the network will vary depending on your setup, but could take a very long time. To query the status of your node:
  
(Query via the RPC (default port: 26657))
```  
curl http://localhost:26657/status | jq .result.sync_info.catching_up
```
  
If this command returns true then your node is still catching up. If it returns false then your node has caught up to the network current block and you are safe to proceed to upgrade to a validator node.  

## Upgrade to a validator
  
To upgrade the node to a validator, you will need to submit a create-validator transaction:

```
cland tx staking create-validator \
  --amount 1000000000uclan \
  --commission-max-change-rate "0.1" \
  --commission-max-rate "0.20" \
  --commission-rate "0.1" \
  --min-self-delegation "1" \
  --details "validators write bios too" \
  --pubkey=$(cland tendermint show-validator) \
  --moniker $MONIKER_NAME \
  --chain-id $CHAIN_ID \
  --gas-prices 0uclan \
  --from <key-name>
  ```

## Backup critical files
  
There are certain files that you need to backup to be able to restore your validator if, for some reason, it damaged or lost in some way. Please make a secure backup of the following files located in ~/.clan/config/:
* priv_validator_key.json
* node_key.json
It is recommended that you encrypt the backup of these files.  
  
## Check your node logs:  
  
```
journalctl -u cland -f
```
  
  
