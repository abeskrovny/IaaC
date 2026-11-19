# Установка и настройка NetBox

NetBox — это открытая платформа для управления сетевой инфраструктурой и центрами обработки данных, объединяющая в себе функции:
- **IPAM (IP Address Management)**: учет IP-адресов и подсетей (на самом деле значительно шире, включая информацию о L2 сегментации, ASN и так далее);
- **DCIM (Data Center Infrastructure Management)**: инвентаризация стоек, серверов, сетевого оборудования и кабельных соединений.

В отличие от систем мониторинга, которые лишь фиксируют текущее состояние сети, NetBox хранит эталонную модель того, как инфраструктура должна быть спроектирована и настроена. Благодаря развитому REST API и GraphQL API, система легко интегрируется в пайплайны автоматизации, выступая связующим звеном между физическим уровнем оборудования и логической конфигурацией сервисов.

В нашей архитектуре NetBox используется в концепции распределенного источника истины (Splitted Source of Truth), и я зафиксировал этот подход для учета в будущих задачах. Это означает, что система отвечает за строго определенный домен данных — например, за физическую топологию, распределение IP-пространства, VLAN и базовые параметры устройств, — в то время как другие аспекты конфигурации (такие как секреты, прикладные настройки сервисов или состояние виртуальных сред) делегированы профильным инструментам вроде OpenBao или Git-репозиториев. Такой подход позволяет избежать создания громоздкой монолитной базы, сохраняя высокую скорость работы автоматизации и четкое разделение зон ответственности между компонентами инфраструктуры.

## Установка и первоначальная настройка NetBox
Установка NetBox будем производить в docker-окружении. Использование официального репозитория `netbox-docker` — самый надежный способ быстро поднять актуальную версию NetBox со всеми зависимостями (PostgreSQL, Redis, worker).

В качестве корня для Docker-инфраструктуры будем использовать каталог `/opt`. Директория `/opt` (optional — дополнительный) — это стандартный каталог в файловой системе Linux, предназначенный для установки стороннего программного обеспечения и автономных пакетов, не входящих в базовую поставку операционной системы. 

Такой подход реализует изоляцию прикладных сервисов от системных файлов и упрощает управление дисковым пространством.

Начнем с клонирования репозитория:
```bash
admin@srvOoBService01:/opt$ cd /opt/ && sudo git clone -b release https://github.com/netbox-community/netbox-docker.git
cd netbox-docker
Cloning into 'netbox-docker'...
remote: Enumerating objects: 5991, done.
remote: Counting objects: 100% (24/24), done.
remote: Compressing objects: 100% (17/17), done.
Receiving objects: 100% (5991/5991), 1.63 MiB | 6.00 KiB/s, done.
remote: Total 5991 (delta 11), reused 9 (delta 6), pack-reused 5967 (from 2)
Resolving deltas: 100% (3402/3402), done.
```

Произведем настройку переопределений. Для этого создадим файл `docker-compose.override.yml`, чтобы пробросить порты наружу контейнера:
```yaml
services:
  netbox:
    ports:
      - "8000:8080"
```

здесь:
- `8000`: порт на стороне хостовой системы;
- `8080`: порт на стороне контейнера.

Забираем образы:
```bash
sudo docker compose pull
```

Однако, в наших пинатах можно столкнуться с особенностями блокировок. Ошибка `connection timed out` при обращении к `docker.io` (адрес `172.64.144.78:443`) означает, что соединение с Docker Hub блокируется или обрывается (в РФ доступ к серверам Docker Hub ограничен, а IP `172.64.144.78` принадлежит `Cloudflare`).

Чтобы обойти это ограничение и скачать образы для NetBox, необходимо настроить зеркала реестров (registry mirrors) в конфигурации демона Docker.

Изменим список публичных зеркал в файле `/etc/docker/daemon.json`:
```bash
sudo tee /etc/docker/daemon.json <<EOF
{
  "registry-mirrors": [
    "https://timeweb.cloud",
    "https://mirror.gcr.io"
  ]
}
EOF
```

Требуется перезапустить службу Docker, чтобы применить новую конфигурацию:
```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

После чего (в моем случае), образы были "залиты":
```bash
admin@srvOoBService01:/opt/netbox-docker$ sudo docker compose pull
[sudo: authenticate] Password:         
[+] Pulling 5/5
 ✔ redis-cache Skipped - Image is already being pulled by redis            0.0s 
 ✔ netbox-worker Skipped - Image is already being pulled by netbox         0.0s 
 ✔ postgres Pulled                                                         6.7s 
 ✔ redis Pulled                                                            7.0s 
 ✔ netbox Pulled 
