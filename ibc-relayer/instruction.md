# Инструкция по установке quark-1 testnet

> **Примечание: мы предполагаем, что вы уже инициализировали тестовый смарт-контракт. Вам понадобится адрес контракта, чтобы поместить его в `config.toml` для того, чтобы ретранслятор работал только с пакетами, приходящими от этого контракта.**

Это руководство предназначено для подключения к сети cosmoshub testnet (theta-testnet-001). Чтобы подключить hermes с помощью juno, проделайте те же шаги (начиная с шага 2), но измените имя пользователя и имя сервиса, а также откомментируйте конфиг сети для juno в файле `config.toml`. В этом руководстве hermes устанавливается как демон.

## 1. Установка Hermes v1.0.0

```
curl -L "https://github.com/informalsystems/ibc-rs/releases/download/v1.0.0/hermes-v1.0.0-${PLATFORM}-unknown-linux-gnu.tar.gz" > hermes.tar.gz && \
    tar -C ./ -vxzf hermes.tar.gz && \
    rm -f hermes.tar.gz  && \
    sudo mv ./hermes /usr/local/bin/ && \
    sudo chgrp root /usr/local/bin/hermes && \
    sudo chown root /usr/local/bin/hermes
```

Проверьте, что он работает и версия в порядке (Должна быть `hermes 1.0.0+ed4dd8c`)

`$ hermes --version`

## 2. Добавление пользователя

```
$ sudo useradd -m ibc-cosmoshub-rly
$ sudo su ibc-cosmoshub-rly
$ cd ~/
$ mkdir ~/.hermes
```

## 3. Созание сервиса

`vim /etc/systemd/system/neutron-ibc-cosmoshub-relayer.service`:

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

Скопируйте пример [конфига](https://github.com/neutron-org/testnets/blob/main/quark/ibc-relayer/config.toml) в `~/.hermes/config.toml` **и заполните недостающие параметры.**

Не забудьте заполнить недостающие параметры (отмечены комментариями TODO).

ПРИМЕЧАНИЕ: `websocket_addr` должен начинаться с протокола ws|wss

Проверьте корректность конфигурации:

`$ hermes health-check`

## 5. Добавление ключей к ретранслятору

Не забудьте сгенерировать свои мнемоники для учетных записей и заполнить их в командах bash, приведенных ниже:

```
$ sudo su ibc-cosmoshub-rly
$ export NEUTRON_MNEMONIC="TODO"
$ export TARGET_CHAIN_MNEMONIC="TODO"
$ export TARGET_CHAIN_ID="TODO" # e.g., "theta-testnet-001" for Cosmos Hub testnet or "uni-5" for Juno testnet 
$ export TARGET_KEY_NAME="TODO" # e.g. "cosmoshub-ibc-relayer" or "juno-ibc-relayer" (matches key-name in config.toml)
$ hermes keys add --chain quark-1 --mnemonic-file <(echo "$NEUTRON_MNEMONIC") --key-name neutron-ibc-relayer
$ hermes keys add --chain $TARGET_CHAIN_ID --mnemonic-file <(echo "$TARGET_CHAIN_MNEMONIC") --key-name $TARGET_KEY_NAME
```

## 6. Проверка средств

Убедитесь, что на ключах ретранслятора, предоставленных на предыдущем шаге, достаточно средств. Вы можете найти инструкции по пополнению счета [здесь] (https://github.com/neutron-org/testnets/blob/main/quark/testcases/ICA+ICQ.md#getting-ready).

## 7. Запуск сервиса

Start it:

`$ sudo systemctl start neutron-ibc-cosmoshub-relayer.service`

Запускайте его при каждой загрузке:

`$ sudo systemctl enable neutron-ibc-cosmoshub-relayer.service`

## 8. Убедитесь, что все работает нормально

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

Вы увидите много текста, но вас интересует только `connection_id` на Neutron в конце вывода:

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
💡 Сохраните где-нибудь только что созданный neutron `connection_id` - он необходим для запуска сценария тестирования.
</aside>
