# Конфигурация Neutron Query Relayer

## Переквизиты

- Go версии 1.18.7 или старше.

## Установка релея

> Примечание: если вы не хотите билдить ретранслятор, можно скачать бинарник со страницы [releases](https://github.com/neutron-org/neutron-query-relayer/releases/tag/v0.1.1).

```
$ git clone -b v0.1.1 https://github.com/neutron-org/neutron-query-relayer.git
$ cd neutron-query-relayer
$ make install
```

Команда установит бинарник релея в каталог `$GOPATH/bin`. Проверить версию, можно выполнив:

```
$ neutron_query_relayer version
Version: 0.1.1
Commit: 934ff75595858461bbc5eb300cff03c43f11f4ab
```

## Ставим neutron

Смотрим [инструкции](https://github.com/neutron-org/testnets/blob/main/quark/README.md#node-installation) или загружаем бинрник со страницы [релизы](https://github.com/neutron-org/neutron/releases/tag/v0.1.1).


## Конфигурация

ICQ-ретранслятор должен иметь адрес на чейне Neutron, и на этом адресе должно быть несколько токенов testnet `$ntrn`. Смотрим [инструкцию](https://github.com/neutron-org/testnets/blob/main/quark/testcases/ICA+ICQ.md#generate-the-relayer-address-on-neutron-and-get-testnet-ntrn-tokens) о том, как получить токены testnet `$ntrn`.

Выполняем `neutrond keys add my_wallet --recover --home /path/to/home --keyring-backend test` и вставляем мнемонику.

Загружаем файл `.env` на сервер:

```
wget https://raw.githubusercontent.com/systemd-run/quark-tasks/main/icq-relayer/.env
```

Открываем этот файл в текстовом редакторе и заполняем следующие переменные (подробное описание параметров см. в [документации](https://docs.neutron.org/relaying/icq-relayer#configuration)):

- RELAYER_NEUTRON_CHAIN_RPC_ADDR (например, `http://23.109.158.236:26657` — примечание: не используйте `tcp://` (используйте `http://`) и убирайте слеши в конце)
- RELAYER_NEUTRON_CHAIN_REST_ADDR (например, `http://23.109.158.236:1317` — примечание: не используйте `tcp://` (используйте `http://`) и убирайте слеши в конце)
- RELAYER_NEUTRON_CHAIN_HOME_DIR (например, `/path/to/home` — путь к домашнему каталогу, используйте тот же путь, который был передан neutrond с параметром `--home` option when importing key in previous step)
- RELAYER_NEUTRON_CHAIN_CONNECTION_ID
- RELAYER_TARGET_CHAIN_RPC_ADDR (например, `http://164.90.146.43:26657` — примечание: не используйте `tcp://` (используйте `http://`) и убирайте слеши в конце)
- RELAYER_REGISTRY_ADDRESSES (адрес контракта)
- RELAYER_STORAGE_PATH (просто путь - новый каталог будет создан автоматически)

## Запуск ICQ релея

```
export $(grep -v '^#' .env | xargs) && neutron_query_relayer start
```
