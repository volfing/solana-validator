### Подготавливаем сервер

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install nano curl jq logrotate git bc -y
```

### Предварительные проверки сервера

#### Проверяем диски

```bash
curl -sL yabs.sh | bash -s — -ig
```

Смотрим запись/чтение 4к файлов, скорость должна быть выше 500мб/с (это минимум, который возможно даже будет не достаточным).

#### Проверяем износ дисков

Выводим список дисков

```bash
lsblk
```

Далее ставим тулз для проверки дисков + по каждому диску смотрим инфу (износ, температура, объем)

```bash
apt install smartmontools
smartctl -a /dev/nvme0n1
```

#### Проверяем скорость интернета

Скорость должна быть выше 500 мбит/с

```bash
curl -sL yabs.sh | bash -s — -fg
```

#### Проверяем оперативку

```bash
apt install memtester
free -m #получаем объем свободной памяти и тестируем его, если памяти свободной мало, то рестарт сервера
memtester 100G 1
```
### Создаем SWAP файл

```bash
swapoff -a
dd if=/dev/zero of=/swapfile bs=1G count=300
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
```

```bash
nano /etc/fstab
```

Закомментировать символом `#` строку, в которой содержится слово `swap` (существующий SWAP файл)

Добавить в конец файла две строки:

```plaintext
/swapfile none swap sw 0 0
tmpfs /mnt/ramdisk tmpfs nodev,nosuid,noexec,nodiratime,size=300G 0 0
```

### Монтриуем RAM диск

```bash
mkdir -p /mnt/ramdisk
mount /mnt/ramdisk
```

### Устанавливаем Solana CLI

Перед установкой проверить актуальную версию ПО в гитхабе

```bash
touch $HOME/.bash_profile
sh -c "$(curl -sSfL https://release.solana.com/v1.18.17/install)"
. $HOME/.bash_profile
```

### Подготоваливаем валидатор

Перед запуском заменить RPC url на нужный

```bash
solana config set --url http://api.testnet.solana.com
```

```bash
sudo bash -c "cat >/etc/sysctl.d/21-solana-validator.conf <<EOF
# Increase UDP buffer sizes
net.core.rmem_default = 134217728
net.core.rmem_max = 134217728
net.core.wmem_default = 134217728
net.core.wmem_max = 134217728

# Increase memory mapped files limit
vm.max_map_count = 1000000

# Increase number of allowed open file descriptors
fs.nr_open = 1000000
EOF"
```

```bash
sudo sysctl -p /etc/sysctl.d/21-solana-validator.conf
```

Добавить в system.conf строку `DefaultLimitNOFILE=1000000`

```bash
sudo nano /etc/systemd/system.conf

echo "----------- Add solana-nofiles.conf -----------"
sudo bash -c "cat >/etc/security/limits.d/90-solana-nofiles.conf <<EOF
# Increase process file descriptor count limit
* - nofile 1000000
EOF"

echo "----------- Create solana folder -----------"
mkdir -p $HOME/solana
```

### Связываем ключи валидатора

Перенести аккаунт `testnet-validator-keypair.json`, созданный при регистрации, в папку `solana`.

```bash
echo "----------- Add testnet-validator-keypair.json to config -----------"
solana config set --keypair $HOME/solana/testnet-validator-keypair.json

echo "----------- Request airdrop to validator address -----------"
solana airdrop 1

echo "----------- Create Vote account -----------"
solana-keygen new -o $HOME/solana/vote-account-keypair.json

echo "----------- Create Withdrawer account -----------"
solana-keygen new -o $HOME/solana/authorized-withdrawer-keypair.json

echo "----------- Connect all accounts -----------"
solana create-vote-account $HOME/solana/vote-account-keypair.json \
  $HOME/solana/testnet-validator-keypair.json \
  $HOME/solana/authorized-withdrawer-keypair.json --commission 10
```

#### Создание стейк аккаунта

```bash
solana-keygen new -o /root/solana/validator-stake-keypair.json

solana create-stake-account /root/solana/validator-stake-keypair.json 101 --stake-authority /root/solana/authorized-withdrawer-keypair.json --withdraw-authority /root/solana/authorized-withdrawer-keypair.json

solana delegate-stake /root/solana/validator-stake-keypair.json /root/solana/vote-account-keypair.json --stake-authority authorized-withdrawer-keypair.json
```

Сохраняем резервную копию ключей на компьютере.

Удаляем witdrawer ключ:

```bash
rm $HOME/solana/authorized-withdrawer-keypair.json
```

### Настраиваем ротацию логов

