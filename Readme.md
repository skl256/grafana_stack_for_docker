# Стек сервисов Grafana, Prometheus, Pushgateway, Loki и Promtail для Docker (swarm, compose)

Логи собираются со всех контейнеров в кластере swarm без необходимости установки дополнительного ПО на ноды в кластере.
Метрики собираются с exporter'ов, которые устанавливаются отдельно (при необходимости).
Можно использовать также и на одной ноде (одной VM, одной машине) c docker compose.

## Быстрый старт

* Для docker swarm (должен быть инициализирован swarm: `docker swarm init`)

```
git clone https://github.com/skl256/grafana_stack_for_docker.git && \
cd grafana_stack_for_docker && \
sudo mkdir -p /mnt/common_volume/swarm/grafana/config && \
sudo mkdir -p /mnt/common_volume/grafana/{grafana-config,grafana-data,prometheus-data,loki-data,promtail-data} && \
sudo chown -R $(id -u):$(id -g) {/mnt/common_volume/swarm/grafana/config,/mnt/common_volume/grafana} && \
touch /mnt/common_volume/grafana/grafana-config/grafana.ini && \
cp config/* /mnt/common_volume/swarm/grafana/config/ && \
docker stack deploy -c grafana.yaml grafana
```

* Для docker compose

```
git clone https://github.com/skl256/grafana_stack_for_docker.git && \
cd grafana_stack_for_docker && \
sudo mkdir -p /mnt/common_volume/swarm/grafana/config && \
sudo mkdir -p /mnt/common_volume/grafana/{grafana-config,grafana-data,prometheus-data,loki-data,promtail-data} && \
sudo chown -R $(id -u):$(id -g) {/mnt/common_volume/swarm/grafana/config,/mnt/common_volume/grafana} && \
touch /mnt/common_volume/grafana/grafana-config/grafana.ini && \
cp config/* /mnt/common_volume/swarm/grafana/config/ && \
mv grafana.yaml docker-compose.yaml && \
docker compose up -d
```

* Перейти в браузере по адресу http:// **IP адрес сервера, на котором запущен стек** :3000 (логин `admin` пароль `admin`)
* Расширенные labels не будут работать до внесения изменений в `/etc/docker/daemon.json`

## Особенности конфигурации из примера

* Конфигурация для использования в docker swarm. Если необходимо использовать просто с docker compose: 
  * удалить или скорректировать параметры `deploy:` в `grafana.yaml`, переименовать `mv grafana.yaml docker-compose.yaml`
  * не использовать docker secrets, т.к. docker compose в настоящий момент не поддерживает secrets
* В конфигурации используются bind volumes. Перед запуском необходимо: 
  * В `grafana.yaml` скорректировать используемые в примере пути: `/mnt/common_volume/swarm/grafana/config` (файлы конфигураций) и `/mnt/common_volume/grafana` (файлы данных) на свои. 
    * Для кластера (несколько нод): подразумевается, что `/mnt/common_volume` это общий том для нод в кластере. Если создать общий том для всех нод нет возможности, необходимо запускать все сервисы, кроме promtail, только на одной ноде, а для promtail создать bind volume на каждой ноде (это корректно, что для каждого экземпляра promtail будет свой volume).
    * Для запуска всего стека на одной оде (одной VM, одной машине) можно использовать любой удобный volume.
    * В любом случае необходимо **обязательно** создать пустые папки и файлы конфигурации (для последних - даже если они пустые, например `grafana.ini`). Команды для создания указаны в `grafana.yaml` в комментариях в секции `volume`.
* В конфигурации `prometheus.yaml` указаны примеры сбора метрик с различных exporter'ов, для сбора метрик с них, последние необходимо установить и настроить (в противном случае, с отсутствующих не будут собираться метрики).

## Конфигурация отдельных сервисов

### Grafana

Документация: <https://grafana.com/docs/grafana/latest/setup-grafana/configure-grafana/>

* Образ `grafana/grafana-oss:10.2.2`
* Порт `3000`
* Для использования reserve proxy необходимо следовать руководству: <https://grafana.com/tutorials/run-grafana-behind-a-proxy/>
* Вместо редактирования `grafana.ini` домен и url можно указать в environment variables `GF_SERVER_DOMAIN`, `GF_SERVER_ROOT_URL`
* Рекомендуется использовать БД вместо хранения данных grafana в SQLite, прример конфигурации подключения к PostgreSQL указан в `grafana.yaml`
* Для отправки скриншотов с dashboards вместе с алертами (поддерживаются не все мессенджеры), необходимо использовать grafana image renderer (в конфигурации из примера уже настроено всё необходимое)
* Начальный логин `admin` пароль `admin`

