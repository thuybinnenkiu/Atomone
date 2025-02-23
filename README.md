Dependencies Installation

**Install dependencies for building from source**
```
sudo apt update
sudo apt install -y curl git jq lz4 build-essential
```

**Install Go**
```
sudo rm -rf /usr/local/go
curl -L https://go.dev/dl/go1.22.7.linux-amd64.tar.gz | sudo tar -xzf - -C /usr/local
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> $HOME/.profile
source .profile
```

Node Installation

**Clone project repository**
```
cd && rm -rf atomone
git clone https://github.com/atomone-hub/atomone
cd atomone
git checkout v1.0.0
```

**Build binary**
```
make install
```

**Prepare cosmovisor directories**
```
mkdir -p $HOME/.atomone/cosmovisor/genesis/bin
ln -s $HOME/.atomone/cosmovisor/genesis $HOME/.atomone/cosmovisor/current -f
```

**Copy binary to cosmovisor directory**
```
cp $(which atomoned) $HOME/.atomone/cosmovisor/genesis/bin
```

**Set node CLI configuration**
```
atomoned config chain-id atomone-1
atomoned config keyring-backend file
atomoned config node tcp://localhost:29957
```
**Initialize the node**
```
atomoned init "Your Node Name" --chain-id atomone-1
```

**Download genesis and addrbook files**
```
curl -L https://snapshots.nodejumper.io/atomone/genesis.json > $HOME/.atomone/config/genesis.json
curl -L https://snapshots.nodejumper.io/atomone/addrbook.json > $HOME/.atomone/config/addrbook.json
```

**Set seeds**
```
sed -i -e 's|^seeds *=.*|seeds = "8542cd7e6bf9d260fef543bc49e59be5a3fa9074@seed.publicnode.com:26656, 10ed1e176d874c8bb3c7c065685d2da6a4b86475@seed-atomone.ibs.team:16684,ebc272824924ea1a27ea3183dd0b9ba713494f83@atomone-mainnet-seed.autostake.com:27396,258f523c96efde50d5fe0a9faeea8a3e83be22ca@seed.atomone-1.atomone.aviaone.com:10293,57e11247cd5c12420c37e68fe3157bc51ca84ca3@mainnet.seednode.citizenweb3.com:26756,f19d9e0f8d48119aa4cafde65de923ae2c29181a@atomone-mainnet-seed.itrocket.net:61656,9dac79c27e4dba77cb22b0b25933fee6c8121cf7@72.46.84.243:26656,ebc272824924ea1a27ea3183dd0b9ba713494f83@atomone-mainnet-peer.autostake.com:27396,eec666a28728bad46a31fd352cc61bc5998efc1d@116.202.156.139:26706,9524bac2c6be4d8b747e6b75d9b924000f9f6835@atomone.peer.stavr.tech:23456"|' $HOME/.atomone/config/config.toml
```

**Set minimum gas price**
```
sed -i -e 's|^minimum-gas-prices *=.*|minimum-gas-prices = "0.006uatone"|' $HOME/.atomone/config/app.toml
```

**Set pruning**
```
sed -i \
  -e 's|^pruning *=.*|pruning = "custom"|' \
  -e 's|^pruning-keep-recent *=.*|pruning-keep-recent = "100"|' \
  -e 's|^pruning-interval *=.*|pruning-interval = "17"|' \
  $HOME/.atomone/config/app.toml
```

**Enable prometheus**
```
sed -i -e 's|^prometheus *=.*|prometheus = true|' $HOME/.atomone/config/config.toml
```

**Change ports**
```
sed -i -e "s%:1317%:29917%; s%:8080%:29980%; s%:9090%:29990%; s%:9091%:29991%; s%:8545%:29945%; s%:8546%:29946%; s%:6065%:29965%" $HOME/.atomone/config/app.toml
sed -i -e "s%:26658%:29958%; s%:26657%:29957%; s%:6060%:29960%; s%:26656%:29956%; s%:26660%:29961%" $HOME/.atomone/config/config.toml
```

**Download latest chain data snapshot**
```
curl "https://snapshots.nodejumper.io/atomone/atomone_latest.tar.lz4" | lz4 -dc - | tar -xf - -C "$HOME/.atomone"
```

**Install Cosmovisor**
```
go install cosmossdk.io/tools/cosmovisor/cmd/cosmovisor@v1.7.0
```

# Create a service
sudo tee /etc/systemd/system/atomone.service > /dev/null << EOF
[Unit]
Description=AtomOne node service
After=network-online.target
[Service]
User=$USER
WorkingDirectory=$HOME/.atomone
ExecStart=$(which cosmovisor) run start
Restart=on-failure
RestartSec=5
LimitNOFILE=65535
Environment="DAEMON_HOME=$HOME/.atomone"
Environment="DAEMON_NAME=atomoned"
Environment="UNSAFE_SKIP_BACKUP=true"
Environment="DAEMON_ALLOW_DOWNLOAD_BINARIES=true"
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable atomone.service

# Start the service and check the logs
sudo systemctl start atomone.service
sudo journalctl -u atomone.service -f --no-hostname -o cat
Secure Server Setup (Optional)

# generate ssh keys, if you don't have them already, DO IT ON YOUR LOCAL MACHINE
ssh-keygen -t rsa

# save the output, we'll use it later on instead of YOUR_PUBLIC_SSH_KEY
cat ~/.ssh/id_rsa.pub
# upgrade system packages
sudo apt update
sudo apt upgrade -y

# add new admin user
sudo adduser admin --disabled-password -q

# upload public ssh key, replace YOUR_PUBLIC_SSH_KEY with the key above
mkdir /home/admin/.ssh
echo "YOUR_PUBLIC_SSH_KEY" >> /home/admin/.ssh/authorized_keys
sudo chown admin: /home/admin/.ssh
sudo chown admin: /home/admin/.ssh/authorized_keys

echo "admin ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers

# disable root login, disable password authentication, use ssh keys only
sudo sed -i 's|^PermitRootLogin .*|PermitRootLogin no|' /etc/ssh/sshd_config
sudo sed -i 's|^ChallengeResponseAuthentication .*|ChallengeResponseAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PasswordAuthentication .*|PasswordAuthentication no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PermitEmptyPasswords .*|PermitEmptyPasswords no|' /etc/ssh/sshd_config
sudo sed -i 's|^#PubkeyAuthentication .*|PubkeyAuthentication yes|' /etc/ssh/sshd_config

sudo systemctl restart sshd

# install fail2ban
sudo apt install -y fail2ban

# install and configure firewall
sudo apt install -y ufw
sudo ufw default allow outgoing
sudo ufw default deny incoming
sudo ufw allow ssh
sudo ufw allow 9100
sudo ufw allow 26656

# make sure you expose ALL necessary ports, only after that enable firewall
sudo ufw enable

# make terminal colorful
sudo su - admin
source <(curl -s https://raw.githubusercontent.com/nodejumper-org/cosmos-scripts/master/utils/enable_colorful_bash.sh)

# update servername, if needed, replace YOUR_SERVERNAME with wanted server name
sudo hostnamectl set-hostname YOUR_SERVERNAME

# now you can logout (exit) and login again using ssh admin@YOUR_SERVER_IP
