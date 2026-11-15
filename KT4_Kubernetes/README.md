# Контрольная работа №4. Подготовка инфраструктуры для Kubernetes

## 1. Цель работы

Подготовить инфраструктуру из трёх виртуальных машин для развёртывания Kubernetes-кластера, настроить сетевое взаимодействие и удалённый доступ между узлами.

## 2. Архитектура

Подготовлены три виртуальные машины:

| Узел | Роль | IP-адрес | Операционная система |
|---|---|---|---|
| `k8s-master` | Master + Worker | `192.168.0.24` | Ubuntu 26.04.1 LTS |
| `k8s-worker1` | Worker | `192.168.0.14` | Ubuntu 24.04.5 LTS |
| `k8s-worker2` | Worker | `192.168.0.16` | Ubuntu 24.04.5 LTS |

Все узлы используют архитектуру ARM64 и подключены к общей сети `192.168.0.0/24`.

Для Worker-узлов выделено по 2 vCPU, 2 ГБ оперативной памяти и 20 ГБ дискового пространства.

## 3. Подготовка виртуальных машин

Созданы и настроены три виртуальные машины с Ubuntu Server.

Назначены имена узлов и пользователи:

- `k8s-master` — `meriko`;
- `k8s-worker1` — `core`;
- `k8s-worker2` — `core`.

На управляющем узле изменено системное имя хоста:

```bash
sudo hostnamectl set-hostname k8s-master
```

В `/etc/hosts` обновлена локальная запись:

```text
127.0.1.1 k8s-master
```

## 4. Настройка сети

Для всех виртуальных машин настроен режим сетевого подключения **Bridged Networking**.

Узлы получили IP-адреса из одной локальной подсети:

```text
k8s-master   192.168.0.24
k8s-worker1  192.168.0.14
k8s-worker2  192.168.0.16
```

### 4.1. Проверка сетевой связности

Для проверки соединения между узлами использовалась команда:

```bash
ping -c 4 IP-АДРЕС
```

Проверена связь:
- между Worker1 и Worker2 в обоих направлениях;
- между Worker2 и Master.

**Результат:** сетевые проверки завершились успешно. При проверке соединения Worker1 и Worker2 потерь пакетов не обнаружено.

## 5. Настройка SSH

Для удалённого управления виртуальными машинами использовался протокол SSH.

Проверены подключения ко всем трём узлам:

```bash
ssh meriko@192.168.0.24
ssh core@192.168.0.14
ssh core@192.168.0.16
```

**Результат:** SSH-подключения ко всем трём виртуальным машинам работают успешно.

## 6. Отключение Swap

Для корректной работы Kubernetes в стандартной конфигурации kubelet Swap должен быть отключён.

Проверка выполнена на всех трёх узлах командой:

```bash
swapon --show
```

Результат: команда не выводит активных разделов Swap на Master, Worker1 и Worker2.

Дополнительно проверен файл `/etc/fstab`:
- **Master:** запись `/swap.img` закомментирована, автоматическое подключение Swap отключено.
- **Worker1:** запись Swap отсутствует.
- **Worker2:** запись Swap отсутствует.

Таким образом, на всех трёх узлах Swap отключён, дополнительных изменений на Worker1 и Worker2 не потребовалось.



## 7. Установка Docker и bridge-utils

На всех трёх узлах установлены Docker Engine и `bridge-utils`.

**Установка Docker:**

```bash
wget -qO- https://get.docker.com/ | sudo sh
```

**Установка bridge-utils:**

```bash
sudo apt update
sudo apt install bridge-utils -y
```

**Проверка:**

```bash
docker --version
sudo systemctl is-active docker
dpkg -l bridge-utils
```

Результаты проверки:

| Узел | Docker | bridge-utils |
|---|---|---|
| k8s-master | 29.9.3 | 1.7.1-4ubuntu3 |
| k8s-worker1 | 29.9.3 | 1.7.1-1ubuntu2 |
| k8s-worker2 | 29.9.3 | 1.7.1-1ubuntu2 |

Docker запущен (`active`), оба компонента установлены на всех узлах.


## 8. Настройка SSH-подключения по ключам

На управляющем Mac создан SSH-ключ ED25519. Публичный ключ добавлен в `~/.ssh/authorized_keys` на всех трёх узлах.

**Проверка подключения без пароля:**

```bash
ssh -o BatchMode=yes meriko@192.168.0.24 hostname
ssh -o BatchMode=yes core@192.168.0.14 hostname
ssh -o BatchMode=yes core@192.168.0.16 hostname
```

Все три узла успешно отвечают на SSH-подключение без ввода пароля.

## 9. Установка cri-dockerd

`cri-dockerd` обеспечивает взаимодействие Kubernetes с Docker Engine через интерфейс CRI. Он необходим, поскольку современный Kubernetes больше не поддерживает Docker напрямую.

Версия: **0.4.9**.

Установлен на всех трёх узлах. Сокет `cri-docker.socket` включён и активен.

**Проверка:**

