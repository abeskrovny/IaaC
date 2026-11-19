# OpenBAO
[OpenBao](https://openbao.org/docs/) является открытым форком HashiCorp Vault (под лицензией MPL 2.0), он полностью совместим с его API, клиентами и коллекциями Ansible (`community.hashi_vault`), но при этом не имеет лицензионных ограничений.

OpenBao (открытый форк HashiCorp Vault) спроектирован как универсальное зашифрованное хранилище секретов, а не только как инструмент для CI/CD. Для хранения статических учетных данных используется встроенный механизм KV (Key-Value) Secrets Engine.

Как работает OpenBao:
- **Движок KV (Key-Value)**. Это основной инструмент для хранения паролей от внешних сервисов. Внутри него создается произвольная структура папок (путей), например `secret/external-accounts/aws` или `secret/services/mail`, и записываются туда пары «ключ-значение», которые могут быть логином, паролем, токенено, URL'ом и т.д.
- **Версионирование (KV v2)**. Вторая версия движка автоматически сохраняет историю изменений. Если кто-то случайно перезаписал пароль, напроимер, от внешнего аккаунта, то можно легко посмотреть старое значение или откатиться к предыдущей версии.
- **Гибкое разграничение прав (Policies)**. С помощью политик доступа можно разрешить конкретному человеку или сервису читать пароли только из определенной ветки (например, только из `secret/marketing/*`), закрыв доступ ко всем остальным.
- **Полный аудит**. OpenBao записывает в лог каждое обращение к секрету: кто, когда и с какого IP-адреса запрашивал конкретный пароль.

Сценарии использования для внешних аккаунтов:
1. **Для приложений и скриптов**. Скрипты автоматизации, бэкапы или CI/CD-пайплайны (в том числе и GitLab Runner, настраиваемый в соответствующей [части](https://github.com/abeskrovny/IaaC/blob/main/GitLab.md)) могут забирать пароли от внешних API и сервисов напрямую из OpenBao по API перед запуском задачи, не храня их в коде.
2. **Для команды (как корпоративный менеджер паролей)**. Сотрудники могут заходить в веб-интерфейс (UI) OpenBao под своими учетными записями (например, через LDAP, OIDC или по токенам) и безопасно брать нужные логины и пароли от общих внешних аккаунтов.

Ограничения, которые стоит учитывать:
- **Нет автозаполнения в браузере**. В отличие от классических менеджеров паролей (Bitwarden, KeePass, 1Password), у OpenBao нет браузерных расширений для автоматической подстановки логинов на сайтах. Пароли придется копировать из веб-интерфейса или забирать через CLI/API.
- **Нет автоматической ротации для внешних сайтов**. Для внутренних баз данных (PostgreSQL, MySQL) OpenBao умеет сам менять пароли по расписанию. Для сторонних веб-сервисов пароли придется обновлять в хранилище вручную после их смены на самом сайте.

## Установка и настройка в Docker-среде
Подготовим директории для конфигурации и данных:
```
cd /opt && sudo mkdir -p /opt/openbao-docker/{config,data}
```

Создаем файл конфигурации `/opt/openbao-docker/config/config.hcl`:
```hcl
ui = true
disable_mlock = true

storage "raft" {
  path    = "/openbao-docker/data"
  node_id = "node1"
}

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = 1 # Для стенда; в продакшене рекомендуется подключить TLS-сертификаты
}

api_addr = "http://127.0.0.1:8200"
cluster_addr = "https://127.0.0.1:8201"
```

и файл сборки `/opt/openbao-docker/docker-compose.yml`:
```yaml
services:
  openbao:
    image: openbao/openbao:latest
    container_name: openbao
    restart: unless-stopped
    ports:
      - "8200:8200"
    volumes:
      - ./config:/openbao/config:ro
      - ./data:/openbao/data
    cap_add:
      - IPC_LOCK
    command: server -config=/openbao-docker/config/config.hcl
```

Запускаем контейнер:
```bash
cd /opt/openbao-docker && sudo docker compose up -d
```

Процесс `OpenBao` внутри контейнера запускается от имени системного пользователя `openbao` (обычно имеет `UID 100` и `GID 1000`), но примонтированная директория на хосте принадлежит пользователю `root`. Необходимо поправить права доступа, чтобы контейнер корректно запустился:
```bash
sudo chown -R 100:1000 /opt/openbao/
```

Далее необходимо произвести первичную инициализацию, используя команду `docker exec -e BAO_ADDR='http://127.0.0.1:8200' openbao bao operator init`. Команда выдаст 5 ключей "распечатывания" (Unseal Keys) и 1 Root Token. Их надо сохранить в надежном месте:
```bash
admin@srvOoBService01:/opt/openbao$ sudo docker exec -e BAO_ADDR='http://127.0.0.1:8200' openbao bao operator init
Unseal Key 1: MawDj/6mhwc/xgSdhi+/xzz6Yo6JAyw9bLBPlsmcJJKk
Unseal Key 2: bWXvaVneC6cNrxzKElbaZlGLiBNRf080BBd2CJvejYSN
Unseal Key 3: /RnJZOLyJvyiLKyn1bM8J9yzKk9PcHeK39NM7SaHeHlH
Unseal Key 4: ZrXkCf+udNIfr3xCzm0+0uF+LApWU8cBIoJrLVf19bNB
Unseal Key 5: EXtflYnZRj4BJN45a8Ow7P4QAcUNJ+oQYbop03+By/Lq

Initial Root Token: s.I41MOy4VdBL1LluI12UkNnzR

Vault initialized with 5 key shares and a key threshold of 3. Please securely
distribute the key shares printed above. When the Vault is re-sealed,
restarted, or stopped, you must supply at least 3 of these keys to unseal it
before it can start servicing requests.

Vault does not store the generated root key. Without at least 3 keys to
reconstruct the root key, Vault will remain permanently sealed!

It is possible to generate new unseal keys, provided you have a quorum
of existing unseal keys shares. See "bao operator rotate-keys" for more
information.
```

> В реальной инфраструктуре публиковать ключи шифрования и токены в открытом доступе категорически запрещено. Однако в данном случае мы работаем в изолированной тестовой среде EVE-NG, поэтому для наглядности примера я оставлю эти данные незамаскированными.

После запуска OpenBao находится в запечатанном (Seal) режиме. Необходимо распечатать (Unseal) хранилище, выполнив команду `docker exec -it -e BAO_ADDR='http://127.0.0.1:8200' openbao bao operator unseal` три раза подряд, каждый раз вводя один из пяти полученных ключей (я буду использовать первые 3):
```bash
admin@srvOoBService01:/opt/openbao$ sudo docker exec -it -e BAO_ADDR='http://127.0.0.1:8200' openbao bao operator unseal      
Unseal Key (will be hidden): MawDj/6mhwc/xgSdhi+/xzz6Yo6JAyw9bLBPlsmcJJKk
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       5
Threshold          3
Unseal Progress    1/3
Unseal Nonce       498e81eb-87e1-46f0-835a-40602954a1f6
Version            2.7.2
Commit Date        2026-10-26T20:32:27Z
Storage Type       raft
HA Enabled         true

admin@srvOoBService01:/opt/openbao$ sudo docker exec -it -e BAO_ADDR='http://127.0.0.1:8200' openbao bao operator unseal
Unseal Key (will be hidden): bWXvaVneC6cNrxzKElbaZlGLiBNRf080BBd2CJvejYSN
Key                Value
---                -----
Seal Type          shamir
Initialized        true
Sealed             true
Total Shares       5
Threshold          3
Unseal Progress    2/3
Unseal Nonce       498e81eb-87e1-46f0-835a-40602954a1f6
Version            2.7.2
Commit Date        2026-10-26T20:32:27Z
Storage Type       raft
HA Enabled         true

admin@srvOoBService01:/opt/openbao$ sudo docker exec -it -e BAO_ADDR='http://127.0.0.1:8200' openbao bao operator unseal
Unseal Key (will be hidden): /RnJZOLyJvyiLKyn1bM8J9yzKk9PcHeK39NM7SaHeHlH
Key                     Value
---                     -----
Seal Type               shamir
Initialized             true
Sealed                  false
Total Shares            5
Threshold               3
Version                 2.7.2
Commit Date             2026-10-26T20:32:27Z
Storage Type            raft
Cluster Name            vault-cluster-0b3b220a
Cluster ID              cdb00cd4-8ad4-f5de-0365-5b869c5ef036
HA Enabled              true
HA Cluster              n/a
HA Mode                 standby
Raft Committed Index    26
Raft Applied Index      26
```

После этого веб-интерфейс станет доступен по адресу `http://<IP_сервера>:8200/ui`:

![Стартовый экран OpenBAO](Images/OpenBAO01.png)

Войти в систему можно через WEB-инрерфейс, с использованием токена указав `Initial Root Token` в поле Token (он начинается с `hvs.` или `s.` и был получен в терминале во время первоначальной инициализации `bao operator init`), либо можно использовать CLI контейнера, о чем речь пойдет ниже.

Если вдруг на указанном порту возникают проблемы с доступом, необходимо проверить наличие строки `ui = true` в `config/config.hcl` и, в случае необходимости, перезапустить контейнер.

## Стратегия резервного копирования
Стратегия резервного копирования OpenBao в Docker напрямую зависит от того, какой бэкенд хранения (`storage`) прописан в `config/config.hcl`.

Для небольших инсталляций, Docker-окружений и кластеров без сложной внешней обвязки лучше всего выбрать Integrated Storage (Raft) — этот вариант уже используется в текущей конфигурации. Он не требует сторонних сервисов, поддерживает отказоустойчивость (HA) «из коробки» и позволяет делать горячие бэкапы одной командой.

Следует отметить, что OpenBao (наследуя архитектуру HashiCorp Vault) поддерживает несколько типов хранилищ, которые делятся на встроенные и внешние. Все данные, записываемые в любой из этих бэкендов, предварительно шифруются самим OpenBao, поэтому хранилище видит только зашифрованные бинарные блоки.

_Таблица. Сравнение основных бэкендов хранения OpenBao_

| **Бэкенд** | **Тип и архитектура** | **Отказоустойчивость (HA)** | **Сложность обслуживания** | **Резервное копирование** |
|:---:|---|---|---|---|
| **Raft (Integrated Storage)** | Встроенное распределенное хранилище (на базе BoltDB) | **Да** (кластер из 3 или 5 нод) | **Низкая**: всё внутри одного бинарника/контейнера | Встроенная команда `raft snapshot save` без остановки |
| **PostgreSQL** | Внешняя реляционная СУБД | **Да**, если база в HA-кластере, например, Patroni | **Средняя**: нужно отдельно администрировать и обновлять БД | Стандартный `pg_dump` или PITR-бэкапы самой СУБД |
| **File (Filesystem)** | Локальные файлы на диске контейнера/хоста | **Нет**: строго одна нода | **Минимальная**: просто папка на диске | Только холодный бэкап (остановка контейнера и архивация) |
| **Consul** | Внешнее распределенное KV-хранилище | **Да** | **Высокая**: требует развертывания отдельного кластера Consul | Снапшоты Consul (`consul snapshot save`) |
| **In-Memory** | Хранение в оперативной памяти | **Нет** | **Нулевая**: данные пропадают при рестарте | Невозможно (только для автотестов) |


## Использование OpenBAO в качестве корпоративного менеджера паролей

## Использование OpenBAO в качестве источника секретов для приложений и скриптов


!!!Интеграция со всеми компонентами
