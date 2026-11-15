# KT3. Установка и настройка Prometheus

## Цель работы

Установить и настроить систему мониторинга Prometheus на сервере Ubuntu, создать отдельного системного пользователя, настроить права доступа, запустить сервис через systemd и обеспечить доступ к веб-интерфейсу.

## Используемое окружение

- ОС: Ubuntu 26.04.1 LTS
- Архитектура: ARM64 (`aarch64`)
- Prometheus: 3.16.0
- Среда: виртуальная машина VMware
- Адрес сервера: `172.16.222.129`
- Порт веб-интерфейса: `9090`

## Ход выполнения работы

### 1. Проверка системы и обновление пакетов

Проверена версия операционной системы и архитектура сервера:

```bash
lsb_release -a
uname -m
whoami
```

Обновлён список доступных пакетов:

```bash
sudo apt update
```

Скриншот: `screenshots/01_server_info.png`

Скриншот: `screenshots/02_apt_update.png`

### 2. Создание пользователя и группы

Для запуска Prometheus создана отдельная системная группа и системный пользователь без возможности входа в систему:

```bash
sudo groupadd --system prometheus

sudo useradd -s /usr/sbin/nologin \
  --system -g prometheus prometheus
```

Проверка созданных пользователя и группы:

```bash
getent group prometheus
id prometheus
```

Скриншот: `screenshots/03_prometheus_user.png`

### 3. Создание каталогов

Созданы каталоги для конфигурационных файлов и хранения данных Prometheus:

```bash
sudo mkdir -p /etc/prometheus
sudo mkdir -p /var/lib/prometheus
```

- `/etc/prometheus` — конфигурационные файлы.
- `/var/lib/prometheus` — данные временных рядов (TSDB).

Скриншот: `screenshots/04_directories.png`

### 4. Загрузка и распаковка Prometheus

Поскольку сервер использует архитектуру ARM64, загружен соответствующий дистрибутив Prometheus версии 3.16.0.

Переход во временный каталог и загрузка архива:

```bash
cd /tmp

wget https://github.com/prometheus/prometheus/releases/download/v3.16.0/prometheus-3.16.0.linux-arm64.tar.gz
```

Распаковка архива:

```bash
tar -zxvf prometheus-3.16.0.linux-arm64.tar.gz
```

Исполняемые файлы размещены в `/usr/local/bin`, конфигурационный файл — в `/etc/prometheus`:

```bash
sudo mv /tmp/prometheus-3.16.0.linux-arm64/prometheus /usr/local/bin/

sudo mv /tmp/prometheus-3.16.0.linux-arm64/promtool /usr/local/bin/

sudo mv /tmp/prometheus-3.16.0.linux-arm64/prometheus.yml /etc/prometheus/
```

Скриншот: `screenshots/05_installation.png`

**Примечание:** в архиве Prometheus 3.16.0 для ARM64 отсутствовали каталоги `consoles` и `console_libraries`, поэтому они не переносились. Для запуска сервиса и работы веб-интерфейса они не потребовались.

### 5. Настройка прав доступа

Владельцем исполняемых файлов, конфигурации и каталога хранения данных назначен пользователь `prometheus`:

```bash
sudo chown prometheus:prometheus /usr/local/bin/prometheus

sudo chown prometheus:prometheus /usr/local/bin/promtool

sudo chown -R prometheus:prometheus /etc/prometheus

sudo chown -R prometheus:prometheus /var/lib/prometheus
```

Это позволяет запускать Prometheus от отдельного непривилегированного пользователя.

Скриншот: `screenshots/06_permissions_service.png`

### 6. Создание systemd-службы

Создан файл `/etc/systemd/system/prometheus.service`:

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/

[Install]
WantedBy=multi-user.target
```

В конфигурации службы указано:

- `User` и `Group` — пользователь и группа, от имени которых работает сервис.
- `ExecStart` — команда запуска Prometheus.
- `--config.file` — путь к конфигурационному файлу.
- `--storage.tsdb.path` — путь для хранения данных мониторинга.
- `WantedBy=multi-user.target` — запуск службы при загрузке системы.

После создания службы выполнена перезагрузка конфигурации systemd и запуск Prometheus:

```bash
sudo systemctl daemon-reload
sudo systemctl start prometheus
sudo systemctl status prometheus
```

Для автоматического запуска при загрузке системы выполнено:

```bash
sudo systemctl enable prometheus
sudo systemctl is-enabled prometheus
```

Служба успешно запущена, её состояние — `active (running)`. Автозапуск включён (`enabled`).

Скриншот: `screenshots/07_prometheus_running.png`

### 7. Настройка межсетевого экрана

Проверено состояние UFW:

```bash
sudo ufw status
```

Так как межсетевой экран активен, добавлено правило для доступа к веб-интерфейсу Prometheus:

```bash
sudo ufw allow 9090/tcp
```

После этого проверены правила:

```bash
sudo ufw status
```

Порт `9090/tcp` разрешён для входящих подключений по IPv4 и IPv6.

### 8. Проверка веб-интерфейса

В браузере открыт адрес:

```text
http://172.16.222.129:9090
```

Веб-интерфейс Prometheus успешно загрузился.

Скриншот: `screenshots/08_prometheus_web.png`

## Результат работы

В ходе практической работы:

- Установлен Prometheus версии 3.16.0 для архитектуры ARM64.
- Созданы отдельные системные пользователь и группа.
- Подготовлены каталоги конфигурации и хранения данных.
- Настроены права доступа к файлам и каталогам.
- Создан systemd unit для управления службой.
- Prometheus успешно запущен и добавлен в автозагрузку.
- В UFW открыт порт `9090/tcp`.
- Проверена доступность веб-интерфейса Prometheus.

**Результат:** Prometheus успешно установлен, настроен и работает на сервере Ubuntu.