### Grafana image renderer

* Образ `grafana/grafana-image-renderer:3.9.0`
* Установить timezone: например `TZ: "Europe/Moscow"`
* В основном сервисе (grafana) установить подключение к grafana image renderer:

```
GF_RENDERING_SERVER_URL: "http://grafana-image-renderer:8081/render"
GF_RENDERING_CALLBACK_URL: "http://grafana:3000/"
GF_UNIFIED_ALERTING_SCREENSHOTS_CAPTURE: true
GF_LOG_FILTERS: "rendering:debug"
```

### Prometheus

Документация: <https://prometheus.io/docs/alerting/latest/configuration/>

* Образ `prom/prometheus:v2.48.0`
* Порт `9090`
* Команда: `command: ["--config.file=/etc/prometheus/prometheus.yaml", "--web.config.file=/etc/prometheus/web-config.yaml", "--web.enable-lifecycle"]` (`web.enable-lifecycle` для перечитывания `web-config` "на лету")

Пример `prometheus.yaml` (вместо `password` безопасно использовать `password_file` совместно с docker secrets)

```
global:
  scrape_interval: 15s

scrape_configs:

# node exporter (https://github.com/prometheus/node_exporter)
- job_name: 'node'
  static_configs:
  - targets: ['192.168.100.2:9100', '192.168.100.4:9100', '192.168.100.6:9100'] # change 192.168.100.x to your nodes IPs
  basic_auth:
    username: 'admin' # change
    password: 'admin' # change # or use password_file instead (docker secrets)
    # password_file: '/run/secrets/node_exporter_password'
```

Пример `web-config.yaml` (используется для basic_auth)
<https://github.com/prometheus/prometheus/blob/main/documentation/examples/web-config.yaml>

* В данном примере один `web-config.yaml` будет использоваться и для prometheus и для pushgateway
* password_hash генерируется командой `htpasswd -nBC 10 "" | tr -d ':\n'` 
  * Если htpasswd не установлен: для Debian, Ubuntu `sudo apt install apache2-utils`
  * для ОС использующих yum `sudo yum install -y httpd-tools`

```
basic_auth_users:
  admin: $2y$10$K7gXeAs0VbhjHMdlV1Hn0OlWcqIoK7P9s/dVKB3HoyYcLuscxSpXe # change "$2y$10..." to basic auth password_hash
  bobi: $2y$10$1sYkKxi49lGpFdlu7aDkTeWkzvkqaeCTb4PDBR/pNxeETO8N3shZS # you can add more users
```

* Время и максимальный объём хранимых логов можно настроить с помощь команд запуска `--storage.tsdb.retention.time=15d`, `--storage.tsdb.retention.size=0` (напр. `512MB`) 
  * подробнее: <https://prometheus.io/docs/prometheus/latest/storage/>

### Pushgateway

* Образ `prom/pushgateway:v1.6.2`
* Порт 9091
* Команда: `command: ["--web.config.file=/etc/prometheus/web-config.yaml", "--web.enable-lifecycle"]` (`web.enable-lifecycle` для перечитывания `web-config` "на лету")

В конфиг `prometheus.yaml` необходимо добавить в качестве target'а сервис pushgateway с `honor_labels: true`

```
# pushgateway
- job_name: 'pushgateway'
  honor_labels: true
  static_configs:
  - targets: ['pushgateway:9091']
  basic_auth:
    username: 'admin' # change
    password: 'admin' # change # or use password_file instead (docker secrets)
    # password_file: '/run/secrets/node_exporter_password'
```

### Loki

