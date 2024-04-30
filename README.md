# Union
Union-Testnet-8
![babylon](https://pbs.twimg.com/profile_banners/1687201209306824704/1712069560/1500x500)

## Sistem gereksinimleri:

**Ubuntu 22.04+**

NODE TİPİ | CPU     | RAM      | SSD     |
| ------------- | ------------- | ------------- | -------- |
| Union | 4          | 8         | 160  |
  
# Kurulum

## Güncellemeler
```
sudo apt -q update
sudo apt -qy install curl git jq lz4 build-essential fail2ban ufw
sudo apt -qy upgrade
```
## Go Yükleyelim.
```
sudo rm -rf /usr/local/go
curl -Ls https://go.dev/dl/go1.21.4.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
eval $(echo 'export PATH=$PATH:/usr/local/go/bin' | sudo tee /etc/profile.d/golang.sh)
eval $(echo 'export PATH=$PATH:$HOME/go/bin' | tee -a $HOME/.profile)
```

## Binary Dosyalarını İndirelim

```
# Proje Binary dosyalarını indiriyoruz.
cd $HOME
wget -O uniond https://snap.nodex.one/union-testnet/uniond-v0.22.0
chmod +x uniond

# cosmovisor için binary dosyalarını yükleyelim.
mkdir -p $HOME/.union/cosmovisor/genesis/bin
mv uniond $HOME/.union/cosmovisor/genesis/bin/

# symlinks oluşturalım
sudo ln -s $HOME/.union/cosmovisor/genesis $HOME/.union/cosmovisor/current -f
sudo ln -s $HOME/.union/cosmovisor/current/bin/uniond /usr/local/bin/uniond -f
```
## Cosmovisor ve System dosyalarını oluşturalım.

```
# Cosmovisor indiriyoruz.
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.5.0

# Service dosyasını oluşturalım.
sudo tee /etc/systemd/system/union.service > /dev/null << EOF
[Unit]
Description=union node service
After=network-online.target
 
[Service]
User=$USER
ExecStart=$(which cosmovisor) run start --home $HOME/.union
Restart=on-failure
RestartSec=10
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.union"
Environment="DAEMON_NAME=uniond"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:$HOME/.union/cosmovisor/current/bin"
 
[Install]
WantedBy=multi-user.target
EOF
```

```
sudo systemctl daemon-reload
sudo systemctl enable union
```

## Node Kuralım

> `$MONIKER` yerine validator isminizi girmeyi unutmayın.

```
# Workaround mandatory home argument
echo 'alias uniond="uniond --home ~/.union/"' >> ~/.bashrc
source ~/.bashrc

# Initialize the node
uniond init $MONIKER --chain-id union-testnet-8

# Download genesis and addrbook
curl -Ls https://snap.nodex.one/union-testnet/genesis.json > $HOME/.union/config/genesis.json
curl -Ls https://snap.nodex.one/union-testnet/addrbook.json > $HOME/.union/config/addrbook.json

# Add seeds
sed -i -e "s|^seeds *=.*|seeds = \"d1d43cc7c7aef715957289fd96a114ecaa7ba756@testnet-seeds.nodex.one:23110\"|" $HOME/.union/config/config.toml

# Set minimum gas price
sed -i -e "s|^minimum-gas-prices *=.*|minimum-gas-prices = \"0muno\"|" $HOME/.union/config/app.toml

# Set pruning
sed -i \
  -e 's|^chain-id *=.*|chain-id = "union-testnet-8"|' \
  -e 's|^keyring-backend *=.*|keyring-backend = "test"|' \
  -e 's|^node *=.*|node = "tcp://localhost:23157"|' \
  $HOME/.union/config/client.toml

# Port Değişimi (Opsiyonel)-İsterseniz geçebilirsiniz. 
sed -i -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:23158\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:23157\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:23160\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:23156\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":23166\"%" $HOME/.union/config/config.toml
sed -i -e "s%^address = \"tcp://0.0.0.0:1317\"%address = \"tcp://0.0.0.0:23117\"%; s%^address = \":8080\"%address = \":23180\"%; s%^address = \"0.0.0.0:9090\"%address = \"0.0.0.0:23190\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:23191\"%; s%:8545%:23145%; s%:8546%:23146%; s%:6065%:23165%" $HOME/.union/config/app.toml
```

## Snapshot İndiriyoruz.
```
sudo apt update
sudo apt-get install snapd lz4 -y

# Node durdurup dataları resetliyoruz
sudo systemctl stop union
cp $HOME/.union/data/priv_validator_state.json $HOME/.union/priv_validator_state.json.backup
rm -rf $HOME/.union/data
rm -rf $HOME/.union/wasm
uniond tendermint unsafe-reset-all --home ~/.union/ --keep-addr-book

# Snapshot'ı indiriyoruz.
SNAP_NAME=$(curl -s https://ss-t.union.nodestake.org/ | egrep -o ">20.*\.tar.lz4" | tr -d ">")
curl -o - -L https://ss-t.union.nodestake.org/${SNAP_NAME}  | lz4 -c -d - | tar -x -C $HOME/.union
mv $HOME/.union/priv_validator_state.json.backup $HOME/.union/data/priv_validator_state.json
```

## Ağı başlatıyoruz.
```
sudo systemctl start union && sudo journalctl -u union -f --no-hostname -o cat
```

> [BURADAN](https://explorer.coinhunterstr.com/union) son blok sayısını kontrol edebilirsiniz. Ağ ile eşleştikten sonra, Validator oluşturma adımına geçin.

# Validator Oluşturma

> İlk olarak [DİSCORD](https://discord.gg/union-build) kanalına giriyoruz ve oluşturacağımız cüzdan adresine token istiyoruz. Faucet olmadığı için ekibin size token göndermesi gerekiyor. 

## Cüzdan oluşturuyoruz.
> `wallet` yerine istediğiniz bir isim yazıyoruz. Sonrasında size verilen adresi ve gizli kelimeleri bir yere not etmeyi unutmayın.

```
uniond keys add wallet
```
## Pubkey
> Öncelikle `pubkey`mizi öğreniyoruz.

> Aşağıdaki komutu çalıştırınca;  `{"@type":"/cosmos.crypto.ed25519.PubKey","key":"keynumbersss"}` buna benzer bir çıktı alıyoruz. Bir yere not edelim.

```
uniond tendermint show-validator --home /root/.union
```
```
cd $HOME
```
```
nano validator.json
```
> Aşağıdaki komutu kendinize göre düzenlemeniz gerekiyor.

> `moniker` yerine Validator isminizi giriyorsunuz ( hepsinde"" kalacak!)

> `identity` yerine keybase.io sitesine avatarınızı yükleyip. oradan verilen ID'yi ekliyoruz. 

> `website` bölümüne istediğiniz bir uzantı girebilirsiniz. github, twitter etc.

> `security` bölümüne mail adresinizi girebilirsiniz.

> `details` bölümüne istediğiniz bir şey yazabilirsiniz.

```
{
  "pubkey": <PUBKEY>,
  "amount": "1000000muno",
  "moniker": "<MONIKER>",
  "identity": "optional identity signature (ex. UPort or Keybase)",
  "website": "validator's (optional) website",
  "security": "validator's (optional) security contact email",
  "details": "validator's (optional) details",
  "commission-rate": "0.1",
  "commission-max-rate": "0.2",
  "commission-max-change-rate": "0.01",
  "min-self-delegation": "1"
}
```
> Bunları düzenledikten sonra terminale kopyalayıp ekliyoruz. Sonrasında CTRL X Y enter ile çıkıyoruz.

```
sudo systemctl restart union
```

> `wallet` yerine kendi cüzdan adınızı girmeyi unutmayın.

```
uniond --home $HOME/.union tx staking create-validator $HOME/validator.json --from wallet --chain-id union-testnet-8 --fees 0muno -y 
```

> Tüm adımları yaptıktan sonra [BURADAN](https://explorer.coinhunterstr.com/union) kendi validatorünüzü kontrol edebilirsiniz. 