```bash
sudo tee <<EOF >/dev/null $HOME/solana/solana.logrotate
$HOME/solana/solana.log {
  rotate 1
  daily
  missingok
  postrotate
    systemctl kill -s USR1 solana.service
  endscript
}
EOF
rm -rf /etc/logrotate.d/sol
ln -s $HOME/solana/solana.logrotate /etc/logrotate.d/sol
sudo systemctl restart logrotate
```

### Создаем сервис валидатора

```bash
nano $HOME/solana/solana.service
```

```bash
[Unit]
Description=Solana Validator
After=network.target
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=1
User=root
LimitNOFILE=1000000
LogRateLimitIntervalSec=0
Environment="SOLANA_METRICS_CONFIG=host=https://metrics.solana.com:8086,db=tds,u=testnet_write,p=c4fa841aa918bf8274e3e2a44d77568d9861b3ea"
ExecStart=/root/solana/validator.sh

[Install]
WantedBy=multi-user.target
```

```bash
nano /root/solana/validator.sh
chmod +x /root/solana/validator.sh
```

```bash
#!/bin/bash
exec /root/.local/share/solana/install/active_release/bin/solana-validator \
    --identity /root/solana/testnet-validator-keypair.json \
    --accounts /mnt/ramdisk/accounts \
    --vote-account /root/solana/vote-account-keypair.json \
    --known-validator 5D1fNXzvv5NjV1ysLjirC4WY92RNsVH18vjmcszZd8on \
    --known-validator 7XSY3MrYnK8vq693Rju17bbPkCN3Z7KvvfvJx4kdrsSY \
    --known-validator Ft5fbkqNa76vnsjYNwjDZUXoTWpP7VYm3mtsaQckQADN \
    --known-validator 9QxCLckBiJc783jnMvXZubK4wH86Eqqvashtrwvcsgkv \
    --only-known-rpc \
    --log /root/solana/solana.log \
    --ledger /root/solana/ledger \
    --rpc-port 8899 \
    --dynamic-port-range 8000-8020 \
    --entrypoint entrypoint.testnet.solana.com:8001 \
    --entrypoint entrypoint2.testnet.solana.com:8001 \
    --entrypoint entrypoint3.testnet.solana.com:8001 \
    --expected-genesis-hash 4uhcVJyU9pJkvQyS88uRDiswHXSCkY3zQawwpjk2NsNY \
    --wal-recovery-mode skip_any_corrupted_record \
    --limit-ledger-size
```

```bash
echo "----------- Add helpful solana tools -----------"
. <(wget -qO- https://raw.githubusercontent.com/SecorD0/utils/main/miscellaneous/insert_variable.sh) -n solana_log -v "tail -fn 100 $HOME/solana/solana.log" -a
. <(wget -qO- https://raw.githubusercontent.com/SecorD0/utils/main/miscellaneous/insert_variable.sh) -n solana_catchup -v "solana catchup --our-localhost --follow --log" -a
. <(wget -qO- https://raw.githubusercontent.com/SecorD0/utils/main/miscellaneous/insert_variable.sh) -n solana_leader_slot -v ". <(wget -qO- https://raw.githubusercontent.com/SecorD0/Solana/main/multi_tool.sh) -ls" -a
```

### Настраиваем мониторинг через telegraf

```bash
echo "----------- Installing telegraf -----------"
curl -s https://repos.influxdata.com/influxdata-archive.key > influxdata-archive.key
echo '943666881a1b8d9b849b74caebf02d3465d6beb716510d86a39f6c8e8dac7515 influxdata-archive.key' | sha256sum -c && cat influxdata-archive.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/influxdata-archive.gpg > /dev/null
echo 'deb [signed-by=/etc/apt/trusted.gpg.d/influxdata-archive.gpg] https://repos.influxdata.com/debian stable main' | sudo tee /etc/apt/sources.list.d/influxdata.list
sudo apt-get update && sudo apt-get install telegraf

echo "----------- Setup telegraf user -----------"
sudo adduser telegraf sudo
sudo adduser telegraf adm
sudo -- bash -c 'echo "telegraf ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers'
sudo cp /etc/telegraf/telegraf.conf /etc/telegraf/telegraf.conf.orig
sudo rm -rf /etc/telegraf/telegraf.conf

echo "----------- Clone repo -----------"
cd $HOME/solana
git clone https://github.com/stakeconomy/solanamonitoring/
cd solanamonitoring
read -p $'\e[40m\e[92mEnter a node moniker:\e[0m ' moniker
```

Создаем конфиг

```bash
sudo nano /etc/telegraf/telegraf.conf
```