* Образ `grafana/loki:2.9.0`
* Порт 3100 
  * В примере порт закрыт, т.к. при использовании loki внутри docker обычно нет необходимости подключения к loki снаружи стека
  * При необходимости открыть порт: настроить авторизацию (например, средствами [nginx basic auth](https://docs.nginx.com/nginx/admin-guide/security-controls/configuring-http-basic-authentication/))
* Команда: `command: ["--config.file=/etc/loki/loki.yaml"]`

Содержимое стандартного файла конфигурации находится в контейнере в `/etc/loki/local-config.yaml`. Стандартный файл переопределяется командой `--config.file=/etc/loki/loki.yaml`

В стандартный файл можно добавить запрет отправки analytics:

```
analytics:
  reporting_enabled: false
```

Можно добавить настройку очистки старых логов (по умолчанию логи хранятся вечно): <https://grafana.com/docs/loki/latest/operations/storage/retention/>

```
limits_config:
  retention_period: 7d # days to delete old logs, you can change
  max_query_lookback: 7d # days to delete old logs, you can change

chunk_store_config:
  max_look_back_period: 7d # days to delete old logs, you can change

compactor:
  working_directory: /loki/retention
  shared_store: filesystem
  compaction_interval: 15m
  retention_enabled: true
  retention_delete_delay: 2h
  retention_delete_worker_count: 150
```

В конфигурации из примера переопределено значение `query_ingesters_within` для предотвращения проявления [grafana/loki/issues/6043](https://github.com/grafana/loki/issues/6043)

### Promtail

* Образ `grafana/promtail:2.9.0`
* Команда: `["--config.file=/etc/promtail/promtail.yaml", "--config.expand-env=true"]`
* Сервис Promtail должен быть запущен по одному на каждой ноде, с контейнеров которой требуется собирать логи 
  * При этом, файл `/var/promtail/positions_${HOST_HOSTNAME}.yaml` (хранит позиции чтения файлов) уникальный для каждой ноды (в примере это достигается подстановкой ${HOST_HOSTNAME} в имя файла, но если вы не используете общий том для всех нод, то можно не брать во внимание эту особенность.

Пример `promtail.yaml` (за основу взят <https://gist.github.com/ruanbekker/c6fa9bc6882e6f324b4319c5e3622460?permalink_comment_id=4570985#gistcomment-4570985>)

```
server:
  http_listen_address: 0.0.0.0
  http_listen_port: 9080

positions:
  filename: "/var/promtail/positions_${HOST_HOSTNAME}.yaml" # remove "_${HOST_HOSTNAME}" if you do not use docker swarm

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:

- job_name: containers
  static_configs:
  - targets:
      - localhost
    labels:
      job: containers_logs
      node_hostname: "${HOST_HOSTNAME}" # remove line if you do not use docker swarm
      __path__: /var/lib/docker/containers/*/*log

  pipeline_stages:
  - json:
      expressions:
        log: log
        stream: stream
        time: time
        tag: attrs.tag
        # docker compose
        compose_project: attrs."com.docker.compose.project"
        compose_service: attrs."com.docker.compose.service"
        # docker swarm
        stack_name: attrs."com.docker.stack.namespace"
        service_name: attrs."com.docker.swarm.service.name"
        service_id: attrs."com.docker.swarm.service.id"
        task_name: attrs."com.docker.swarm.task.name"
        task_id: attrs."com.docker.swarm.task.id"
        node_id: attrs."com.docker.swarm.node.id"
  - regex:
      expression: "^/var/lib/docker/containers/(?P<container_id>.{12}).+/.+-json.log$"
      source: filename
  - timestamp:
      format: RFC3339Nano
      source: time
  - labels:
      stream:
      container_id:
      tag:
      # docker compose
      compose_project:
      compose_service:
      # docker swarm
      stack_name:
      service_name:
      service_id:
      task_name:
      task_id:
      node_id:
  - output:
      source: log
```

* В `promtail.yaml` можно использовать environment variables (например `custom_label: "${ENV}"`) для этого нужно передать дополнительно команду `-config.expand-env=true`
* **Для корректной работы labels** необходимо внести в `/etc/docker/daemon.json` (на каждом хосте) настройку `log-driver` и `log-opts`: 
  * После внесения изменений в `/etc/docker/daemon.json` необходимо перезапустить docker daemon (`sudo systemctl restart docker`) для ОС, использующих systemctl
  * Для изменения логирования необходимо также перезапустить контейнеры (для compose и обычных контейнеров), контейнеры swarm пересоздаются при перезапуске docker

```
{
   "metrics-addr":"0.0.0.0:9323",
   "log-driver": "json-file",
   "log-opts": {
     "labels-regex": "^.+"
  }
}
```