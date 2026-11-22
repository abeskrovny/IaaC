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
      - "10.1.111.12:80:8080"
    networks:
      - default
      - gitopsnet

networks:
  gitopsnet:
    external: true
```

здесь:
- `10.1.111.12:80`: адрес и порт но стороне хостовой системы;
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

В моем окружении EVE-NG не хватило времени для старта сервера и он вернул ошибку, но последовательная попытка запуска привела к успеху.

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

Весь процесс создания бакапа по шагам (выполняется из корневой папки контейнера `/opt/netbox-docker`):

#### Шаг 1. Бэкап базы данных PostgreSQL
Дамп базы без остановки контейнеров (на "горячую") производится с помощью скрипта `pg_dump.bash`:
```bash
sudo docker compose exec -T postgres pg_dump -U netbox netbox > ~/netbox_db_backup.sql
```

> Использование "домашней папки" позволяет решить вопрос с правами при записе файла `netbox_db_backup.sql`.

Чтобы сжать уже готовый файл дампа отдельной командой с добавлением текущей даты в имя архива, необходимо выполнить:
```bash
gzip -c netbox_db_backup.sql > ~/netbox_db_backup_$(date +%F).sql.gz
```




~?~?~?~~?





#### Шаг 2. Бэкап папки с конфигурацией
На этой стадии необходимо заархивировать локальную папку `./configuration` и сам `docker-compose.yml`:
```bash
tar -czf netbox_config_backup.tar.gz configuration/ docker-compose.yml .env
```

#### Шаг 3. Бэкап именованных томов
Бэкап именованных томов (`media`, `reports`, `scripts`) производится с использованием одноразового контейнера `alpine`, который примонтирует тома и упакует их в архивы в текущую директорию.
```bash
docker run --rm \
  -v netbox-docker_netbox-media-files:/media:ro \
  -v netbox-docker_netbox-reports-files:/reports:ro \
  -v netbox-docker_netbox-scripts-files:/scripts:ro \
  -v $(pwd):/backup \
  alpine tar -czf /backup/netbox_volumes_backup.tar.gz /media /reports /scripts
```

#### Шаг 4. Сохранение файловых архивов
Полученные в итоге файлы: `netbox_db_backup.sql.gz` (или `netbox_db_backup.sql`), `netbox_config_backup.tar.gz` и `netbox_volumes_backup.tar.gz` можно скопировать на другой сервер или в облако.

### Восстановление из бэкапа
Будем рассматривать перенос NetBox на чистый сервер, как наиболее объемную задачу. Для ее выполнения необходимо перенести  архивы в корневой каталог контейнера и выполнить следующие шаги:

#### Шаг 1. Восстановление конфигурации
На этом шаге распаковываем архив с конфигами в новую папку проекта:
```bash
sudo tar -xzf netbox_config_backup.tar.gz
```

#### Шаг 2: Создание структуры томов
Создать структуру контейнеров и томов можно в следующем порядке: запустите сначала только базу данных и создать пустые тома, не запуская сам NetBox.
```bash
docker compose up -d postgres
```

Необходимо подождать 10–15 секунд, пока PostgreSQL инициализирует пустую базу.

#### Шаг 3: Восстановлени базы данных
Залеваем SQL-дамп обратно в контейнер с базой:
```bash
cat netbox_db_backup.sql | docker compose exec -T postgres psql -U netbox -d netbox
```

#### Шаг 4: Распаковываем бэкапы именованных томов
На этой стадии снова используем временный контейнер `alpine`, чтобы развернуть архив обратно в тома Docker:
```bash
docker run --rm \
  -v netbox-docker_netbox-media-files:/media \
  -v netbox-docker_netbox-reports-files:/reports \
  -v netbox-docker_netbox-scripts-files:/scripts \
  -v $(pwd):/backup \
  alpine sh -c "tar -xzf /backup/netbox_volumes_backup.tar.gz -C / && chown -R 999:999 /media /reports /scripts"
