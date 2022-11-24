# Инструкция по установке quark-1 testnet

> **Примечание: мы предполагаем, что вы уже инициализировали тестовый смарт-контракт. Вам понадобится адрес контракта, чтобы поместить его в `config.toml` для того, чтобы ретранслятор работал только с пакетами, приходящими от этого контракта.**

Это руководство предназначено для подключения к сети cosmoshub testnet (theta-testnet-001). Чтобы подключить hermes с помощью juno, делаем то же самое (начиная с шага 2), но меняем имя пользователя и имя сервиса, а также правим конфиг сети для juno в файле `config.toml`. В этом руководстве hermes устанавливается как демон.

## 1. Ставим Hermes v1.0.0

```
curl -L "https://github.com/informalsystems/ibc-rs/releases/download/v1.0.0/hermes-v1.0.0-${PLATFORM}-unknown-linux-gnu.tar.gz" > hermes.tar.gz && \
    tar -C ./ -vxzf hermes.tar.gz && \
    rm -f hermes.tar.gz  && \
    sudo mv ./hermes /usr/local/bin/ && \
    sudo chgrp root /usr/local/bin/hermes && \
    sudo chown root /usr/local/bin/hermes
```

Смотрим, что он работает и версия в порядке (Должна быть `hermes 1.0.0+ed4dd8c`)

`$ hermes --version`

## 2. Добавляем пользователя

```
$ sudo useradd -m ibc-cosmoshub-rly
$ sudo su ibc-cosmoshub-rly
$ cd ~/
$ mkdir ~/.hermes
$ exit # выходим в учетку рута
```

## 3. Создаем сервис

`vim /etc/systemd/system/neutron-ibc-cosmoshub-relayer.service`

```
[Unit]
Description=Neutron IBC CosmosHub Relayer
After=network.target

[Service]
User=ibc-cosmoshub-rly
ExecStart=/usr/local/bin/hermes start

[Install]
WantedBy=multi-user.target
```

## 4. Настройка ретранслятора

Копируем пример [конфига](https://github.com/neutron-org/testnets/blob/main/quark/ibc-relayer/config.toml) в `~/.hermes/config.toml` **и заполняем недостающие параметры.**

`wget https://raw.githubusercontent.com/neutron-org/testnets/main/quark/ibc-relayer/config.toml -o ~/.hermes/config.toml`


> Не забудьте йопд, заполнить недостающие параметры (отмечены комментариями TODO).

> От себя добавлю, в конфиге необходимо вписать три параметра, коротко об них:
 
    `RPC` смотрим в `config.toml`, строка 86 > блок RPC Server Configuration Options, *657-й порт
    `GRPC` смотрим в `app.toml`, строка 156 > блок gRPC Configuration, порт *9090
    `websocket` выглядит как `ws://[rpc_addr]/websocket`, хз пока что это значит 
    
> (upd похоже что так: если RPC `http://127.0.0.1:26657` то websocket = `ws://127.0.0.1:26657/websocket`)

ПРИМЕЧАНИЕ: `websocket_addr` должен начинаться с протокола ws|wss

Проверяем корректность конфигурации:

`$ hermes health-check`

## 5. Добавление ключей для релея

Генерируем свои мнемоники для учетных записей и заполняем их в командах bash, приведенных ниже:

```
$ sudo su ibc-cosmoshub-rly
$ export NEUTRON_MNEMONIC="TODO"
$ export TARGET_CHAIN_MNEMONIC="TODO"
$ export TARGET_CHAIN_ID="TODO" # например "theta-testnet-001" для Cosmos Hub или "uni-5" для Juno 
$ export TARGET_KEY_NAME="TODO" # например "cosmoshub-ibc-relayer" or "juno-ibc-relayer" (должно совпадать с именами ключей `key-name` в конфиге config.toml)
$ hermes keys add --chain quark-1 --mnemonic-file <(echo "$NEUTRON_MNEMONIC") --key-name neutron-ibc-relayer
$ hermes keys add --chain $TARGET_CHAIN_ID --mnemonic-file <(echo "$TARGET_CHAIN_MNEMONIC") --key-name $TARGET_KEY_NAME
```

## 6. Чек баланса

Смотрим, что на ключах ретранслятора, предоставленных на предыдущем шаге, достаточно средств. Вы можете найти инструкции по пополнению счета [здесь](https://github.com/neutron-org/testnets/blob/main/quark/testcases/ICA+ICQ.md#getting-ready).

## 7. Запуск сервиса

Запускаем:

`$ sudo systemctl start neutron-ibc-cosmoshub-relayer.service`

Запускаем при каждой загрузке:

`$ sudo systemctl enable neutron-ibc-cosmoshub-relayer.service`

## 8. Наблюдаем что все работает нормально

Состояние сервиса:

`$ sudo systemctl status neutron-ibc-cosmoshub-relayer.service`

Логи:

`$ journalctl --unit=neutron-ibc-cosmoshub-relayer`

## 9. Создаем соединение между чейнами

Для Cosmos hub:

```bash
$ sudo su ibc-cosmoshub-rly
$ hermes create connection --a-chain quark-1 --b-chain theta-testnet-001
$ exit
```

Для Juno:

```
$ sudo su ibc-juno-rly
$ hermes create connection --a-chain quark-1 --b-chain uni-5
$ exit
```

Видим много текста, но вас интересует только `connection_id` на Neutron в конце вывода:

```
SUCCESS Connection {
    delay_period: 0ns,
    a_side: ConnectionSide { <<< IMPORTANT: a_side
        chain: BaseChainHandle {
            chain_id: ChainId {
                id: "neutron-devnet-1",
                version: 1,
            },
            runtime_sender: Sender { .. },
        },
        client_id: ClientId(
            "07-tendermint-7",
        ),
        connection_id: Some(
            ConnectionId(
                "connection-7", <<< THIS CONNTECTION_ID
            ),
        ),
    },
```

<aside>
💡 Сохраняем где-нибудь только что созданный neutron `connection_id` - он необходим для запуска сценария тестирования.
</aside>
