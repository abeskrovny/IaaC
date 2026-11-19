## OpenBAO

[OpenBao](https://openbao.org/docs/) является открытым форком HashiCorp Vault (под лицензией MPL 2.0), он полностью совместим с его API, клиентами и коллекциями Ansible (`community.hashi_vault`), но при этом не имеет лицензионных ограничений.

OpenBao (открытый форк HashiCorp Vault) спроектирован как универсальное зашифрованное хранилище секретов, а не только как инструмент для CI/CD. Для хранения статических учетных данных используется встроенный механизм KV (Key-Value) Secrets Engine.


Как это работает в OpenBao

• Движок KV (Key-Value). Это основной инструмент для хранения паролей от внешних сервисов. Внутри него вы создаете произвольную структуру папок (путей), например secret/external-accounts/aws или secret/services/mail, и записываете туда пары «ключ-значение» (логин, пароль, токен, URL).
• Версионирование (KV v2). Вторая версия движка автоматически сохраняет историю изменений. Если вы случайно перезаписали пароль от внешнего аккаунта, можно легко посмотреть старое значение или откатиться к предыдущей версии.
• Гибкое разграничение прав (Policies). С помощью политик доступа вы можете разрешить конкретному человеку или сервису читать пароли только из определенной ветки (например, только из secret/marketing/*), закрыв доступ ко всем остальным.
• Полный аудит. OpenBao записывает в лог каждое обращение к секрету: кто, когда и с какого IP-адреса запрашивал конкретный пароль.

Сценарии использования для внешних аккаунтов

1. Для приложений и скриптов. Скрипты автоматизации, бэкапы или CI/CD-пайплайны (в том числе ваш GitLab Runner) могут забирать пароли от внешних API и сервисов напрямую из OpenBao по API перед запуском задачи, не храня их в коде.
2. Для команды (как корпоративный менеджер паролей). Сотрудники могут заходить в веб-интерфейс (UI) OpenBao под своими учетными записями (например, через LDAP, OIDC или по токенам) и безопасно брать нужные логины и пароли от общих внешних аккаунтов.

Ограничения, которые стоит учитывать

• Нет автозаполнения в браузере. В отличие от классических менеджеров паролей (Bitwarden, KeePass, 1Password), у OpenBao нет браузерных расширений для автоматической подстановки логинов на сайтах. Пароли придется копировать из веб-интерфейса или забирать через CLI/API.
• Нет автоматической ротации для внешних сайтов. Для внутренних баз данных (PostgreSQL, MySQL) OpenBao умеет сам менять пароли по расписанию. Для сторонних веб-сервисов пароли придется обновлять в хранилище вручную после их смены на самом сайте.





Создайте директории для конфигурации и данных:
```
cd /opt && sudo mkdir -p /opt/openbao-docker/{config,data}
```

Создайте файл конфигурации `/opt/openbao-docker/config/config.hcl`:
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

Создайте `/opt/openbao-docker/docker-compose.yml`:
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

Запустите контейнер и выполните первичную инициализацию:
```bash
cd /opt/openbao-docker && sudo docker compose up -d
```

Процесс `OpenBao` внутри контейнера запускается от имени системного пользователя `openbao` (обычно имеет `UID 100` и `GID 1000`), но примонтированная директория на хосте принадлежит пользователю `root`. Необходимо поправить права доступа, чтобы контейнер корректно запустился:
```bash
sudo chown -R 100:1000 /opt/openbao/
```

Далее необходимо произвести первичную инициализацию, используя команду `docker exec -e BAO_ADDR='http://127.0.0.1:8200' openbao bao operator init`. Команда выдаст 5 ключей распечатывания (Unseal Keys) и 1 Root Token. Обязательно сохраните их в надежном месте:
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

После запуска OpenBao находится в запечатанном (Seal) состоянии. Необходимо распечатать (Unseal) хранилище, выполнив команду `docker exec -it -e BAO_ADDR='http://127.0.0.1:8200' openbao bao operator unseal` три раза подряд, каждый раз вводя один из пяти полученных ключей (я буду использовать первые 3):
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
window@srvOoBService01:/opt/openbao$ sudo docker exec -it -e BAO_ADDR='http://127.0.0.1:8200' openbao bao operator unseal
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
window@srvOoBService01:/opt/openbao$ sudo docker exec -it -e BAO_ADDR='http://127.0.0.1:8200' openbao bao operator unseal
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