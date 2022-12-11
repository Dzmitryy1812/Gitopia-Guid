# Gitopia-Guid

Обновляем и устанавливаем необходимые пакеты:
```
sudo apt update && sudo apt upgrade -y
sudo apt install curl tar wget clang pkg-config libssl-dev jq build-essential bsdmainutils git make ncdu gcc git jq chrony liblz4-tool -y
```
Устанавливаем GO:
```
ver="1.18.2"
cd $HOME
wget "https://golang.org/dl/go$ver.linux-amd64.tar.gz"
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go$ver.linux-amd64.tar.gz"
rm "go$ver.linux-amd64.tar.gz"
echo "export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin" >> ~/.bash_profile
source ~/.bash_profile
go version
```
Версия GO должна быть 1.18.2

Создаем переменные
```
NODENAME="Имя вашей ноды"
```
Далее сохраняем переменные в баш:
```
PORT=15
echo "export NODENAME=$NODENAME" >> $HOME/.bash_profile
echo "export WALLET=wallet" >> $HOME/.bash_profile
echo "export GCHAIN_ID=gitopia-janus-testnet-2" >> $HOME/.bash_profile
echo "export GPORT=${GPORT}" >> $HOME/.bash_profile
source $HOME/.bash_profile
```
Скачиваем и устанавливаем бинарник:
```
cd $HOME
curl https://get.gitopia.com | bash
git clone -b v1.2.0 gitopia://gitopia/gitopia
cd gitopia && make install
```
Начинаем инициацию
```
 gitopiad init $NODENAME --chain-id $GCHAIN_ID
```
Записываем чейн и keyring-backend в конфиг, меняем порт
```
gitopiad config chain-id $GCHAIN_ID
gitopiad config keyring-backend test
gitopiad config node tcp://localhost:${GPORT}657
```
Скачиваем генезис файл
```
wget https://server.gitopia.com/raw/gitopia/testnets/master/gitopia-janus-testnet-2/genesis.json.gz
gunzip genesis.json.gz
mv genesis.json $HOME/.gitopia/config/genesis.json
```
Скачиваем addrbook
```
wget -qO $HOME/.gitopia/config/addrbook.json "https://raw.githubusercontent.com/sergiomateiko/addrbooks/main/gitopia/addrbook.json"
```
Меняем порты по желанию 
```
sed -i.bak -e "s%^proxy_app = \"tcp://127.0.0.1:26658\"%proxy_app = \"tcp://127.0.0.1:${GPORT}658\"%; s%^laddr = \"tcp://127.0.0.1:26657\"%laddr = \"tcp://127.0.0.1:${GPORT}657\"%; s%^pprof_laddr = \"localhost:6060\"%pprof_laddr = \"localhost:${GPORT}060\"%; s%^laddr = \"tcp://0.0.0.0:26656\"%laddr = \"tcp://0.0.0.0:${GPORT}656\"%; s%^prometheus_listen_addr = \":26660\"%prometheus_listen_addr = \":${GPORT}660\"%" $HOME/.gitopia/config/config.toml
sed -i.bak -e "s%^address = \"tcp://0.0.0.0:1317\"%address = \"tcp://0.0.0.0:${GPORT}317\"%; s%^address = \":8080\"%address = \":${GPORT}080\"%; s%^address = \"0.0.0.0:9090\"%address = \"0.0.0.0:${GPORT}090\"%; s%^address = \"0.0.0.0:9091\"%address = \"0.0.0.0:${GPORT}091\"%" $HOME/.gitopia/config/app.toml
```
Отключаем индексацию
```
indexer="null"
sed -i -e "s/^indexer *=.*/indexer = \"$indexer\"/" $HOME/.gitopia/config/config.toml
```
Ставим минимальную цену газа
```
sed -i -e "s/^minimum-gas-prices *=.*/minimum-gas-prices = \"0.001utlore\"/" $HOME/.gitopia/config/app.toml
```
Настраиваем прунинг по желанию
```
pruning="custom" 
pruning_keep_recent="100" 
pruning_keep_every="0" 
pruning_interval="50" 
sed -i -e "s/^pruning *=.*/pruning = \"$pruning\"/" $HOME/.gitopia/config/app.toml
sed -i -e "s/^pruning-keep-recent *=.*/pruning-keep-recent = \"$pruning_keep_recent\"/" $HOME/.gitopia/config/app.toml
sed -i -e "s/^pruning-keep-every *=.*/pruning-keep-every = \"$pruning_keep_every\"/" $HOME/.gitopia/config/app.toml
sed -i -e "s/^pruning-interval *=.*/pruning-interval = \"$pruning_interval\"/" $HOME/.gitopia/config/app.toml
```
Ставим seed'ы и записываем их
```
seeds=""
peers="93b218e53303ca91b7bb4f22edbb858496b1b434@65.108.6.45:60756,fbe3b1e34e1dfe9ae2cd0db471b0a807bbb3c5f2@65.109.90.178:11356"
sed -i -e "s/^seeds *=.*/seeds = \"$SEEDS\"/; s/^persistent_peers *=.*/persistent_peers = \"$PEERS\"/" $HOME/.gitopia/config/config.toml
```
Сбрасываем данные цепи
```
gitopiad tendermint unsafe-reset-all
sudo tee /etc/systemd/system/gitopiad.service > /dev/null <<EOF
[Unit]
Description=gitopiaNode
After=network-online.target

[Service]
User=$USER
ExecStart=$(which gitopiad) start
Restart=on-failure
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF
```                                                               
Запускаем сервисный файл
```
sudo systemctl daemon-reload && sudo systemctl enable gitopiad && sudo systemctl restart gitopiad
```
Посмотреть логи
```
sudo journalctl -u gitopiad -f -o cat
```
Эксплорер можете найти [здесь](https://explorer.gitopia.com/)

Создаем кошелек
```
gitopiad keys add $WALLET
```
Кран - Заходим на [сайт](https://gitopia.com/) в личный кабинет, подключаем кошелек, сид фразу берем от созданного ранее кошелька для использования крана, запрашиваем токены.

Создаем переменную с адресом для удобства
```
GADDRESS=$(gitopiad keys show $WALLET -a)
echo 'export EADDRESS='${GADDRESS} >> $HOME/.bash_profile
gitopiad query bank balances $GADDRESS
GVALOPER=$(gitopiad keys show $WALLET --bech val -a)
echo 'export EVALOPER='${GVALOPER} >> $HOME/.bash_profile
source $HOME/.bash_profile
```
Создаем валидатора
```
gitopiad tx staking create-validator \
 --amount=1000000utlore \
 --pubkey=$(gitopiad tendermint show-validator) \
 --moniker=$NODENAME \
 --chain-id=$GCHAIN_ID \
 --commission-rate="0.10" \
 --commission-max-rate="0.20" \
 --commission-max-change-rate="0.01" \
 --min-self-delegation="1000000" \
 --gas="auto" \
 --gas-prices="0.002utlore" \
 --gas-adjustment="1.3" \
 --from=$WALLET
  ```
Редактируем информацию у валидатора (по желанию)
```
gitopiad tx staking edit-validator \
--from=$WALLET \
--website="https://t.me/cryptorussianbears" \
--identity="2D5D009F1C1AAD3A" \
--details="Early adopter cryptoenthusiast" \
--chain-id=$GCHAIN_ID \
--fees=300utlore \
--gas-adjustment="1"
 ```
После запуска вадидатора идем заполнять [форму](https://airtable.com/shrMQFJxcsMD0XV2M), указываем все свои данные.

Для быстрой синзронизации ноды делаем снепшот по желанию(взят с kjnodes.com) 
```
sudo systemctl stop gitopiad
cp $HOME/.gitopia/data/priv_validator_state.json $HOME/.gitopia/priv_validator_state.json.backup
rm -rf $HOME/.gitopia/data
```
```
curl -L https://snapshots.kjnodes.com/gitopia-testnet/snapshot_latest.tar.lz4 | lz4 -dc - | tar -xf - -C $HOME/.gitopia
mv $HOME/.gitopia/priv_validator_state.json.backup $HOME/.gitopia/data/priv_validator_state.json
```
```
sudo systemctl start gitopiad && journalctl -u gitopiad -f --no-hostname -o cat
```