```

Запустим контейнеры:
```bash
sudo docker compose up -d
```

Однако, в моем окружении EVE-NG не хватило времени для старта сервера и он вернул ошибку, но последовательная попытка запуска привела к успеху.

Создадим суперпользователя для входа в веб-интерфейс и генерации API-токенов:
```bash
admin@srvOoBService01:/opt/netbox-docker$ sudo docker compose exec netbox /opt/netbox/netbox/manage.py createsuperuser
🧬 loaded config '/etc/netbox/config/configuration.py'
🧬 loaded config '/etc/netbox/config/extra.py'
🧬 loaded config '/etc/netbox/config/logging.py'
🧬 loaded config '/etc/netbox/config/plugins.py'
/opt/netbox/netbox/netbox/settings.py:282: FutureWarning: LOGIN_REQUIRED is deprecated and will be removed in NetBox v5.0. This parameter can be removed from your configuration file.
  warnings.warn(
Username: admin
Email address: admin@local.org
Password: 
Password (again): 
This password is too short. It must contain at least 12 characters.
Password must have at least one numeral.
Bypass password validation and create user anyway? [y/N]: y
Superuser created successfully.
```

> Пользователи `admin` не пересекаются между областями видимости и, вообще говоря, принадлежат разным системам: хоста и контейнера.

Получим токен (только чтение): `Bearer nbt_x75oyug8vQzJ.BGgKW10Xl2241kqU0YTeTohZW5IIcZWdCW7bGivE`, который можно проверить обращением к серверу:
```bash
slipknot@MegaFire ~ % curl -X GET \
-H "Authorization: Bearer nbt_x75oyug8vQzJ.BGgKW10Xl2241kqU0YTeTohZW5IIcZWdCW7bGivE" \
-H "Content-Type: application/json" \
-H "Accept: application/json; indent=4" \
http://10.1.10.88:8000/api/status/
{
    "django-version": "6.1.3",
    "hostname": "2928a1d61acf",
    "installed_apps": {
        "django_filters": "26.2",
        "django_prometheus": "2.4.0",
        "django_rq": "4.2.0",
        "django_tables2": "3.0.1",
        "drf_spectacular": "0.30.0",
        "drf_spectacular_sidecar": "2026.10.1",
        "mptt": "0.18.0",
        "rest_framework": "3.18.3",
        "social_django": "6.1.0",
        "taggit": "7.0.0",
        "timezone_field": "7.2.2"
    },
    "netbox-version": "4.7.5",
    "netbox-full-version": "4.7.5-Docker-5.1.2",
    "plugins": {},
    "python-version": "3.14.4",
    "rq-workers-running": 1
}%           
```

Чтобы контейнеры запускались автоматически при включении или перезагрузке сервера, нужно добавить параметр `restart: unless-stopped` (или `always`) в конфигурацию каждого сервиса в файле `docker-compose.yml` и убедиться, что сама служба `Docker` стоит в автозагрузке. 

Какую политику выбрать:
- `restart: unless-stopped` (рекомендуется). Контейнер будет автоматически запускаться при старте системы и перезапускаться при сбоях, но не поднимется сам, если вы остановили его вручную через `docker stop` или `docker compose stop`.
- `restart: always`. Контейнер будет запускаться при старте системы и перезапускаться всегда, даже если до перезагрузки сервера вы остановили его вручную.
- `restart: on-failure`. Перезапускает контейнер только при аварийном завершении (код выхода не равен 0) и не подходит для простого автозапуска после плановой перезагрузки.

Например:
```yaml
services:
  web:
    image: nginx:latest
    restart: on-failure

  db:
    image: postgres:15
    restart: unless-stopped
```

После сохранения файла необходимо пересобрать контейнер:
```bash
sudo docker compose up -d
```

Контейнеры стартуют только после запуска демона `Docker`, поэтому необходимо убедиться, что служба добавлена в автозагрузку ОС (в современных дистрибутивах Linux это обычно включено по умолчанию):
```bash
sudo systemctl is-enabled docker
sudo systemctl is-enabled containerd
```

и включить в автозагрузку, если это потребуется:
```bash
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
```

## Стратегия резервного копирования
Перед тем, как переходить к "наполнению" базы, необходимо сразу рассмотреть стратегию резервного копирования.

Если посмотреть в файл `docker-compose.yml`, можно увидеть опции `volumes` у каждого образа, которые пробрасывают файловую систему хоста внутрь контейнера. Например, для сервиса `netbox`:
```yaml
...
    volumes:
      - ./configuration:/etc/netbox/config:z,ro
      - netbox-media-files:/opt/netbox/netbox/media:rw
      - netbox-reports-files:/opt/netbox/netbox/reports:rw
      - netbox-scripts-files:/opt/netbox/netbox/scripts:rw
...
```

В конфигурации NetBox Docker эти четыре строки делятся на два разных типа монтирования: обычная локальная папка (`bind mount`) и именованные тома Docker (`named volumes`):
1. **Локальная папка (`Bind Mount`)**:
    - `./configuration:/etc/netbox/config:z,ro`: Точка в начале пути (`./`) означает текущую директорию, в которой лежит файл `docker-compose.yml`. Если проект лежит в `/opt/netbox-docker/`, то реальная папка с конфигами на сервере находится по пути `/opt/netbox-docker/configuration/`. Именно сюда нужно класть файлы настроек вроде `configuration.py` или `plugins.py`.

2. **Именованные тома (`Named Volumes`)**: Строки без точки или слэша в начале (`netbox-media-files`, `netbox-reports-files`, `netbox-scripts-files`) — это внутренние тома, которыми управляет сам Docker. По умолчанию в Linux они хранятся в системной директории Docker по пути `/var/lib/docker/volumes/`.
    - `netbox-media-files:/opt/netbox/netbox/media:rw`: Файлы лежат в `/var/lib/docker/volumes/netbox-docker_netbox-media-files/_data`. Сюда сохраняются загруженные через веб-интерфейс картинки и вложения.
    - `netbox-reports-files:/opt/netbox/netbox/reports:rw`: Здесь хранятся пользовательские отчеты.
    - `netbox-scripts-files:/opt/netbox/netbox/scripts:rw`: Сюда помещаются кастомные скрипты автоматизации.

Проект содержит не один сервис, поэтому таких папок может быть много.

Однако, есть достаточно простой метод быстро узнать точный путь или перенести файлы:
- **Узнать точный путь на диске**. Выполнить в терминале команду `docker volume inspect <имя_тома>` (список всех томов можно посмотреть через `docker volume ls`). В поле `Mountpoint` будет указан точный путь на хосте.
- **Загрузить свои скрипты или отчеты**. Можете загружать их прямо через веб-интерфейс NetBox (в разделах Customization -> Scripts / Reports), и Docker сам положит их в нужный том.
- **Сделать папки обычными (как `configuration`)**. Если удобнее редактировать скрипты напрямую в папке рядом с `docker-compose.yml`, необходимо просто заменить имена томов на относительные пути с точкой:
```yaml
volumes:
  - ./configuration:/etc/netbox/config:z,ro
  - ./media:/opt/netbox/netbox/media:rw
  - ./reports:/opt/netbox/netbox/reports:rw
  - ./scripts:/opt/netbox/netbox/scripts:rw
```

В этом случае все папки создадутся непосредственно рядом с файлом `docker-compose.yml` (не следует забывать при этом удалить их упоминание из секции `volumes:` в `docker-compose.yml`).

### Создание бэкапа
Для полноценного бэкапа NetBox нужно сохранить базу данных PostgreSQL, файлы конфигурации и именованные тома (`media`, `reports` и `scripts`).

Лучший способ работать с именованными томами Docker — использовать временный контейнер-помощник, который упакует их в архив, не требуя прямого доступа в системную папку `/var/lib/docker/`.





Выполните эти команды из папки, где лежит ваш docker-compose.yml (предполагается, что папка проекта называется netbox-docker, поэтому имена томов имеют префикс netbox-docker_):
• Бэкап базы данных PostgreSQL. Снимите дамп базы без остановки контейнеров с помощью pg_dump.bash
```bash
docker compose exec -T postgres pg_dump -U netbox netbox > netbox_db_backup.sql
```

• Бэкап папки с конфигурацией. Просто заархивируйте локальную папку ./configuration и сам docker-compose.yml.```bash
tar -czf netbox_config_backup.tar.gz configuration/ docker-compose.yml .env
```

• Бэкап именованных томов (media, reports, scripts). Запустите одноразовый контейнер alpine, который примонтирует ваши тома и упакует их в архивы в текущую директорию.
```bash
docker run --rm \
  -v netbox-docker_netbox-media-files:/media:ro \
  -v netbox-docker_netbox-reports-files:/reports:ro \
  -v netbox-docker_netbox-scripts-files:/scripts:ro \
  -v $(pwd):/backup \
  alpine tar -czf /backup/netbox_volumes_backup.tar.gz /media /reports /scripts
```

В итоге у вас получится три файла (netbox_db_backup.sql, netbox_config_backup.tar.gz и netbox_volumes_backup.tar.gz), которые можно скопировать на другой сервер или в облако.

2. Разворачивание (восстановление) из бэкапа

Если вы переносите NetBox на чистый сервер, перенесите туда ваши архивы и выполните следующие шаги:
• Шаг 1: Восстановите конфигурацию. Распакуйте архив с конфигами в новую папку проекта.
```bash
tar -xzf netbox_config_backup.tar.gz
```

• Шаг 2: Создайте структуру контейнеров и томов. Запустите только базу данных и создайте пустые тома, но пока не запускайте сам NetBox.
```bash
docker compose up -d postgres
```

(Подождите 10–15 секунд, пока PostgreSQL инициализирует пустую базу).

• Шаг 3: Восстановите базу данных. Залейте ваш SQL-дамп обратно в контейнер с базой.
```bash
cat netbox_db_backup.sql | docker compose exec -T postgres psql -U netbox -d netbox
```

• Шаг 4: Распакуйте файлы в именованные тома. Снова используйте временный контейнер alpine, чтобы развернуть архив обратно в тома Docker.
```bash
docker run --rm \
  -v netbox-docker_netbox-media-files:/media \
  -v netbox-docker_netbox-reports-files:/reports \
  -v netbox-docker_netbox-scripts-files:/scripts \
  -v $(pwd):/backup \
  alpine sh -c "tar -xzf /backup/netbox_volumes_backup.tar.gz -C / && chown -R 999:999 /media /reports /scripts"
```

(Примечание: chown -R 999:999 гарантирует, что у пользователя внутри контейнера NetBox будут права на чтение и запись восстановленных файлов).

• Шаг 5: Запустите все сервисы. Теперь поднимите весь стек NetBox.
```bash
docker compose up -d
```

(Совет: перед запуском команд проверьте точные имена ваших томов через docker volume ls — если папка вашего проекта называется иначе, префикс перед _netbox-media-files будет другим).




локальные папки и статические тома (media, reports, scripts) можно просто заархивировать на диске. Однако так делать категорически нельзя с томом базы данных PostgreSQL, если контейнеры при этом запущены.

Том с базой данных (postgres-data) нельзя копировать напрямую во время работы контейнера.
• Риск повреждения данных. PostgreSQL постоянно держит часть данных в оперативной памяти и пишет изменения на диск в реальном времени. Если вы заархивируете папку базы на ходу, файлы попадут в архив в несогласованном состоянии, и база при разворачивании просто не запустится.
• Проблемы с версиями. Прямая копия файлов базы заработает только на точно такой же версии PostgreSQL и архитектуре процессора, тогда как текстовый дамп (pg_dump) можно развернуть почти где угодно.

Если вы хотите делать резервную копию простым копированием файлов без сложных команд с временными контейнерами, придерживайтесь такой схемы:
1. Остановите контейнеры командой `docker compose down` (если допустим небольшой простой на время копирования).
2. Заархивируйте все нужные папки на диске, включая `/var/lib/docker/volumes/...` и папку проекта. Для Linux лучше использовать `tar -czpf` (флаг p сохраняет права доступа).
3. Запустите контейнеры обратно через `docker compose up -d`.


Обновление NetBox и GitLab, развернутых в Docker (обычно через docker-compose), требует строгого порядка и предварительного бэкапа.


Общие правила перед началом
1. Сделайте резервную копию (дамп базы данных PostgreSQL, файлы конфигурации и пользовательские данные / тома Docker).
2. Проверьте совместимость и релизные заметки (Release Notes) для версий, на которые вы планируете переходить.


1. Как обновлять NetBox (в Docker)

Официальный репозиторий развертывания — netbox-community/netbox-docker. Процесс выглядит так:
1. Перейдите в папку с проектом (где лежит ваш docker-compose.yml):bash
cd /path/to/netbox-docker
Используйте код с осторожностью.
2. Обновите исходный код репозитория (если вы клонировали репозиторий с GitHub):bash
git pull
Используйте код с осторожностью.
3. Скачайте новые образы контейнеров:bash
docker compose pull
Используйте код с осторожностью.
4. Запустите обновление / пересоздайте контейнеры:bash
docker compose up -d
Используйте код с осторожностью.
5. Примените миграции базы данных и соберите статику (если это требуется в вашем варианте docker-compose, обычно выполняется автоматически или через команду manage.py migrate):bash
docker compose exec netbox python3 manage.py migrate
Используйте код с осторожностью.
Совет: Если вы прыгаете через несколько мажорных версий (например, с v3.7 на v4.x), рекомендуется делать это пошагово через промежуточные релизы, сверяясь с документацией NetBox Upgrading.

## Наполнение базы
В данном разделе я опираюсь, в основном, на свой опыт в преломлении умозрительной локации, состоящей на нижних уровнях из архитектуры домашней сети (до L2 включительно) и на остальных - виртуальном окружении AI-кластера в EVE-NG.






Организация -> Владение - это логическое разделение на теннанты внутри нетбокс
- Группа собственников (ЮСК)
- Собственник (Технический отдел)

Право собственности - связка с группами в нетбоксе к группам пользователей - кто управляет
- Группа владельцев: OwnerGroup(USK Admins)
- Владелец: Owner (ProdWinAdmins)