```bash
# Global Agent Configuration
[agent]
  hostname = "$moniker" # set this to a name you want to identify your node in the grafana dashboard
  flush_interval = "15s"
  interval = "15s"

# Input Plugins
[[inputs.cpu]]
    percpu = true
    totalcpu = true
    collect_cpu_time = false
    report_active = false
[[inputs.disk]]
    ignore_fs = ["devtmpfs", "devfs"]
[[inputs.diskio]]
[[inputs.mem]]
[[inputs.net]]
[[inputs.system]]
[[inputs.swap]]
[[inputs.netstat]]
[[inputs.processes]]
[[inputs.kernel]]
[[inputs.diskio]]

# Output Plugin InfluxDB
[[outputs.influxdb]]
  database = "metricsdb"
  urls = [ "http://metrics.stakeconomy.com:8086" ] # keep this to send all your metrics to the community dashboard otherwise use http://yourownmonitoringnode:8086
  username = "metrics" # keep both values if you use the community dashboard
  password = "password"

[[inputs.exec]]
  commands = ["sudo su -c /root/solana/solanamonitoring/monitor.sh -s /bin/bash root"] # change home and username to the useraccount your validator runs at
  interval = "5m"
  timeout = "1m"
  data_format = "influx"
  data_type = "integer"
```

Подставляем в `/root/solana/solanamonitoring/monitor.sh` настройки нашего валидатора

Запускаем сервис

```bash
sudo systemctl enable telegraf
sudo systemctl restart telegraf
```

### Устанавливаем mission control

```bash
cd $HOME/solana/
curl -s -L https://raw.githubusercontent.com/Chainflow/solana-mission-control/main/scripts/install_script.sh | bash
source ~/.bashrc
```

Меняем настройки и экспортируем

```bash
export RPC_ENDPOINT="<validator-endpoint>" # Ex - export RPC_ENDPOINT="https://api.rpc.solana.com"
export NETWORK_RPC="<network-endpoint>" # Ex - export NETWORK_RPC="https://api.rpc.com"
export VALIDATOR_NAME="<moniker>" # Your validator name
export PUB_KEY="<node-Public-key>"  # Ex - export PUB_KEY="valmmK7i1AxXeiTtQgQZhQNiXYU84ULeaYF1EH1pa"
export VOTE_KEY="<vote-key>" # Ex - export VOTE_KEY="2oxQJ1qpgUZU9JU84BHaoM1GzHkYfRDgDQY9dpH5mghh"
export TELEGRAM_CHAT_ID=<id> # Ex - export TELEGRAM_CHAT_ID=22128812
export TELEGRAM_BOT_TOKEN="<token>" # Ex - TELEGRAM_BOT_TOKEN="1117273891:AAE12xZU5x4JRj5YSF5LBeu1fPF0T4xj-UI"
export SOLANA_BINARY_PATH="<solana-client-binary-path>" # Ex - export SOLANA_BINARY_PATH="/home/ubuntu/.local/share/solana/install/active_release/bin/solana"
```

```
curl -s -L https://raw.githubusercontent.com/Chainflow/solana-mission-control/main/scripts/tool_installation.sh | bash
```

### Запускаем демон с валидатором

```bash
echo "----------- Link service file -----------"
rm -rf /etc/systemd/system/solana.service
ln -s /root/solana/solana.service /etc/systemd/system/solana.service

echo "----------- Start solana service -----------"
sudo systemctl daemon-reload
sudo systemctl enable solana
sudo systemctl restart solana
```

### Полезные команды

#### Обновлении комиссии вотера

```bash
solana vote-update-commission 81iuTYDaeJ71XFGkPXNUuQ8gNvHQfBxpU7Vj2hzJ9Q4e 10 /root/solana/authorized-withdrawer-keypair.json
```

#### Создание стейк аккаунта

```bash
solana-keygen new -o /root/solana/validator-stake-keypair.json

solana create-stake-account /root/solana/validator-stake-keypair.json 101 --stake-authority /root/solana/authorized-withdrawer-keypair.json --withdraw-authority /root/solana/authorized-withdrawer-keypair.json

solana delegate-stake /root/solana/validator-stake-keypair.json /root/solana/vote-account-keypair.json --stake-authority authorized-withdrawer-keypair.json
```

#### Проверка стейка на вот аккаунте

```bash
solana stakes 81iuTYDaeJ71XFGkPXNUuQ8gNvHQfBxpU7Vj2hzJ9Q4e
```

#### Сеттинг урл сети

```bash
solana config set --url https://api.testnet.solana.com
solana config set --url https://api.mainnet-beta.solana.com
solana config set --url https://api.devnet.solana.com
```

#### Установка ключа ноды по умолчанию

```bash
solana config set --keypair /root/solana/validator-keypair.json
```

