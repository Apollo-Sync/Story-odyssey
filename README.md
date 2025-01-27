**Manual Installation**
```
Recommended Hardware: 6 Cores, 16GB RAM, 400GB of storage (NVME), 100 Mb/s
```

**install dependencies, if needed**
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl git wget htop tmux build-essential jq make lz4 gcc unzip -y
```
**install go, if needed**
```
cd $HOME
VER="1.22.3"
wget "https://golang.org/dl/go$VER.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$VER.linux-amd64.tar.gz"
rm "go$VER.linux-amd64.tar.gz"
[ ! -f ~/.bash_profile ] && touch ~/.bash_profile
echo "export PATH=$PATH:/usr/local/go/bin:~/go/bin" >> ~/.bash_profile
source $HOME/.bash_profile
[ ! -d ~/go/bin ] && mkdir -p ~/go/bin
```

**set vars**
```
echo "export MONIKER="test"" >> $HOME/.bash_profile
echo "export STORY_CHAIN_ID="odyssey-0"" >> $HOME/.bash_profile
echo "export STORY_PORT="52"" >> $HOME/.bash_profile
source $HOME/.bash_profile
```

**download binaries**
```
cd $HOME
wget -O geth https://github.com/piplabs/story-geth/releases/download/v0.11.0/geth-linux-amd64
chmod +x $HOME/geth
mv $HOME/geth ~/go/bin/
[ ! -d "$HOME/.story/story" ] && mkdir -p "$HOME/.story/story"
[ ! -d "$HOME/.story/geth" ] && mkdir -p "$HOME/.story/geth"
```

**install Story**
```
cd $HOME
rm -rf story
git clone https://github.com/piplabs/story
cd story
git checkout v0.13.2
go build -o story ./client 
mv $HOME/story/story $HOME/go/bin/
```

**init story app**
```
story init --moniker test --network odyssey
```

**set seeds and peers**
```
SEEDS="434af9dae402ab9f1c8a8fc15eae2d68b5be3387@story-testnet-seed.itrocket.net:29656"
PEERS="c2a6cc9b3fa468624b2683b54790eb339db45cbf@story-testnet-peer.itrocket.net:26656,9f429c4f6cdeabfdfa413c061a4c4e42d1ce2f3f@65.108.121.227:12156,1dede3da666c4eba0508db2588ed6404008d322a@37.27.225.240:26656,2ac7e5089cb547df5444105d5045892fc34ace06@135.181.139.234:26656,75ac7b193e93e928d6c83c273397517cb60603c0@3.142.16.95:26656,bbd35223a30922b4b83b4c0d080f33e564e8624a@89.58.17.242:25556,f13f0bf2b5aea85ca15677879e7f65ee02886111@49.12.122.53:27656,51488526c0b7b16173713b63b3bf904376afd630@135.181.181.59:26656,6957f98d23b11373854736710af5fdd88384948c@38.242.221.185:26656"
sed -i -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*seeds *=.*/seeds = \"$SEEDS\"/}" \
       -e "/^\[p2p\]/,/^\[/{s/^[[:space:]]*persistent_peers *=.*/persistent_peers = \"$PEERS\"/}" $HOME/.story/story/config/config.toml
```

**download genesis and addrbook**
```
wget -O $HOME/.story/story/config/genesis.json https://raw.githubusercontent.com/Apollo-Sync/Story-odyssey/refs/heads/main/addrbook.json
wget -O $HOME/.story/story/config/addrbook.json https://raw.githubusercontent.com/Apollo-Sync/Story-odyssey/refs/heads/main/genesis.json
```

**set custom ports in story.toml file**
```
sed -i.bak -e "s%:1317%:${STORY_PORT}317%g;
s%:8551%:${STORY_PORT}551%g" $HOME/.story/story/config/story.toml
```

**set custom ports in config.toml file**
```
sed -i.bak -e "s%:26658%:${STORY_PORT}658%g;
s%:26657%:${STORY_PORT}657%g;
s%:26656%:${STORY_PORT}656%g;
s%^external_address = \"\"%external_address = \"$(wget -qO- eth0.me):${STORY_PORT}656\"%;
s%:26660%:${STORY_PORT}660%g" $HOME/.story/story/config/config.toml
```

**enable prometheus and disable indexing**
```
sed -i -e "s/prometheus = false/prometheus = true/" $HOME/.story/story/config/config.toml
sed -i -e "s/^indexer *=.*/indexer = \"null\"/" $HOME/.story/story/config/config.toml
```

**create geth servie file**
```
sudo tee /etc/systemd/system/story-geth.service > /dev/null <<EOF
[Unit]
Description=Story Geth daemon
After=network-online.target

[Service]
User=$USER
ExecStart=$HOME/go/bin/geth --odyssey --syncmode full --http --http.api eth,net,web3,engine --http.vhosts '*' --http.addr 0.0.0.0 --http.port ${STORY_PORT}545 --authrpc.port ${STORY_PORT}551 --ws --ws.api eth,web3,net,txpool --ws.addr 0.0.0.0 --ws.port ${STORY_PORT}546
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```

**create story service file**
```
sudo tee /etc/systemd/system/story.service > /dev/null <<EOF
[Unit]
Description=Story Service
After=network.target

[Service]
User=$USER
WorkingDirectory=$HOME/.story/story
ExecStart=$(which story) run

Restart=on-failure
RestartSec=5
LimitNOFILE=65535
[Install]
WantedBy=multi-user.target
EOF
```

**download snapshots**
1. backup priv_validator_state.json
```
cp $HOME/.story/story/data/priv_validator_state.json $HOME/.story/story/priv_validator_state.json.backup
```
2. remove old data and unpack Story snapshot
```
rm -rf $HOME/.story/story/data
curl https://apollo-sync.com/testnet/story/story_2025-01-26_2257155_snap.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.story/story
```

**restore priv_validator_state.json**
```
mv $HOME/.story/story/priv_validator_state.json.backup $HOME/.story/story/data/priv_validator_state.json
```

**delete geth data and unpack Geth snapshot**
```
rm -rf $HOME/.story/geth/odyssey/geth/chaindata
mkdir -p $HOME/.story/geth/odyssey/geth
curl https://apollo-sync.com/testnet/story/geth_story_2025-01-26_2257155_snap.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.story/geth/odyssey/geth
```
**enable and start geth, story**
```
sudo systemctl daemon-reload
sudo systemctl enable story story-geth
sudo systemctl restart story-geth && sleep 5 && sudo systemctl restart story
```

**check logs**
```
journalctl -u story -u story-geth -f
```