```bash
cri-dockerd --version
systemctl is-active cri-docker.socket
```

Результат на всех узлах: версия `0.4.9`, статус `active`.



## 10. Установка Kubernetes

На всех трёх узлах установлены компоненты Kubernetes версии **v1.37.2**:

- `kubeadm` — создание и настройка кластера.
- `kubelet` — управление Pod на узлах.
- `kubectl` — управление кластером.

Установка выполнена из официального репозитория Kubernetes `pkgs.k8s.io`.

Для всех компонентов включена блокировка автоматического обновления:

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

**Проверка версий:**

```bash
kubelet --version
kubeadm version
kubectl version --client
```

Результат: все три компонента установлены на `k8s-master`, `k8s-worker1` и `k8s-worker2`, версия — `v1.37.2`.



## 11. Подключение Worker-нод к кластеру

После инициализации Control Plane к кластеру подключены две Worker-ноды с помощью `kubeadm join`.

Перед подключением настроен UFW на Master:
- Разрешён входящий TCP-порт 6443 от `192.168.0.14` (Worker1).
- Разрешён входящий TCP-порт 6443 от `192.168.0.16` (Worker2).

Для подключения использовался CRI socket:
`unix:///run/cri-dockerd.sock`.

Обе Worker-ноды успешно прошли TLS Bootstrap и присоединились к кластеру.

**Проверка:**

kubectl get nodes -o wide

Результат:
- k8s-master — Ready
- k8s-worker1 — Ready
- k8s-worker2 — Ready

Все три узла используют Kubernetes v1.37.2.


## 12. Установка и настройка Flannel

Для организации сетевого взаимодействия между Pod
используется Flannel с backend VXLAN.

Flannel установлен в namespace kube-flannel
и работает как DaemonSet — по одному Pod на каждом узле.

Для корректной работы на всех узлах:
- Загружен модуль ядра br_netfilter.
- Настроена автоматическая загрузка модуля.
- Включён bridge-nf-call-iptables.
- Включён bridge-nf-call-ip6tables.
- Включён IPv4 forwarding.

**Проверка:**

kubectl get pods -n kube-flannel -o wide

Результат: все три Pod находятся в состоянии Running,
READY 1/1.



## 13. Развёртывание Nginx

Создан Kubernetes Deployment `nginx` с шестью репликами.

Используемый образ: nginx:stable.

Команда создания:

kubectl create deployment nginx --image=nginx:stable --replicas=6

Проверка:

kubectl get pods -o wide

Результат:
- 6 Pod успешно запущены.
- Все Pod имеют статус Running и Ready 1/1.
- 3 Pod размещены на k8s-worker1.
- 3 Pod размещены на k8s-worker2.


## 14. Создание HTTP Service

Создан Service nginx-http типа ClusterIP.

Команда:

kubectl expose deployment nginx --name=nginx-http --type=ClusterIP --port=80 --target-port=80

Параметры:
- ClusterIP: 192.168.3.24
- Port: 80
- TargetPort: 80
- Backend: 6 Nginx Pod.

Проверка endpoints:

kubectl get endpoints nginx-http

Проверка HTTP:

curl -I http://192.168.3.24

Результат: HTTP/1.1 200 OK.



## 15. Создание TLS Secret и HTTPS-конфигурации

Для настройки HTTPS создан самоподписанный TLS-сертификат с помощью OpenSSL.

Создан Kubernetes Secret:
- Name: `nginx-tls`
- Type: `kubernetes.io/tls`
- Data: 2 (сертификат и приватный ключ).

Создан ConfigMap `nginx-https-config` с конфигурацией Nginx для HTTPS.

Secret и ConfigMap подключены к Deployment `nginx` через volumes и volumeMounts.

Проверка конфигурации:

`kubectl exec deploy/nginx -- nginx -t`

Результат: синтаксис конфигурации корректен, проверка завершилась успешно.

## 16. Создание HTTPS Service

Создан Service `nginx-https` типа ClusterIP.

Команда:

`kubectl expose deployment nginx --name=nginx-https --type=ClusterIP --port=443 --target-port=443`

Параметры:
- ClusterIP: `192.168.3.73`
- Port: `443`
- TargetPort: `443`
- Backend: 6 Nginx Pod.

## 17. Проверка HTTPS и endpoints

Проверка HTTPS:

`curl -k -I https://192.168.3.73`

Результат: `HTTP/1.1 200 OK`.

Проверка endpoints:

`kubectl get endpoints nginx-http nginx-https`

Результат:
- `nginx-http` — 6 endpoints на порту 80.
- `nginx-https` — 6 endpoints на порту 443.

Оба Service успешно направляют запросы на шесть реплик Nginx.

## 18. Вывод

Развёрнут Kubernetes-кластер из трёх узлов. В кластере запущено приложение Nginx с шестью репликами. Настроены HTTP и HTTPS Services. Работоспособность приложения и сетевого взаимодействия проверена с помощью curl. Оба сервиса возвращают HTTP 200 OK.