```

> Командв `chown -R 999:999` гарантирует, что у пользователя внутри контейнера NetBox будут права на чтение и запись восстановленных файлов.

#### Шаг 5: Запуск сервисов
На данном этапе возможно поднять весь стек NetBox. Но перед запуском необходимо проверить точные имена томов, используя команду `docker volume ls`. Eсли папка проекта называется иначе, префикс перед `_netbox-media-files` будет другим.
```bash
docker compose up -d
```

## Обновление NetBox
Так как я использовал официальный репозиторий (netbox-community/netbox-docker), процесс будет выглядеть следующим образом:

Перейдите в папку с проектом (где лежит `docker-compose.yml`):
```bash
cd /opt/netbox-docker
```

И обновяем исходный код репозитория:
```bash
git pull
```

и скачиваем новые образы контейнеров:
```bash
docker compose pull
```

И, наконец, запускаем контейнер с пересборкой:
```bash
docker compose up -d
```

В случае, если в docker-compose требуется проведение миграции базы данных, необходимо выполнить ее командой:
```bash
docker compose exec netbox python3 manage.py migrate
```

хотя, обычно этот процесс выполняется автоматически.

> Если обновление производится через несколько мажорных версий (например, с v3.7 на v4.x), рекомендуется делать это пошагово через промежуточные релизы, сверяясь с документацией NetBox Upgrading.

## Наполнение базы
Внедрение NetBox как единого источника правды (Source of Truth) часто начинается с хаотичного занесения в базу всего, что есть под рукой: стоек, коммутаторов и подсетей. Однако без заложенного на старте фундамента такая база быстро превращается в неуправляемый каталог с «ничейными» ресурсами. Наиболее правильный вектор проектирования — начать с определения модели владения (Tenancy) и прав собственности (Ownership)

Ниже я разберу, как выстроить базовую иерархию и разграничить зоны ответственности администраторов, опираясь на собственный опыт проектирования гибридного стенда: его нижние уровни (до L2 включительно) построены по архитектуре домашней сети, а верхние — развернуты как виртуальное окружение AI-кластера в EVE-NG.

Начнем с права собственности (Ownership). Связка `OwnerGroup` и `Owner` отвечает за операционное управление, т.е., непосредственно персонал, имеющий право создавать, изменять и удалять эти объекты в самом NetBox. Само прраво собственности появилось в новых версиях NetBox, начиная с v4.2+ для привязки к группам пользователей / RBAC (Role-Based Access Control — управление доступом на основе ролей), которая дополняет классическую модель владения (Tenancy). 

Введение модели Ownership позволяет осуществить однозначную двустороннюю связь между базой данных пользователей (локальной, LDAP и т. д.) и абстрактной моделью инфраструктуры в NetBox, четко закрепляя каждую сущность за ответственной командой администраторов.



- **Разделение ответственности (Делегирование)**: Пока в базе мало объектов, кажется, что всё может заводить один администратор. Но как только подключаются коллеги (ProdWinAdmins), им нужно дать права редактировать только свои серверы и подсети, не задевая ресурсы сетевиков или других отделов.
• Защита от случайного удаления: Настроив права доступа (Permissions) с привязкой к группам владельцев на старте, вы гарантируете, что администраторы Windows-инфраструктуры не смогут случайно удалить транзитный свитч или чужой VLAN.
• Готовность к автоматизации (API-токены): Когда вы начнете подключать Ansible, Terraform или скрипты синхронизации, их сервисные учетные записи тоже нужно будет привязать к конкретным группам владельцев, чтобы автоматика работала строго в пределах своей зоны ответственности.



Организация -> Владение - это логическое разделение на теннанты внутри нетбокс
- Группа собственников (ЮСК)
- Собственник (Технический отдел)

Право собственности - связка с группами в нетбоксе к группам пользователей - кто управляет
- Группа владельцев: OwnerGroup(USK Admins)
- Владелец: Owner (ProdWinAdmins)