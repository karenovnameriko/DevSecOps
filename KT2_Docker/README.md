# KT-2. Установка Docker и создание контейнеров с помощью Ansible

## Цель работы

Освоить автоматизацию установки Docker и управления контейнерами с помощью Ansible Playbook.

## Структура проекта

```text
KT2_Docker/
├── ansible.cfg
├── inventory/
│   └── hosts
├── group_vars/
│   └── docker.yml
├── playbook.yml
├── README.md
└── screenshots/
```

- `ansible.cfg` — настройки Ansible.
- `inventory/hosts` — список управляемых хостов.
- `group_vars/docker.yml` — переменные группы `docker`.
- `playbook.yml` — сценарий установки Docker и создания контейнеров.
- `screenshots/` — скриншоты выполнения работы.

## Используемые технологии

- Ansible
- Docker Engine
- Ubuntu 26.04.1 LTS (ARM64)
- Docker Hub
- Python 3

## Выполнение работы

### 1. Настройка Ansible

Создан файл `ansible.cfg` с настройками inventory и отключённой проверкой SSH host keys.

В `inventory/hosts` добавлен управляемый сервер с группой `docker`.

### 2. Настройка переменных

В файле `group_vars/docker.yml` определены переменные:

```yaml
---
container_count: 4
default_container_name: docker
default_container_image: ubuntu
default_container_command: sleep 1d
```

Они задают количество контейнеров, их имена, используемый образ и команду запуска.

### 3. Создание Playbook

В `playbook.yml` реализованы следующие действия:

1. Установка необходимых системных пакетов.
2. Создание каталога для ключей APT.
3. Загрузка GPG-ключа Docker для проверки подлинности пакетов.
4. Добавление официального репозитория Docker.
5. Установка Docker Engine.
6. Установка Python-модуля Docker в виртуальное окружение.
7. Загрузка образа Ubuntu из Docker Hub.
8. Создание четырёх контейнеров с помощью цикла `with_sequence`.

Для создания контейнеров используется модуль `community.docker.docker_container`.

### 4. Запуск Playbook

Playbook запущен командой:

```bash
ansible-playbook playbook.yml -k -K
```

Параметры:

- `-k` — запрос SSH-пароля.
- `-K` — запрос пароля для повышения привилегий через sudo.

Результат выполнения:

- `ok=9`
- `changed=8`
- `unreachable=0`
- `failed=0`

Ошибок выполнения не обнаружено.

### 5. Проверка установки Docker

Версия Docker проверена командой:

```bash
docker --version
```

Установленная версия:

```text
Docker version 29.9.3
```

### 6. Проверка созданных контейнеров

Для просмотра всех контейнеров использована команда:

```bash
docker ps -a
```

Созданы четыре контейнера на основе образа `ubuntu`:

- `docker1`
- `docker2`
- `docker3`
- `docker4`

Все контейнеры созданы успешно.

## Результат работы

В ходе практической работы с помощью Ansible автоматизирована установка Docker на удалённый сервер Ubuntu, настроен официальный репозиторий Docker, загружен образ Ubuntu и созданы четыре контейнера.

