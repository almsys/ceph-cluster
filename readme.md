# Ceph Production-Ready: Полное руководство по развертыванию и эксплуатации

## Содержание

1. [Введение](#введение)
2. [Архитектура и планирование](#архитектура-и-планирование)
3. [Требования к оборудованию](#требования-к-оборудованию)
4. [Подготовка инфраструктуры](#подготовка-инфраструктуры)
5. [Установка Ceph](#установка-ceph)
6. [Конфигурация и оптимизация](#конфигурация-и-оптимизация)
7. [Масштабирование кластера](#масштабирование-кластера)
8. [Мониторинг и алертинг](#мониторинг-и-алертинг)
9. [Эксплуатация и обслуживание](#эксплуатация-и-обслуживание)
10. [Troubleshooting](#troubleshooting)


---

## Введение

### Что такое Ceph?

Ceph - это распределенная система хранения данных с открытым исходным кодом, предоставляющая:
- **Object Storage** (S3/Swift-совместимый)
- **Block Storage** (RBD для виртуализации)
- **File Storage** (CephFS)

### Для кого это руководство?

Документ предназначен для инженеров, развертывающих production-кластер Ceph и содержит практические рекомендации, основанные на реальном опыте эксплуатации.

### Ключевые принципы production-развертывания

⚠️ **КРИТИЧНО**: 
- Никогда не используйте фактор репликации 1
- Всегда используйте две сети (public + cluster)
- Регулярно заменяйте диски (не старше 3-4 лет)
- Синхронизируйте время на всех узлах
- Тестируйте отказы в тестовом окружении

---

## Архитектура и планирование

### Минимальная production-конфигурация

```
┌─────────────────────────────────────────────────┐
│                  Ceph Cluster                    │
├─────────────┬─────────────┬─────────────────────┤
│   Node 1    │   Node 2    │      Node 3         │
│   (Rack 1)  │   (Rack 2)  │     (Rack 3)        │
├─────────────┼─────────────┼─────────────────────┤
│ MON         │ MON         │ MON                 │
│ MGR(active) │ MGR(standby)│ MGR(standby)        │
│ 10x OSD     │ 10x OSD     │ 10x OSD             │
│ 4TB each    │ 4TB each    │ 4TB each            │
├─────────────┼─────────────┼─────────────────────┤
│ Public:  25Gbit/s  (клиентский трафик)          │
│ Cluster: 25Gbit/s  (репликация, recovery)       │
└─────────────────────────────────────────────────┘
```

### Типовые конфигурации

| Размер кластера | Узлов | OSD/узел | Емкость | RAM/узел | Use Case |
|----------------|-------|----------|---------|----------|----------|
| Малый | 3 | 10 | 120 TB | 96 GB | Dev/Test, Small prod |
| Средний | 5-7 | 12 | 300+ TB | 128 GB | Production |
| Большой | 10+ | 12-24 | 1+ PB | 256+ GB | Enterprise |

### Расчет ресурсов

#### Формулы планирования:

**Емкость с учетом репликации:**
```
Полезная емкость = (Сырая емкость × Количество OSD) / Фактор репликации
Пример: (4 TB × 30 OSD) / 3 = 40 TB полезной емкости
```

**Оперативная память:**
```
RAM = (Количество OSD × 8 GB) + 16 GB (система)
Пример: (10 OSD × 8 GB) + 16 GB = 96 GB RAM
```

**CPU:**
```
Минимум 1-2 ядра на OSD
Рекомендуется: 24+ ядер с высокой частотой (3.0+ GHz)
```

### Топология отказоустойчивости

**CRUSH Map** должна соответствовать физической топологии:

```
root default
    ├── datacenter DC1
    │   ├── rack rack1
    │   │   └── host node1
    │   │       ├── osd.0
    │   │       └── osd.1
    │   ├── rack rack2
    │   │   └── host node2
    │   │       ├── osd.2
    │   │       └── osd.3
    │   └── rack rack3
    │       └── host node3
    │           ├── osd.4
    │           └── osd.5
```

**Правило размещения:**
- Данные реплицируются на разные стойки (rack)
- Минимум 3 стойки для отказоустойчивости
- При отказе одной стойки кластер остается работоспособным

---

## Требования к оборудованию

### Процессор (CPU)

✅ **Рекомендации:**
- **Количество ядер**: 16-24+ (не менее 1-2 ядер на OSD)
- **Частота**: 3.0+ GHz на ядро (чем выше, тем лучше)
- **Архитектура**: x86_64 (Intel Xeon, AMD EPYC)

⚙️ **Обязательные настройки:**
```bash
# Отключить энергосбережение (C-states)
# Ceph активно использует однопоточные операции
# Сон ядер вызывает задержки
echo performance | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

**Почему это важно?**
- Ceph использует много однопоточных операций
- Блокировки в коде требуют быстрых ядер
- Ядро в глубоком сне может добавить миллисекунды задержки

### Оперативная память (RAM)

✅ **Рекомендации по объему:**

| Тип OSD | RAM на OSD | Пример конфигурации |
|---------|------------|---------------------|
| SSD | 8 GB | 10 OSD = 80 GB + 16 GB система = 96 GB |
| HDD | 5 GB | 10 OSD = 50 GB + 16 GB система = 66 GB |
| NVMe | 8-12 GB | Зависит от нагрузки |

⚠️ **ВАЖНО:**
- Ceph **очень требователен к памяти**
- Один OSD может потреблять 30-40 GB при массовом recovery
- Недостаток памяти = OOM killer = каскадные отказы

🛡️ **Защита от OOM:**
```bash
# Ограничение через systemd (обязательно!)
systemctl edit ceph-osd@0.service

[Service]
MemoryMax=8G      # жесткий лимит
MemoryHigh=7G     # soft limit, начинает замедлять
CPUQuota=200%     # не более 2 ядер
```

### Диски (Storage)

#### ❌ Чего избегать:

- **HDD диски** (кроме cold storage)
- **Consumer SSD** без power-loss protection
- **Диски старше 3-4 лет** (высокий риск отказа)
- **Смешивание HDD и SSD в одном пуле**

#### ✅ Рекомендуемые диски:

**Tier 1: Высокая производительность**
- Intel DC P4610/P5520 (NVMe)
- Samsung PM1733/PM9A3 (NVMe)
- Micron 7450 (NVMe)
- Характеристики: 3+ DWPD, power-loss protection

**Tier 2: Баланс цена/производительность**
- Intel D3-S4610/S4510 (SATA SSD)
- Samsung PM883/PM893 (SATA SSD)
- Micron 5300/5400 (SATA SSD)
- Характеристики: 1-3 DWPD, power-loss protection

**Tier 3: Емкость (только для cold storage)**
- Enterprise HDD 7200 RPM (Seagate Exos, WD Ultrastar)
- Использовать только для редко читаемых данных

#### 🔧 Важные характеристики:

1. **Power-Loss Protection** (конденсаторы)
   - Критично для Ceph из-за частых fsync операций
   - Диски без PLP медленнее в 3-5 раз
   - Риск повреждения данных при отключении питания

2. **DWPD (Drive Writes Per Day)**
   - Минимум 1 DWPD для production
   - Оптимально 3+ DWPD для высокой нагрузки
   - Формула расчета: `DWPD = TBW / (емкость в TB × 365 дней × гарантия)`

3. **Размер диска**
   - Оптимально: 2-8 TB
   - Слишком большие диски (16+ TB) увеличивают время recovery
   - Много маленьких дисков = больше OSD = больше RAM

#### 💡 Оптимизация NVMe:

```bash
# Для NVMe можно создать несколько OSD на одном диске
# Это увеличивает утилизацию за счет параллелизма

# Пример: 1 NVMe диск = 3 OSD
ceph-volume lvm create --data /dev/nvme0n1p1
ceph-volume lvm create --data /dev/nvme0n1p2
ceph-volume lvm create --data /dev/nvme0n1p3

# Плюсы:
# - Выше утилизация быстрых дисков
# - Лучший параллелизм

# Минусы:
# - Сложнее в управлении
# - При отказе диска теряется 3 OSD
```

### Сеть (Network)

#### ⚠️ КРИТИЧНО: Две сети обязательны!

```
┌──────────────────────────────────────┐
│          Public Network              │
│      (клиентский трафик)             │
│      25+ Gbit/s bonded               │
└──────────────────────────────────────┘
            ↓         ↓
    ┌───────────┬───────────┐
    │  Client   │  Client   │
    └───────────┴───────────┘

┌──────────────────────────────────────┐
│         Cluster Network              │
│    (репликация, recovery)            │
│      25+ Gbit/s bonded               │
└──────────────────────────────────────┘
            ↓         ↓         ↓
    ┌────────┬────────┬────────┐
    │ Node 1 │ Node 2 │ Node 3 │
    └────────┴────────┴────────┘
```

**Почему две сети?**
- Recovery/rebalance создает огромный трафик
- Без разделения клиенты получают высокие задержки
- Cluster network может быть дешевле (меньше маршрутизации)

#### ✅ Рекомендуемые конфигурации:

| Размер кластера | Public Network | Cluster Network | Bonding |
|----------------|----------------|-----------------|---------|
| Малый (3-5 узлов) | 25 Gbit/s | 25 Gbit/s | LACP |
| Средний (5-10 узлов) | 50 Gbit/s | 50 Gbit/s | LACP |
| Большой (10+ узлов) | 100 Gbit/s | 100 Gbit/s | LACP |

❌ **Неприемлемо для production:**
- Одна сеть (public == cluster)
- Скорость менее 10 Gbit/s
- Общие коммутаторы с другими сервисами без QoS

#### 🔧 Настройка bonding (пример для Ubuntu):

```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  ethernets:
    enp1s0f0:
      dhcp4: no
    enp1s0f1:
      dhcp4: no
    enp2s0f0:
      dhcp4: no
    enp2s0f1:
      dhcp4: no
  
  bonds:
    bond0:  # Public Network
      interfaces: [enp1s0f0, enp1s0f1]
      addresses: [10.0.1.10/24]
      gateway4: 10.0.1.1
      parameters:
        mode: 802.3ad  # LACP
        lacp-rate: fast
        mii-monitor-interval: 100
    
    bond1:  # Cluster Network
      interfaces: [enp2s0f0, enp2s0f1]
      addresses: [10.0.2.10/24]
      parameters:
        mode: 802.3ad
        lacp-rate: fast
        mii-monitor-interval: 100
```

---

## Подготовка инфраструктуры

### Выбор операционной системы

✅ **Рекомендуемые дистрибутивы:**

| ОС | Версия | Ceph версия | Примечание |
|----|--------|-------------|------------|
| Ubuntu LTS | 22.04 | Reef (18.x) | Самый популярный выбор |
| RHEL | 9.x | Reef (18.x) | Enterprise поддержка |
| Rocky Linux | 9.x | Reef (18.x) | RHEL-совместимый, бесплатно |

❌ **Избегать:**
- Ubuntu 14.04/16.04 (устаревшие, без systemd)
- CentOS 7 (EOL)
- Debian stable (старые пакеты)

### Установка и настройка ОС

#### Шаг 1: Базовая установка

```bash
# Обновление системы (Ubuntu)
apt update && apt upgrade -y

# Установка необходимых пакетов
apt install -y \
    python3 \
    python3-pip \
    lvm2 \
    chrony \
    curl \
    wget \
    git \
    vim \
    htop \
    iotop \
    smartmontools \
    nvme-cli

# Для RHEL/Rocky
dnf update -y
dnf install -y python3 lvm2 chrony curl wget git vim htop iotop smartmontools nvme-cli
```

#### Шаг 2: Синхронизация времени (КРИТИЧНО!)

⚠️ **КРИТИЧНО**: Рассинхронизация времени > 50ms вызывает проблемы в Ceph!

```bash
# Установка и настройка chrony
apt install -y chrony

# Настройка /etc/chrony/chrony.conf
cat > /etc/chrony/chrony.conf <<EOF
# Использовать локальные NTP серверы
server ntp1.example.com iburst
server ntp2.example.com iburst
server ntp3.example.com iburst

# Разрешить больший корректирующий шаг
makestep 1.0 3

# Логирование
logdir /var/log/chrony
EOF

# Запуск и проверка
systemctl enable --now chrony
systemctl status chrony

# Проверка синхронизации
chronyc tracking
# Результат должен показывать:
# Leap status     : Normal
# System time     : 0.000000050 seconds slow of NTP time  ← должно быть < 0.05

# Проверка источников
chronyc sources -v
```

**Мониторинг времени:**
```bash
# Добавить в cron проверку
cat > /etc/cron.hourly/check-time-sync <<'EOF'
#!/bin/bash
OFFSET=$(chronyc tracking | grep "System time" | awk '{print $4}')
if (( $(echo "$OFFSET > 0.05" | bc -l) )); then
    echo "WARNING: Time offset $OFFSET seconds" | mail -s "Time Sync Alert" admin@example.com
fi
EOF
chmod +x /etc/cron.hourly/check-time-sync
```

#### Шаг 3: Настройка ядра Linux

```bash
# Создать файл с параметрами ядра
cat > /etc/sysctl.d/99-ceph-performance.conf <<EOF
# ============================================
# Оптимизация сети для Ceph
# ============================================

# Увеличение буферов TCP
net.core.rmem_max = 536870912
net.core.wmem_max = 536870912
net.ipv4.tcp_rmem = 4096 87380 536870912
net.ipv4.tcp_wmem = 4096 65536 536870912

# Размер очереди сетевых пакетов
net.core.netdev_max_backlog = 300000

# TCP настройки
net.ipv4.tcp_max_syn_backlog = 8192
net.core.somaxconn = 8192
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.tcp_tw_reuse = 1

# ============================================
# Управление памятью
# ============================================

# Минимизация swapping
vm.swappiness = 10

# Dirty pages (для оптимизации записи)
vm.dirty_ratio = 10
vm.dirty_background_ratio = 5
vm.dirty_expire_centisecs = 1000

# ============================================
# Файловая система
# ============================================

# Лимиты на открытые файлы
fs.file-max = 1000000
fs.aio-max-nr = 1048576

# ============================================
# Процессы
# ============================================

# Максимум PID (для большого количества OSD)
kernel.pid_max = 4194303

# ============================================
# Дополнительные оптимизации
# ============================================

# Отключение прозрачных huge pages (может вызывать задержки)
# Делается отдельно, см. ниже
EOF

# Применить настройки
sysctl -p /etc/sysctl.d/99-ceph-performance.conf

# Проверка применения
sysctl net.core.rmem_max
sysctl vm.swappiness
```

**Отключение Transparent Huge Pages:**
```bash
# THP может вызывать задержки, рекомендуется отключить
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# Постоянное отключение (через systemd)
cat > /etc/systemd/system/disable-thp.service <<EOF
[Unit]
Description=Disable Transparent Huge Pages (THP)
DefaultDependencies=no
After=sysinit.target local-fs.target
Before=basic.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled'
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/defrag'

[Install]
WantedBy=basic.target
EOF

systemctl daemon-reload
systemctl enable --now disable-thp.service
```

#### Шаг 4: Настройка лимитов

```bash
# Увеличение лимитов для Ceph процессов
cat > /etc/security/limits.d/99-ceph.conf <<EOF
# Лимиты для Ceph демонов
* soft nofile 1048576
* hard nofile 1048576
* soft nproc unlimited
* hard nproc unlimited
* soft memlock unlimited
* hard memlock unlimited
* soft core unlimited
* hard core unlimited
EOF

# Применится после перелогина или перезагрузки
```

#### Шаг 5: Отключение энергосбережения CPU

```bash
# Проверка текущего режима
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# Установка performance режима
apt install -y cpufrequtils

echo 'GOVERNOR="performance"' > /etc/default/cpufrequtils

# Применить немедленно
echo performance | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

systemctl restart cpufrequtils

# Проверка
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
# Все должны показывать: performance
```

#### Шаг 6: Настройка дисков

```bash
# Проверка дисков
lsblk
smartctl -a /dev/sda

# Отключение планировщика I/O для SSD/NVMe
# Для каждого диска:
echo none > /sys/block/sda/queue/scheduler

# Постоянная настройка через udev
cat > /etc/udev/rules.d/60-ceph-disk-scheduler.rules <<EOF
# SSD диски - использовать none или mq-deadline
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="none"

# NVMe диски
ACTION=="add|change", KERNEL=="nvme[0-9]n[0-9]", ATTR{queue/scheduler}="none"

# HDD диски - использовать bfq (если есть)
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="1", ATTR{queue/scheduler}="bfq"
EOF

# Применить правила
udevadm control --reload-rules
udevadm trigger
```

#### Шаг 7: Firewall (если используется)

```bash
# Для Ubuntu (ufw)
ufw allow from 10.0.1.0/24  # Public network
ufw allow from 10.0.2.0/24  # Cluster network

# Для RHEL/Rocky (firewalld)
firewall-cmd --permanent --add-service=ceph
firewall-cmd --permanent --add-service=ceph-mon
firewall-cmd --reload

# Или отключить firewall на cluster network (рекомендуется)
# Cluster network должна быть изолирована
```

---

## Установка Ceph

### Выбор метода установки

| Метод | Версия Ceph | Рекомендация | Примечание |
|-------|-------------|--------------|------------|
| **cephadm** | Octopus+ (15+) | ✅ Рекомендуется | Официальный, контейнеры, автоматизация |
| ceph-ansible | Все версии | ⚠️ Legacy | Для старых версий |
| Rook | Все версии | 🔧 Kubernetes | Только для K8s окружений |
| Ручная установка | Все версии | ❌ Не рекомендуется | Сложно поддерживать |

### Установка через cephadm (рекомендуется)

#### Шаг 1: Установка cephadm на первом узле

```bash
# Установка на Ubuntu
apt install -y cephadm

# ИЛИ скачивание последней версии
curl --silent --remote-name --location \
  https://download.ceph.com/rpm-reef/el9/noarch/cephadm

chmod +x cephadm
mv cephadm /usr/local/bin/

# Проверка версии
cephadm --version
```

#### Шаг 2: Bootstrap первого узла

⚠️ **Подготовка перед bootstrap:**

```bash
# Убедитесь что:
# 1. Hostname разрешается в IP
hostnamectl set-hostname node1.ceph.local
echo "10.0.1.11 node1.ceph.local node1" >> /etc/hosts

# 2. Порты свободны (3300, 6789, 6800-7300, 8443, 9283)
ss -tlnp | grep -E '3300|6789|8443|9283'

# 3. Docker/Podman установлен (cephadm установит автоматически)

# 4. Диски чистые (без партиций)
lsblk
# Очистка дисков при необходимости:
# wipefs -a /dev/sdb
# dd if=/dev/zero of=/dev/sdb bs=1M count=100
```

**Bootstrap кластера:**

```bash
# Базовый bootstrap
cephadm bootstrap \
  --mon-ip 10.0.1.11 \
  --cluster-network 10.0.2.0/24 \
  --initial-dashboard-user admin \
  --initial-dashboard-password 'StrongP@ssw0rd!' \
  --allow-fqdn-hostname \
  --single-host-defaults

# ИЛИ с дополнительными параметрами для production
cephadm bootstrap \
  --mon-ip 10.0.1.11 \
  --cluster-network 10.0.2.0/24 \
  --initial-dashboard-user admin \
  --initial-dashboard-password 'StrongP@ssw0rd!' \
  --allow-fqdn-hostname \
  --dashboard-password-noupdate \
  --ssh-user root \
  --skip-monitoring-stack  # установим позже с настройками

# Процесс займет 5-10 минут
# В конце будет выведена информация:
# - URL dashboard: https://node1:8443
# - Логин: admin
# - Пароль: StrongP@ssw0rd!
# - SSH публичный ключ для других узлов
```

**Сохранить важную информацию:**
```bash
# Сохранить конфигурацию
mkdir -p /root/ceph-install
ceph config dump > /root/ceph-install/initial-config.txt

# Сохранить SSH ключ
cp /etc/ceph/ceph.pub /root/ceph-install/

# Сохранить admin ключ
ceph auth get client.admin > /root/ceph-install/admin-keyring.txt
```

#### Шаг 3: Подготовка остальных узлов

```bash
# На всех остальных узлах (node2, node3, ...):

# 1. Применить все настройки из раздела "Подготовка инфраструктуры"

# 2. Настроить hostname и hosts
hostnamectl set-hostname node2.ceph.local
cat >> /etc/hosts <<EOF
10.0.1.11 node1.ceph.local node1
10.0.1.12 node2.ceph.local node2
10.0.1.13 node3.ceph.local node3
EOF

# 3. Установить Docker/Podman (если нет)
apt install -y docker.io
systemctl enable --now docker
```

#### Шаг 4: Добавление узлов в кластер

```bash
# На первом узле (node1):

# Копирование SSH ключей
ssh-copy-id -f -i /etc/ceph/ceph.pub root@node2
ssh-copy-id -f -i /etc/ceph/ceph.pub root@node3

# Проверка SSH доступа
ssh root@node2 hostname
ssh root@node3 hostname

# Добавление узлов в кластер
ceph orch host add node2 10.0.1.12 --labels _admin
ceph orch host add node3 10.0.1.13 --labels _admin

# Проверка списка узлов
ceph orch host ls

# Результат:
# HOST   ADDR         LABELS  STATUS
# node1  10.0.1.11    _admin  
# node2  10.0.1.12    _admin  
# node3  10.0.1.13    _admin  
```

#### Шаг 5: Развертывание MON и MGR демонов

```bash
# Автоматическое размещение MON (рекомендуется нечетное число: 3, 5, 7)
ceph orch apply mon --placement="3 node1 node2 node3"

# Автоматическое размещение MGR (минимум 2 для HA)
ceph orch apply mgr --placement="3 node1 node2 node3"

# Проверка размещения
ceph orch ps --daemon-type mon
ceph orch ps --daemon-type mgr

# Проверка кворума
ceph mon stat
# Должно показать: e2: 3 mons at {...}, election epoch X, leader X, quorum 0,1,2

# Проверка MGR
ceph mgr stat
# Должно показать: активный MGR и standby
```

#### Шаг 6: Настройка CRUSH Map (топология)

⚠️ **КРИТИЧНО**: CRUSH Map должна соответствовать физической топологии!

```bash
# Создание структуры стоек (racks)
ceph osd crush add-bucket rack1 rack
ceph osd crush add-bucket rack2 rack
ceph osd crush add-bucket rack3 rack

# Перемещение хостов в стойки
ceph osd crush move node1 rack=rack1
ceph osd crush move node2 rack=rack2
ceph osd crush move node3 rack=rack3

# Перемещение стоек под root
ceph osd crush move rack1 root=default
ceph osd crush move rack2 root=default
ceph osd crush move rack3 root=default

# Проверка топологии
ceph osd crush tree

# Результат должен быть:
# ID   CLASS  WEIGHT   TYPE NAME         
# -1          0.00000  root default      
# -3          0.00000      rack rack1    
# -2          0.00000          host node1
# -5          0.00000      rack rack2    
# -4          0.00000          host node2
# -7          0.00000      rack rack3    
# -6          0.00000          host node3
```

**Создание правила размещения для отказоустойчивости:**

```bash
# Создать правило: данные на разных стойках
ceph osd crush rule create-replicated replicated_rack \
  default rack host

# Просмотр правил
ceph osd crush rule ls
ceph osd crush rule dump replicated_rack
```

#### Шаг 7: Добавление OSD (дисков)

**Просмотр доступных устройств:**

```bash
# Посмотреть все доступные диски во всем кластере
ceph orch device ls

# Пример вывода:
# Hostname  Path      Type  Serial           Size   Health  Ident  Fault  Available
# node1     /dev/sdb  ssd   S455NX0M123456   3.8T   Unknown  N/A    N/A    Yes
# node1     /dev/sdc  ssd   S455NX0M123457   3.8T   Unknown  N/A    N/A    Yes
# node2     /dev/sdb  ssd   S455NX0M123458   3.8T   Unknown  N/A    N/A    Yes
# ...

# Посмотреть диски конкретного узла
ceph orch device ls --hostname=node1
```

**Добавление OSD - Вариант 1: Автоматически все доступные:**

```bash
# Добавить все доступные диски во всем кластере
ceph orch apply osd --all-available-devices

# ИЛИ на конкретных хостах
ceph orch apply osd --all-available-devices --hostname=node1
ceph orch apply osd --all-available-devices --hostname=node2
ceph orch apply osd --all-available-devices --hostname=node3

# Проверка процесса
ceph orch ls osd
ceph -w  # watch mode, показывает прогресс
```

**Добавление OSD - Вариант 2: Вручную (контролируемо):**

```bash
# Добавление конкретных дисков
ceph orch daemon add osd node1:/dev/sdb
ceph orch daemon add osd node1:/dev/sdc
ceph orch daemon add osd node1:/dev/sdd
# ... повторить для всех дисков

# Массовое добавление через скрипт
for node in node1 node2 node3; do
  for device in /dev/sd{b..k}; do
    ceph orch daemon add osd $node:$device
  done
done
```

**Проверка добавленных OSD:**

```bash
# Просмотр всех OSD
ceph osd tree

# Пример вывода:
# ID  CLASS  WEIGHT   TYPE NAME         STATUS  REWEIGHT  PRI-AFF
# -1         90.00000  root default                              
# -3         30.00000      rack rack1                            
# -2         30.00000          host node1                        
#  0   ssd    3.00000              osd.0     up   1.00000  1.00000
#  1   ssd    3.00000              osd.1     up   1.00000  1.00000
# ...

# Статистика OSD
ceph osd stat
# Должно показать: X osds: X up, X in

# Детальная информация
ceph osd df tree
```

#### Шаг 8: Создание пулов данных

**⚠️ КРИТИЧНО: Правильная настройка репликации!**

```bash
# НИКОГДА НЕ ИСПОЛЬЗУЙТЕ size=1 !
# Это приведет к потере данных при отказе диска

# Создание пула для general purpose
ceph osd pool create mypool 128 128 replicated

# Настройка репликации
ceph osd pool set mypool size 3        # 3 копии данных (ОБЯЗАТЕЛЬНО!)
ceph osd pool set mypool min_size 2    # минимум 2 копии для работы

# Установить правило размещения
ceph osd pool set mypool crush_rule replicated_rack

# Включить автоматическое управление PG
ceph osd pool set mypool pg_autoscale_mode on

# Включить приложение (например, для RBD)
ceph osd pool application enable mypool rbd

# Проверка настроек пула
ceph osd pool get mypool all
```

**Расчет количества PG (Placement Groups):**

Для старых версий Ceph без pg_autoscale:

```bash
# Формула: (OSD × 100) / replica_size / pool_count
# Пример: (30 OSD × 100) / 3 / 2 = 500 PG

# Округлить до степени 2: 512

ceph osd pool create mypool 512 512 replicated
```

**Типовые пулы:**

```bash
# RBD пул (для виртуальных машин, Kubernetes)
ceph osd pool create rbd-pool 256 256 replicated
ceph osd pool set rbd-pool size 3 min_size 2
ceph osd pool application enable rbd-pool rbd

# CephFS метаданные
ceph osd pool create cephfs-metadata 128 128 replicated
ceph osd pool set cephfs-metadata size 3 min_size 2

# CephFS данные
ceph osd pool create cephfs-data 512 512 replicated
ceph osd pool set cephfs-data size 3 min_size 2

# S3/RadosGW пулы
ceph osd pool create rgw-root 32 32 replicated
ceph osd pool create rgw-data 512 512 replicated
ceph osd pool set rgw-data size 3 min_size 2
```

#### Шаг 9: Проверка работоспособности

```bash
# Общее состояние кластера
ceph -s

# Ожидаемый результат:
#   cluster:
#     id:     abc123-def456-...
#     health: HEALTH_OK     ← ВАЖНО!
# 
#   services:
#     mon: 3 daemons, quorum node1,node2,node3
#     mgr: node1(active, since 1h), standbys: node2, node3
#     osd: 30 osds: 30 up, 30 in
# 
#   data:
#     pools:   1 pools, 128 pgs
#     objects: 0 objects, 0 B
#     usage:   900 MiB used, 120 TiB / 120 TiB avail
#     pgs:     128 active+clean

# Детальная проверка здоровья
ceph health detail

# Если есть warnings
ceph health detail
# Обработать каждый warning!

# Проверка версий
ceph versions

# Тест записи/чтения
rados -p mypool bench 10 write --no-cleanup
rados -p mypool bench 10 seq
rados -p mypool bench 10 rand
```

---

## Конфигурация и оптимизация

### Базовая конфигурация Ceph

#### Настройка двух сетей

```bash
# Проверка текущих настроек
ceph config get mon public_network
ceph config get mon cluster_network

# Установка сетей (если не было сделано при bootstrap)
ceph config set mon public_network 10.0.1.0/24
ceph config set mon cluster_network 10.0.2.0/24

# Проверка применения
ceph config dump | grep network
```

#### Оптимизация общих параметров

```bash
# ============================================
# Параметры восстановления (recovery)
# ============================================

# КРИТИЧНО: замедлить recovery чтобы не убить клиентов
ceph config set osd osd_max_backfills 1              # только 1 backfill на OSD
ceph config set osd osd_recovery_max_active 1        # только 1 recovery операция
ceph config set osd osd_recovery_op_priority 1       # низкий приоритет
ceph config set osd osd_recovery_sleep 0.1           # пауза между операциями (сек)

# Для HDD увеличить паузу
ceph config set osd osd_recovery_sleep_hdd 0.2

# Для SSD меньшая пауза
ceph config set osd osd_recovery_sleep_ssd 0.05

# ============================================
# Таймауты отказа OSD
# ============================================

# КРИТИЧНО: увеличить таймаут чтобы избежать ложных срабатываний
ceph config set global mon_osd_down_out_interval 3600   # 1 час вместо 600 сек (10 мин)

# Время доометки OSD как down
ceph config set global mon_osd_report_timeout 900       # 15 минут

# ============================================
# Отключение debug логов (производительность)
# ============================================

ceph config set global debug_ms 0
ceph config set global debug_mon 0
ceph config set global debug_osd 0
ceph config set global debug_bluestore 0
ceph config set global debug_rocksdb 0

# ============================================
# Настройка скрабирования (проверка целостности)
# ============================================

# Выполнять скрабирование ночью
ceph config set osd osd_scrub_begin_hour 1      # начало в 01:00
ceph config set osd osd_scrub_end_hour 6        # конец в 06:00

# Нагрузка при которой НЕ скрабировать
ceph config set osd osd_scrub_load_threshold 0.5

# Интервалы скрабирования
ceph config set osd osd_scrub_min_interval 86400          # минимум 1 раз в день
ceph config set osd osd_scrub_max_interval 604800         # максимум раз в неделю
ceph config set osd osd_deep_scrub_interval 1209600       # deep scrub раз в 2 недели

# Приоритет скрабирования
ceph config set osd osd_scrub_priority 1                  # низкий приоритет
ceph config set osd osd_scrub_sleep 0.1                   # пауза между операциями
```

#### Оптимизация для SSD

```bash
# ============================================
# BlueStore настройки для SSD
# ============================================

# Размер кэша для SSD (3GB на OSD)
ceph config set osd bluestore_cache_size_ssd 3221225472

# Параллелизм операций
ceph config set osd bluestore_max_ops 4096
ceph config set osd bluestore_max_bytes 268435456

# Оптимизация RocksDB для SSD
ceph config set osd bluestore_rocksdb_options \
  "compression=kNoCompression,max_write_buffer_number=4,min_write_buffer_number_to_merge=1,recycle_log_file_num=4,write_buffer_size=268435456,writable_file_max_buffer_size=0,compaction_readahead_size=2097152,max_background_compactions=2"

# ============================================
# OSD операции
# ============================================

# Увеличение параллелизма
ceph config set osd osd_op_num_threads_per_shard 2
ceph config set osd osd_op_num_shards 8

# Client операции
ceph config set osd osd_client_message_size_cap 524288000
```

#### Оптимизация для HDD (если используются)

```bash
# Размер кэша для HDD (1GB на OSD)
ceph config set osd bluestore_cache_size_hdd 1073741824

# Меньше параллелизма
ceph config set osd osd_op_num_shards 5
```

#### Настройка для производительных кластеров

```bash
# ============================================
# Агрессивные настройки для быстрых кластеров
# ИСПОЛЬЗОВАТЬ ОСТОРОЖНО!
# ============================================

# Большие буферы
ceph config set osd osd_client_message_size_cap 1073741824

# Больше операций одновременно
ceph config set osd osd_max_backfills 2
ceph config set osd osd_recovery_max_active 3

# Меньше задержек
ceph config set osd osd_recovery_sleep 0.01
ceph config set osd osd_recovery_sleep_ssd 0

# Настройка heartbeat
ceph config set osd osd_heartbeat_grace 20
ceph config set osd osd_heartbeat_interval 6
```

### Ограничение ресурсов через systemd

**⚠️ КРИТИЧНО для предотвращения OOM!**

```bash
# Создать override для OSD сервисов
# Для каждого OSD отдельно

# Узнать список OSD на узле
systemctl list-units 'ceph-osd@*.service'

# Создать override для OSD
systemctl edit ceph-osd@0.service

# Добавить следующее содержимое:
[Service]
# Ограничение памяти
MemoryMax=8G          # жесткий лимит (OSD будет убит при превышении)
MemoryHigh=7G         # мягкий лимит (начнется throttling)

# Ограничение CPU
CPUQuota=200%         # не более 2 ядер

# Ограничение I/O (опционально)
IOWeight=100          # нормальный приоритет I/O

# Применить для всех OSD
for osd_id in {0..9}; do
  systemctl edit ceph-osd@${osd_id}.service
  # вставить конфигурацию выше
done

# Перезагрузить systemd
systemctl daemon-reload

# Проверка лимитов
systemctl show ceph-osd@0.service | grep -E 'Memory|CPU'
```

**Увеличение лимитов во время recovery:**

```bash
# Во время массового recovery временно увеличить
systemctl edit ceph-osd@0.service

[Service]
MemoryMax=16G
MemoryHigh=14G

systemctl daemon-reload
systemctl restart ceph-osd@0.service

# ПОСЛЕ завершения recovery вернуть обратно!
```

### Настройка флагов для обслуживания

```bash
# ============================================
# Флаги для контролируемого обслуживания
# ВСЕГДА устанавливать ПЕРЕД обслуживанием!
# ============================================

# Запретить помечать OSD как out при падении
ceph osd set noout

# Запретить начинать recovery
ceph osd set norecover

# Запретить backfill
ceph osd set nobackfill

# Запретить ребалансировку
ceph osd set norebalance

# Запретить deep scrub
ceph osd set nodeep-scrub

# Запретить scrub
ceph osd set noscrub

# Проверка установленных флагов
ceph osd dump | grep flags

# ПОСЛЕ завершения обслуживания ОБЯЗАТЕЛЬНО снять:
ceph osd unset noout
ceph osd unset norecover
ceph osd unset nobackfill
ceph osd unset norebalance
ceph osd unset nodeep-scrub
ceph osd unset noscrub
```

**Типовые сценарии использования флагов:**

```bash
# Сценарий 1: Замена одного диска
ceph osd set noout norebalance nobackfill
# ... замена диска ...
ceph osd unset noout norebalance nobackfill

# Сценарий 2: Перезагрузка узла
ceph osd set noout norecover nobackfill
# ... перезагрузка узла ...
ceph osd unset noout norecover nobackfill

# Сценарий 3: Плановое обслуживание (отключение целой стойки)
ceph osd set noout norecover nobackfill norebalance
# ... обслуживание ...
ceph osd unset noout norecover nobackfill norebalance

# Сценарий 4: Ночное обслуживание (с восстановлением)
ceph osd set noout norebalance
# ... обслуживание ...
ceph osd unset noout norebalance
# recovery начнется автоматически
```

---

## Масштабирование кластера

### Стратегия добавления узлов

#### Исходная конфигурация: 3 узла

```
Начало: 3 узла × 10 OSD = 30 OSD
Емкость: 30 × 4TB = 120TB raw = 40TB usable (replica 3)
```

### Этап 1: Расширение 3 → 5 узлов

#### Расчет ребалансировки

```
Было: 30 OSD
Добавляем: 20 OSD (2 узла × 10 OSD)
Станет: 50 OSD

Процент ребалансировки = (20 / 50) × 100% = 40% данных

При 40TB данных переместится ~16TB
Время при 10 Gbit/s: ~4-5 дней
Время при 25 Gbit/s: ~1.5-2 дня
```

#### Шаг 1: Подготовка новых узлов

```bash
# На новых узлах (node4, node5):

# 1. Установить ОС Ubuntu 22.04
# 2. Применить ВСЕ настройки из раздела "Подготовка инфраструктуры":
#    - Синхронизация времени (chrony)
#    - Настройка ядра (sysctl)
#    - Лимиты (limits.conf)
#    - Отключение энергосбережения CPU
#    - Настройка сети (2 сети!)

# 3. Настроить hostname и /etc/hosts
hostnamectl set-hostname node4.ceph.local

cat >> /etc/hosts <<EOF
10.0.1.11 node1.ceph.local node1
10.0.1.12 node2.ceph.local node2
10.0.1.13 node3.ceph.local node3
10.0.1.14 node4.ceph.local node4
10.0.1.15 node5.ceph.local node5
EOF

# 4. Проверка синхронизации времени
chronyc tracking
# System time должен быть < 0.05 секунд!

# 5. Проверка сети
ping -c 3 10.0.1.11  # public network
ping -c 3 10.0.2.11  # cluster network
```

#### Шаг 2: КРИТИЧНО - Установить флаги ПЕРЕД добавлением!

```bash
# На мастер-узле (node1):

# ⚠️ БЕЗ ЭТОГО Ceph начнет ребалансировку СРАЗУ!
ceph osd set noout
ceph osd set norebalance
ceph osd set nobackfill

# Проверка
ceph osd dump | grep flags
# Должно показать: noout,norebalance,nobackfill

# Проверка состояния
ceph -s
```

#### Шаг 3: Добавление узлов в кластер

```bash
# На мастер-узле (node1):

# Копирование SSH ключей
ssh-copy-id -f -i /etc/ceph/ceph.pub root@node4
ssh-copy-id -f -i /etc/ceph/ceph.pub root@node5

# Проверка SSH
ssh root@node4 'hostname && date'
ssh root@node5 'hostname && date'
# Проверить что время синхронизировано!

# Добавление узлов
ceph orch host add node4 10.0.1.14 --labels _admin
ceph orch host add node5 10.0.1.15 --labels _admin

# Проверка
ceph orch host ls

# Результат:
# HOST   ADDR         LABELS  STATUS
# node1  10.0.1.11    _admin  
# node2  10.0.1.12    _admin  
# node3  10.0.1.13    _admin  
# node4  10.0.1.14    _admin   ← новый
# node5  10.0.1.15    _admin   ← новый
```

#### Шаг 4: Обновление CRUSH Map

```bash
# Создать новые стойки
ceph osd crush add-bucket rack4 rack
ceph osd crush add-bucket rack5 rack

# Переместить новые хосты в стойки
ceph osd crush move node4 rack=rack4
ceph osd crush move node5 rack=rack5

# Переместить стойки под root
ceph osd crush move rack4 root=default
ceph osd crush move rack5 root=default

# Проверка топологии
ceph osd crush tree

# Должно показать:
# ID   CLASS  WEIGHT   TYPE NAME         
# -1          120.00   root default      
# -3           30.00       rack rack1    
# -2           30.00           host node1
# -5           30.00       rack rack2    
# -4           30.00           host node2
# -7           30.00       rack rack3    
# -6           30.00           host node3
# -9            0.00       rack rack4    ← новый
# -8            0.00           host node4
# -11           0.00       rack rack5    ← новый
# -10           0.00           host node5
```

#### Шаг 5: Добавление MON демонов (опционально)

```bash
# При 5 узлах рекомендуется 5 MON

# Проверка текущих MON
ceph mon stat
# e2: 3 mons at {...}

# Добавление MON на новые узлы
ceph orch apply mon --placement="5 node1 node2 node3 node4 node5"

# ИЛИ автоматически
ceph orch apply mon 5

# Проверка
ceph mon stat
# e4: 5 mons at {...}

ceph orch ps --daemon-type mon
```

#### Шаг 6: Добавление OSD

```bash
# Просмотр доступных дисков на новых узлах
ceph orch device ls --hostname=node4
ceph orch device ls --hostname=node5

# Добавление всех доступных дисков
ceph orch apply osd --all-available-devices --hostname=node4
ceph orch apply osd --all-available-devices --hostname=node5

# ИЛИ вручную для каждого диска
for device in /dev/sd{b..k}; do
  ceph orch daemon add osd node4:$device
  ceph orch daemon add osd node5:$device
done

# Мониторинг добавления (займет 5-15 минут)
watch -n 10 'ceph osd tree'

# Проверка что все OSD добавлены
ceph osd tree
# Должно показать 50 OSD: 50 up, 50 in

# Проверка CRUSH весов
ceph osd df tree
# Все узлы должны иметь примерно одинаковый WEIGHT
```

#### Шаг 7: Контролируемая ребалансировка

**⚠️ КРИТИЧНЫЙ ЭТАП: Правильная настройка recovery!**

```bash
# Настройка параметров ребалансировки
# ДЛЯ МИНИМАЛЬНОГО ВЛИЯНИЯ НА КЛИЕНТОВ:

ceph config set osd osd_max_backfills 1
ceph config set osd osd_recovery_max_active 1
ceph config set osd osd_recovery_op_priority 1
ceph config set osd osd_recovery_sleep 0.1          # пауза 100ms
ceph config set osd osd_recovery_sleep_ssd 0.05

# Опционально: ограничить скорость (bytes/sec)
ceph config set osd osd_recovery_max_single_start 1
ceph config set osd osd_recovery_max_chunk 8388608  # 8MB chunks

# Проверка настроек
ceph config dump | grep recovery

# СНЯТИЕ ФЛАГОВ ПОЭТАПНО:

# 1. Снять norebalance ПЕРВЫМ
ceph osd unset norebalance

# Подождать 5-10 минут, мониторить
watch -n 10 'ceph -s'

# Ceph покажет:
#   progress: Rebalancing (0.1% complete)
#   misplaced: X/Y objects (40.0%)

# 2. Снять nobackfill
ceph osd unset nobackfill

# 3. Снять noout
ceph osd unset noout

# Проверка что флаги сняты
ceph osd dump | grep flags
```

#### Шаг 8: Мониторинг ребалансировки

```bash
# Постоянный мониторинг (в отдельном терминале)
watch -n 10 'ceph -s'

# Вывод будет показывать:
#   cluster:
#     health: HEALTH_WARN
#             Degraded data redundancy: X/Y objects degraded (40.0%), ...
#   progress:
#     Rebalancing (15.2% complete)

# Детальная статистика
ceph pg dump | grep -E 'active\+(remapped|backfilling)'

# Процент завершения
ceph status | grep progress

# Ожидаемое время
ceph status | grep 'Rebalancing'

# Проверка нагрузки на OSD
ceph osd perf

# Проверка утилизации сети
iftop -i bond1  # cluster network

# Проверка задержек клиентов
rados -p mypool bench 10 write -t 16
rados -p mypool bench 10 rand -t 16

# Если клиенты страдают: ЗАМЕДЛИТЬ
ceph config set osd osd_recovery_sleep 0.2
ceph config set osd osd_max_backfills 1

# Если все хорошо: МОЖНО УСКОРИТЬ
ceph config set osd osd_recovery_sleep 0.05
ceph config set osd osd_max_backfills 2
```

**Ожидаемое время ребалансировки:**

| Сеть | Данных | Настройка | Время |
|------|--------|-----------|-------|
| 10 Gbit/s | 16 TB | Медленная | 5-7 дней |
| 25 Gbit/s | 16 TB | Медленная | 2-3 дня |
| 25 Gbit/s | 16 TB | Средняя | 1-2 дня |
| 50 Gbit/s | 16 TB | Средняя | 12-24 часа |
| 100 Gbit/s | 16 TB | Быстрая | 6-12 часов |

#### Шаг 9: Проверка после завершения

```bash
# Дождаться HEALTH_OK
ceph -s

# Ожидаемый результат:
#   cluster:
#     id:     abc123...
#     health: HEALTH_OK  ← КРИТИЧНО!
# 
#   services:
#     mon: 5 daemons, quorum node1,node2,node3,node4,node5
#     mgr: node1(active), standbys: node2,node3,node4,node5
#     osd: 50 osds: 50 up, 50 in  ← было 30
#
#   data:
#     pools:   2 pools, 512 pgs
#     objects: 1.2M objects, 40 TiB
#     usage:   120 TiB used, 80 TiB / 200 TiB avail  ← больше места
#     pgs:     512 active+clean  ← все здоровы!

# Проверка распределения данных
ceph osd df tree

# Проверка вариации заполненности
ceph osd df tree | grep VAR
# VAR должна быть близка к 1.0 (идеально 0.9-1.1)

# Детальная проверка здоровья
ceph health detail
# Не должно быть никаких warning!

# Проверка PG
ceph pg stat
ceph pg dump | grep -v active+clean
# Не должно быть вывода (все PG active+clean)

# Тест производительности
rados -p mypool bench 30 write --no-cleanup
rados -p mypool bench 30 seq
rados -p mypool bench 30 rand

# Сброс очереди команд
rados -p mypool cleanup
```

### Этап 2: Расширение 5 → 7 узлов

Процесс идентичный, но процент ребалансировки меньше:

```
Было: 50 OSD
Добавляем: 20 OSD (2 узла × 10 OSD)
Станет: 70 OSD

Процент ребалансировки = (20 / 70) × 100% ≈ 28.5% данных

При 40TB данных переместится ~11.4TB
Время при 25 Gbit/s: ~1-2 дня
```

**Повторить все шаги из Этапа 1:**
1. Подготовка node6 и node7
2. Установка флагов
3. Добавление в кластер
4. Обновление CRUSH Map
5. Добавление MON (теперь 7 MON)
6. Добавление OSD
7. Контролируемая ребалансировка
8. Проверка

### Стратегии добавления узлов

#### Стратегия 1: Постепенное добавление (минимальный риск)

```bash
# День 1: Добавить только node4
ceph osd set noout norebalance nobackfill
# ... добавление node4 ...
ceph osd unset norebalance nobackfill noout
# Ждать завершения ребалансировки (3-5 дней)

# День 6: Добавить node5
ceph osd set noout norebalance nobackfill
# ... добавление node5 ...
ceph osd unset norebalance nobackfill noout
# Ждать завершения

# Плюсы:
# - Меньше одновременной нагрузки
# - Проще откатиться при проблемах
# - Безопаснее

# Минусы:
# - Очень долго (удвоенное время)
# - Больше ребалансировок (2 раза)
```

#### Стратегия 2: Ночная ребалансировка (минимальное влияние)

```bash
# Добавить узлы с флагами
ceph osd set noout norebalance nobackfill
# ... добавление node4 и node5 ...

# Настроить ребалансировку только ночью
ceph config set osd osd_recovery_begin_hour 22  # 22:00
ceph config set osd osd_recovery_end_hour 6     # 06:00

# Снять флаги
ceph osd unset norebalance nobackfill noout

# Плюсы:
# - Минимальное влияние на клиентов днем
# - Контролируемое время работы

# Минусы:
# - ОЧЕНЬ долгое время завершения (недели)
# - Только 8 часов в сутки для recovery
```

#### Стратегия 3: Быстрая ребалансировка в выходные

```bash
# Запланировать на пятницу вечер

# Установить агрессивные параметры
ceph config set osd osd_max_backfills 4
ceph config set osd osd_recovery_max_active 3
ceph config set osd osd_recovery_sleep 0
ceph config set osd osd_recovery_sleep_ssd 0

# Добавить узлы и снять флаги
# ...

# Мониторинг в выходные
# Ожидание: 24-48 часов

# Понедельник: вернуть обычные параметры
ceph config set osd osd_max_backfills 1
ceph config set osd osd_recovery_max_active 1
ceph config set osd osd_recovery_sleep 0.1

# Плюсы:
# - Быстро (1-2 дня)
# - Минимальное влияние на production (выходные)

# Минусы:
# - Нужно дежурить в выходные
# - Высокая нагрузка на кластер
```

### Удаление узлов из кластера

⚠️ **Используйте с осторожностью!**

```bash
# Сценарий: вывод node5 из кластера

# 1. Установить флаги
ceph osd set noout norebalance nobackfill

# 2. Пометить все OSD узла как out
for osd_id in $(ceph osd ls-tree node5); do
  ceph osd out osd.$osd_id
done

# 3. Снять флаг norebalance (начнется перемещение данных)
ceph osd unset norebalance

# 4. Дождаться завершения (данные переедут на другие OSD)
watch -n 10 'ceph -s'
# Ждать HEALTH_OK

# 5. Остановить OSD на node5
ssh root@node5 'systemctl stop ceph-osd.target'

# 6. Удалить OSD из кластера
for osd_id in $(ceph osd ls-tree node5); do
  ceph osd purge osd.$osd_id --yes-i-really-mean-it
done

# 7. Удалить узел из CRUSH Map
ceph osd crush remove node5

# 8. Удалить узел из кластера
ceph orch host rm node5 --force

# 9. Снять оставшиеся флаги
ceph osd unset noout nobackfill

# 10. Проверка
ceph osd tree
ceph -s
```

---

## Мониторинг и алертинг

### Включение Ceph Dashboard

```bash
# Dashboard должен быть включен после bootstrap
# Проверка
ceph mgr module ls | grep dashboard

# Если выключен
ceph mgr module enable dashboard

# Создание пользователя
ceph dashboard ac-user-create admin <strong-password> administrator

# Включение SSL (если нужен кастомный сертификат)
ceph dashboard create-self-signed-cert

# Настройка адреса
ceph config set mgr mgr/dashboard/server_addr 0.0.0.0
ceph config set mgr mgr/dashboard/server_port 8443

# Проверка URL
ceph mgr services

# Результат:
# {
#     "dashboard": "https://node1:8443/"
# }

# Доступ: https://node1:8443
# Логин: admin
# Пароль: <strong-password>
```

### Настройка Prometheus и Grafana

```bash
# Включить модуль Prometheus
ceph mgr module enable prometheus

# Настройка эндпоинта
ceph config set mgr mgr/prometheus/server_addr 0.0.0.0
ceph config set mgr mgr/prometheus/server_port 9283

# Проверка метрик
curl http://localhost:9283/metrics

# Развертывание стека мониторинга через cephadm
ceph orch apply grafana --placement="node1"
ceph orch apply prometheus --placement="node1"
ceph orch apply alertmanager --placement="node1"

# Проверка сервисов
ceph orch ps | grep -E 'grafana|prometheus|alertmanager'

# URL Grafana (по умолчанию)
# http://node1:3000
# Логин: admin
# Пароль: admin (изменить при первом входе)
```

**Интеграция Prometheus с внешним экземпляром:**

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'ceph'
    static_configs:
      - targets: ['node1:9283', 'node2:9283', 'node3:9283']
```

### Критичные метрики для алертов

**Alertmanager правила для Ceph:**

```yaml
# ceph-alerts.yml
groups:
  - name: ceph
    interval: 30s
    rules:
      # Состояние кластера
      - alert: CephHealthWarning
        expr: ceph_health_status == 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Ceph cluster health WARNING"
          description: "Cluster {{ $labels.cluster }} has status HEALTH_WARN"

      - alert: CephHealthError
        expr: ceph_health_status == 2
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Ceph cluster health ERROR"
          description: "Cluster {{ $labels.cluster }} has status HEALTH_ERR"

      # OSD состояние
      - alert: CephOSDDown
        expr: ceph_osd_up == 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Ceph OSD Down"
          description: "OSD {{ $labels.ceph_daemon }} is down for more than 5 minutes"

      - alert: CephOSDNearFull
        expr: ceph_osd_utilization > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Ceph OSD near full"
          description: "OSD {{ $labels.ceph_daemon }} is {{ $value }}% full"

      - alert: CephOSDFull
        expr: ceph_osd_utilization > 90
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Ceph OSD critically full"
          description: "OSD {{ $labels.ceph_daemon }} is {{ $value }}% full"

      # Заполненность кластера
      - alert: CephClusterNearFull
        expr: ceph_cluster_total_used_bytes / ceph_cluster_total_bytes > 0.75
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Ceph cluster near full"
          description: "Cluster is {{ $value | humanizePercentage }} full"

      # Degraded objects
      - alert: CephDegradedObjects
        expr: ceph_pg_degraded > 0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Ceph has degraded objects"
          description: "{{ $value }} objects are degraded"

      # PG состояние
      - alert: CephPGsNotActive
        expr: ceph_pg_total - ceph_pg_active > 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Some PGs are not active"
          description: "{{ $value }} PGs are not in active state"

      # MON кворум
      - alert: CephMonQuorumLost
        expr: ceph_mon_quorum_count < ((ceph_mon_count / 2) + 1)
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Ceph MON quorum lost"
          description: "Only {{ $value }} monitors in quorum"
```

### Мониторинг SMART дисков

```bash
# Включить модуль devicehealth
ceph mgr module enable devicehealth

# Настройка
ceph config set mgr mgr/devicehealth/enable_monitoring true
ceph config set mgr mgr/devicehealth/scrape_frequency 86400  # раз в день

# Проверка SMART метрик
ceph device ls
ceph device info <device-id>
ceph device query-daemon-health-metrics <daemon>

# Просмотр предупреждений
ceph device get-health-metrics <device-id>

# Ожидаемые предикторы отказа:
# - Reallocated_Sector_Ct > 0
# - Current_Pending_Sector > 0
# - Offline_Uncorrectable > 0
# - UDMA_CRC_Error_Count > 1000
```

**Ручная проверка SMART:**

```bash
# На каждом узле
for disk in /dev/sd{a..z}; do
  [ -b "$disk" ] && smartctl -a $disk | grep -E 'Model|Serial|Reallocated|Pending|Uncorrectable|Temperature'
done

# Для NVMe
for disk in /dev/nvme*n1; do
  [ -b "$disk" ] && nvme smart-log $disk | grep -E 'temperature|available_spare|percentage_used'
done
```

### Скрипты мониторинга

**Ежедневная проверка здоровья:**

```bash
#!/bin/bash
# /usr/local/bin/ceph-daily-check.sh

MAILTO="admin@example.com"
SUBJECT="Ceph Daily Health Check"

# Проверка здоровья
HEALTH=$(ceph health detail)

# Проверка заполненности
USAGE=$(ceph df | grep -A 5 "GLOBAL")

# Проверка OSD
OSD_STAT=$(ceph osd stat)
OSD_DOWN=$(ceph osd tree | grep down | wc -l)

# Проверка PG
PG_STAT=$(ceph pg stat)
PG_NOT_CLEAN=$(ceph pg dump | grep -v active+clean | wc -l)

# Формирование отчета
REPORT="Ceph Cluster Health Report
========================

Health Status:
$HEALTH

Usage:
$USAGE

OSD Status:
$OSD_STAT
OSDs Down: $OSD_DOWN

PG Status:
$PG_STAT
PGs not clean: $PG_NOT_CLEAN
"

# Отправка email
echo "$REPORT" | mail -s "$SUBJECT" $MAILTO

# Сохранение в лог
echo "$REPORT" >> /var/log/ceph-daily-check.log
```

```bash
# Добавить в cron
cat > /etc/cron.d/ceph-daily-check <<EOF
0 8 * * * root /usr/local/bin/ceph-daily-check.sh
EOF
```

---

## Эксплуатация и обслуживание

### Замена неисправного диска

**Сценарий: OSD.5 на node2 вышел из строя**

```bash
# Шаг 1: Диагностика
ceph osd tree | grep down
# 5   ssd    3.00000         osd.5        down   1.00000  1.00000

# Проверка логов
journalctl -u ceph-osd@5 -n 100

# Проверка SMART (если диск виден)
smartctl -a /dev/sdc

# Шаг 2: Установить флаги (если заменяете быстро < 1 часа)
ceph osd set noout
# Это предотвратит начало recovery

# Шаг 3: Остановить OSD (если еще работает)
systemctl stop ceph-osd@5

# Шаг 4: Удалить OSD из кластера
ceph osd out osd.5
ceph osd purge osd.5 --yes-i-really-mean-it

# Шаг 5: Физическая замена диска
# - Выключить сервер (или hot-swap если поддерживается)
# - Заменить диск
# - Включить сервер

# Шаг 6: Проверка нового диска
lsblk
smartctl -a /dev/sdc

# Шаг 7: Очистка диска (если нужно)
wipefs -a /dev/sdc
dd if=/dev/zero of=/dev/sdc bs=1M count=100

# Шаг 8: Добавление нового OSD
ceph orch daemon add osd node2:/dev/sdc

# ИЛИ вручную (старый метод)
ceph-volume lvm create --data /dev/sdc

# Шаг 9: Проверка
ceph osd tree
# Новый OSD должен появиться (может быть другой ID)

# Шаг 10: Снять флаг (начнется recovery)
ceph osd unset noout

# Шаг 11: Мониторинг recovery
watch -n 10 'ceph -s'
# Ждать HEALTH_OK
```

### Замена целого узла

```bash
# Сценарий: node3 вышел из строя, нужно заменить весь сервер

# Шаг 1: Установить флаги
ceph osd set noout norecover nobackfill norebalance

# Шаг 2: Пометить все OSD узла как out
for osd_id in $(ceph osd ls-tree node3); do
  ceph osd out osd.$osd_id
done

# Шаг 3: Если узел работает - остановить демоны
ssh node3 'systemctl stop ceph.target'

# Шаг 4: Физическая замена сервера
# - Установка нового сервера
# - Установка ОС
# - Применение всех настроек

# Шаг 5: Добавление узла обратно (с тем же именем)
ssh-copy-id -f -i /etc/ceph/ceph.pub root@node3
ceph orch host add node3 10.0.1.13 --labels _admin

# Шаг 6: Обновление CRUSH
ceph osd crush move node3 rack=rack3

# Шаг 7: Добавление OSD
ceph orch apply osd --all-available-devices --hostname=node3

# Шаг 8: Удаление старых OSD (если были)
# Посмотреть старые OSD с node3
ceph osd tree | grep node3 | grep down
# Удалить каждый
ceph osd purge osd.X --yes-i-really-mean-it

# Шаг 9: Снять флаги
ceph osd unset noout norecover nobackfill norebalance

# Шаг 10: Мониторинг
watch -n 10 'ceph -s'
```

### Обновление Ceph

**⚠️ КРИТИЧНО: Всегда читать Release Notes!**

```bash
# Проверка текущей версии
ceph versions

# Проверка доступных версий для обновления
ceph orch upgrade ls

# ПЕРЕД ОБНОВЛЕНИЕМ:
# 1. Сделать backup конфигурации
mkdir -p /root/ceph-backup-$(date +%F)
ceph config dump > /root/ceph-backup-$(date +%F)/config.txt
ceph osd crush dump > /root/ceph-backup-$(date +%F)/crush-map.json
ceph auth export > /root/ceph-backup-$(date +%F)/auth-keys.txt

# 2. Проверить здоровье кластера
ceph health detail
# Должно быть HEALTH_OK!

# 3. Установить флаги (опционально, для safety)
ceph osd set noout
ceph osd set noscrub
ceph osd set nodeep-scrub

# Запуск обновления
ceph orch upgrade start --image quay.io/ceph/ceph:v18.2.1

# Мониторинг процесса обновления
ceph orch upgrade status

# Результат:
# {
#     "target_image": "quay.io/ceph/ceph:v18.2.1",
#     "in_progress": true,
#     "which": "Upgrading mgr.node1",
#     "services_complete": ["mon", "crash"],
#     "progress": "3/12 daemons upgraded",
#     "message": ""
# }

# Постоянный мониторинг
watch -n 30 'ceph orch upgrade status; ceph -s'

# Обновление идет в порядке:
# 1. MON дем оны
# 2. MGR демоны
# 3. OSD демоны (по одному с каждого хоста)
# 4. MDS демоны (если есть)
# 5. RGW демоны (если есть)

# После завершения (несколько часов)
ceph versions
# Все демоны должны быть на новой версии

# Снять флаги
ceph osd unset noout noscrub nodeep-scrub

# Проверка
ceph health detail
```

**Откат обновления (если что-то пошло не так):**

```bash
# Остановить обновление
ceph orch upgrade stop

# Откатиться на предыдущую версию
ceph orch upgrade start --image quay.io/ceph/ceph:v18.2.0

# Мониторинг отката
ceph orch upgrade status
```

### Регулярное обслуживание

**Еженедельные задачи:**

```bash
#!/bin/bash
# /usr/local/bin/ceph-weekly-maintenance.sh

# 1. Проверка SMART дисков
echo "=== SMART Check ==="
ceph device ls
ceph device query-daemon-health-metrics

# 2. Проверка балансировки
echo "=== OSD Balance Check ==="
ceph osd df tree | grep VAR
# VAR должна быть 0.9-1.1

# 3. Принудительный scrub (если давно не было)
echo "=== Initiating scrub ==="
for pool in $(ceph osd pool ls); do
  ceph osd pool scrub $pool
done

# 4. Очистка старых снапшотов (если используются)
echo "=== Snapshot cleanup ==="
# rbd snap ls <pool>/<image>
# rbd snap purge <pool>/<image>

# 5. Проверка логов на ошибки
echo "=== Error log check ==="
journalctl -u 'ceph-*' --since "7 days ago" | grep -i error

# 6. Backup конфигурации
echo "=== Config backup ==="
BACKUP_DIR="/root/ceph-backup-weekly-$(date +%F)"
mkdir -p $BACKUP_DIR
ceph config dump > $BACKUP_DIR/config.txt
ceph osd crush dump > $BACKUP_DIR/crush-map.json
ceph auth export > $BACKUP_DIR/auth-keys.txt
```

**Ежемесячные задачи:**

- Проверка производительности (rados bench)
- Анализ трендов заполненности
- Планирование замены дисков (> 3 лет)
- Тестирование backup/restore
- Обзор безопасности (CVE, обновления)

### Disaster Recovery

**Backup critical data:**

```bash
# 1. Конфигурация MON
ceph mon dump > mon-dump-$(date +%F).txt

# 2. Конфигурация OSD
ceph osd dump > osd-dump-$(date +%F).txt

# 3. CRUSH Map
ceph osd getcrushmap -o crushmap-$(date +%F).bin
crushtool -d crushmap-$(date +%F).bin -o crushmap-$(date +%F).txt

# 4. Ключи авторизации
ceph auth list > auth-list-$(date +%F).txt

# 5. Конфигурация пулов
for pool in $(ceph osd pool ls); do
  ceph osd pool get $pool all > pool-$pool-$(date +%F).txt
done

# 6. RBD образы (список)
for pool in $(ceph osd pool ls | grep rbd); do
  rbd ls $pool > rbd-list-$pool-$(date +%F).txt
done
```

**Восстановление MON кворума (если потеряли > 50% MON):**

```bash
# КРАЙНИЙ СЛУЧАЙ! Используйте только если потеряли большинство MON!

# На surviving MON:
systemctl stop ceph-mon.target

# Восстановление из monmap
ceph-mon -i <mon-id> --extract-monmap /tmp/monmap

# Удалить failed MON из monmap
monmaptool /tmp/monmap --rm <failed-mon-id>

# Inject monmap обратно
ceph-mon -i <mon-id> --inject-monmap /tmp/monmap

# Запустить MON
systemctl start ceph-mon.target


----—--------------
-------------------

# Ceph Cluster - Troubleshooting Guide & Checklist

## Troubleshooting

### 1. Диагностика общего состояния кластера

#### Проблема: HEALTH_WARN или HEALTH_ERR
```bash
# Проверить детальный статус
ceph health detail
ceph status

# Проверить логи
kubectl logs -n rook-ceph deploy/rook-ceph-operator
kubectl logs -n rook-ceph -l app=rook-ceph-mon
```

**Распространенные причины:**
- OSD down/out
- Недостаточно места на дисках
- Clock skew между нодами
- Медленные операции (slow ops)

---

### 2. Проблемы с OSD

#### OSD не запускаются
```bash
# Проверить статус OSD
ceph osd tree
ceph osd stat

# Проверить логи конкретного OSD
kubectl logs -n rook-ceph -l app=rook-ceph-osd -c osd

# Проверить deployments OSD
kubectl get deployment -n rook-ceph -l app=rook-ceph-osd
```

**Решения:**
- Проверить доступность дисков на нодах
- Убедиться что диски не используются другими процессами: `lsblk`
- Проверить права доступа к дискам
- Очистить failed OSD: 
  ```bash
  ceph osd purge <osd-id> --yes-i-really-mean-it
  ```

#### OSD full или nearfull
```bash
# Проверить использование
ceph osd df tree

# Проверить pool квоты
ceph osd pool get-quota <pool-name>
```

**Решения:**
- Увеличить storage capacity (добавить диски/ноды)
- Удалить старые snapshots
- Настроить автоматическую очистку: 
  ```bash
  ceph osd set-nearfull-ratio 0.85
  ceph osd set-full-ratio 0.95
  ```

#### Медленные OSD (slow ops)
```bash
# Найти медленные операции
ceph health detail | grep slow

# Проверить latency дисков
ceph osd perf
```

**Решения:**
- Проверить загрузку CPU/Memory на нодах
- Проверить disk I/O: `iostat -x 1`
- Проверить network latency между нодами
- Временно увеличить лимиты: 
  ```bash
  ceph config set osd osd_op_complaint_time 60
  ```

---

### 3. Проблемы с MON

#### MON quorum потерян
```bash
# Проверить статус MON
ceph mon stat
kubectl get pod -n rook-ceph -l app=rook-ceph-mon

# Проверить connectivity между MON
kubectl exec -n rook-ceph <mon-pod> -- ceph daemon mon.<id> mon_status
```

**Решения:**
- Убедиться что минимум 3 MON запущены
- Проверить часовые пояса на нодах: `timedatectl`
- Установить NTP синхронизацию
- Восстановить MON из backup если необходимо

#### Clock skew
```bash
# Проверить разницу времени
ceph health detail | grep clock
```

**Решение:**
```bash
# На каждой ноде
sudo timedatectl set-ntp true
sudo systemctl restart systemd-timesyncd

# В кластере
ceph config set global mon_clock_drift_allowed 0.05
```

---

### 4. Проблемы с PG (Placement Groups)

#### PG stuck in activating/peering
```bash
# Проверить статус PG
ceph pg stat
ceph pg dump | grep -E 'stale|down|inconsistent|incomplete'

# Детали конкретной PG
ceph pg <pg-id> query
```

**Решения:**
- Дождаться завершения recovery
- Если PG застряла надолго:
  ```bash
  ceph pg repair <pg-id>
  ```
- В крайнем случае:
  ```bash
  ceph pg force-recovery <pg-id>
  ceph pg force-backfill <pg-id>
  ```

#### Undersized/degraded PGs
```bash
ceph health detail | grep pg
ceph pg ls undersized
ceph pg ls degraded
```

**Решения:**
- Обычно проблема решается автоматически после восстановления OSD
- Проверить что достаточно OSD для replicas
- Временно уменьшить replication size (не рекомендуется для prod):
  ```bash
  ceph osd pool set <pool-name> min_size 1
  ```

#### Inconsistent PGs
```bash
# Найти inconsistent PG
ceph health detail | grep inconsistent

# Детали
ceph pg <pg-id> list_unfound
```

**Решение:**
```bash
# Попытаться восстановить
ceph pg repair <pg-id>

# Если не помогло (используйте осторожно!)
ceph pg <pg-id> mark_unfound_lost revert
```

---

### 5. Проблемы с RBD (Block Storage)

#### PVC не создается
```bash
# Проверить StorageClass
kubectl get sc
kubectl describe sc <storage-class-name>

# Проверить RBD provisioner
kubectl logs -n rook-ceph -l app=csi-rbdplugin-provisioner

# Проверить события PVC
kubectl describe pvc <pvc-name>
```

**Решения:**
- Проверить наличие пула: `ceph osd lspool`
- Создать пул если отсутствует
- Проверить CSI pods: `kubectl get pods -n rook-ceph | grep csi`

#### RBD image не монтируется к поду
```bash
# Проверить CSI nodeplugin
kubectl logs -n rook-ceph -l app=csi-rbdplugin -c csi-rbdplugin

# Проверить статус RBD на ноде
rbd ls <pool-name>
rbd info <pool-name>/<image-name>
```

**Решения:**
- Проверить kernel modules: `lsmod | grep rbd`
- Загрузить модуль: `sudo modprobe rbd`
- Проверить network connectivity между нодой и Ceph cluster

#### RBD image в exclusive-lock состоянии
```bash
# Проверить locks
rbd lock list <pool-name>/<image-name>
```

**Решение:**
```bash
# Удалить lock (только если точно знаете что под не использует image)
rbd lock remove <pool-name>/<image-name> <lock-id> <locker>
```

---

### 6. Проблемы с CephFS

#### MDS недоступен
```bash
# Проверить статус MDS
ceph fs status
ceph mds stat

# Проверить MDS pods
kubectl get pods -n rook-ceph -l app=rook-ceph-mds
```

**Решения:**
- Убедиться что MDS pods запущены
- Проверить логи: `kubectl logs -n rook-ceph -l app=rook-ceph-mds`
- Restart MDS если необходимо

#### CephFS медленно работает
```bash
# Проверить клиентов
ceph fs status
ceph fs perf stats

# Проверить cache
ceph daemon mds.<id> cache status
```

**Решения:**
- Увеличить MDS cache: 
  ```bash
  ceph config set mds mds_cache_memory_limit 4294967296
  ```
- Добавить дополнительные active MDS
- Проверить underlying OSDs

---

### 7. Проблемы с производительностью

#### Общее замедление кластера
```bash
# Проверить общую производительность
ceph osd perf
rados bench -p <pool> 10 write --no-cleanup
rados bench -p <pool> 10 seq

# Проверить network
iperf3 между нодами

# Проверить disk I/O
fio --name=test --size=1G --rw=randwrite --bs=4k --direct=1 --filename=/path/to/test
```

**Решения:**
- Tune kernel parameters:
  ```bash
  echo "net.core.rmem_max = 134217728" >> /etc/sysctl.conf
  echo "net.core.wmem_max = 134217728" >> /etc/sysctl.conf
  sysctl -p
  ```
- Настроить Ceph параметры:
  ```bash
  ceph config set global osd_max_backfills 1
  ceph config set global osd_recovery_max_active 3
  ```

---

### 8. Проблемы с сетью

#### High network latency
```bash
# Проверить latency
ceph osd perf
ceph daemon osd.<id> dump_historic_ops

# Проверить network interface
ip -s link
ethtool <interface>
```

**Решения:**
- Проверить MTU settings (рекомендуется 9000 для jumbo frames)
- Проверить network congestion
- Использовать dedicated network для Ceph cluster/public networks

---

## Pre-Flight Checklist

### Перед установкой Ceph

- [ ] **Hardware Requirements**
  - [ ] Минимум 3 ноды для production
  - [ ] Каждая нода имеет минимум 16GB RAM
  - [ ] Выделенные диски для OSD (не root диск)
  - [ ] 10Gbit network (рекомендуется)
  - [ ] Отдельные network interfaces для public/cluster сети (опционально)

- [ ] **Node Preparation**
  - [ ] Синхронизация времени (NTP) настроена на всех нодах
  - [ ] Все ноды имеют уникальные hostnames
  - [ ] Firewall rules настроены (порты 6789, 3300, 6800-7300)
  - [ ] Kernel modules: rbd, ceph
  - [ ] Диски очищены и не имеют partitions/filesystems

- [ ] **Kubernetes Cluster**
  - [ ] Kubernetes версии 1.22+
  - [ ] Helm 3 установлен
  - [ ] kubectl настроен и работает
  - [ ] CSI snapshotter CRDs установлены (для snapshots)

- [ ] **Network**
  - [ ] Network connectivity между всеми нодами проверена
  - [ ] DNS resolution работает
  - [ ] MTU настроен одинаково на всех нодах
  - [ ] Bandwidth тест пройден (iperf3)

---

## Post-Installation Checklist

### Сразу после установки

- [ ] **Cluster Health**
  - [ ] `ceph status` показывает HEALTH_OK
  - [ ] Все OSD в состоянии up и in: `ceph osd tree`
  - [ ] Все MON в quorum: `ceph mon stat`
  - [ ] PGs в активном состоянии: `ceph pg stat`

- [ ] **Storage Configuration**
  - [ ] Pools созданы с правильным replication size
  - [ ] StorageClasses созданы и работают
  - [ ] Тестовый PVC создается и монтируется
  - [ ] RBD и CephFS provisioners работают

- [ ] **Security**
  - [ ] RBAC настроен правильно
  - [ ] Secrets созданы для authentication
  - [ ] Network policies применены (если используются)

- [ ] **Monitoring**
  - [ ] Prometheus metrics доступны
  - [ ] Grafana dashboards импортированы
  - [ ] Alerts настроены
  - [ ] Log aggregation работает

- [ ] **Backup**
  - [ ] Backup strategy определена
  - [ ] Disaster recovery plan создан
  - [ ] Документация обновлена

---

## Daily Operations Checklist

### Ежедневные проверки

- [ ] **Health Check**
  - [ ] `ceph -s` - общий статус
  - [ ] `ceph health detail` - детали warning/errors
  - [ ] Проверить Grafana dashboards

- [ ] **Capacity Planning**
  - [ ] `ceph df` - проверить свободное место
  - [ ] `ceph osd df tree` - проверить balance OSD
  - [ ] Спрогнозировать когда понадобится расширение

- [ ] **Performance**
  - [ ] Проверить slow ops: `ceph health detail | grep slow`
  - [ ] Проверить latency: `ceph osd perf`
  - [ ] Проверить I/O metrics в monitoring

- [ ] **Logs Review**
  - [ ] Проверить критические ошибки в логах
  - [ ] Проверить Kubernetes events: `kubectl get events -n rook-ceph`

---

## Weekly Maintenance Checklist

### Еженедельные задачи

- [ ] **Deep Scrub Status**
  - [ ] Проверить что scrubbing выполняется регулярно
  - [ ] `ceph pg dump | grep scrub`
  - [ ] Проверить errors после scrub

- [ ] **Backup Verification**
  - [ ] Проверить что backups создаются
  - [ ] Протестировать восстановление (периодически)
  - [ ] Проверить retention policies

- [ ] **Updates Review**
  - [ ] Проверить доступные обновления Rook/Ceph
  - [ ] Запланировать maintenance window если нужно
  - [ ] Проверить changelog и breaking changes

- [ ] **Capacity Trends**
  - [ ] Проанализировать рост данных за неделю
  - [ ] Обновить capacity planning
  - [ ] Заказать новое оборудование если необходимо

- [ ] **Documentation**
  - [ ] Обновить runbooks при наличии инцидентов
  - [ ] Документировать изменения конфигурации
  - [ ] Обновить architecture diagrams

---

## Monthly Maintenance Checklist

### Ежемесячные задачи

- [ ] **Disaster Recovery Test**
  - [ ] Протестировать backup restore процедуру
  - [ ] Проверить RTO/RPO соответствие
  - [ ] Обновить DR документацию

- [ ] **Performance Tuning**
  - [ ] Проанализировать performance metrics за месяц
  - [ ] Оптимизировать placement groups если нужно
  - [ ] Tune CRUSH map при необходимости

- [ ] **Security Audit**
  - [ ] Проверить access logs
  - [ ] Обновить certificates если приближается expiry
  - [ ] Review и rotate secrets

- [ ] **Upgrade Planning**
  - [ ] Проверить roadmap Rook/Ceph
  - [ ] Запланировать апгрейды
  - [ ] Подготовить тестовую среду для апгрейдов

---

## Emergency Response Checklist

### При возникновении критической проблемы

1. **Immediate Assessment** (0-5 минут)
   - [ ] Проверить `ceph status`
   - [ ] Идентифицировать scope проблемы (OSD/MON/MDS/PG)
   - [ ] Проверить есть ли data loss
   - [ ] Оценить impact на applications

2. **Containment** (5-15 минут)
   - [ ] Изолировать проблемные компоненты если возможно
   - [ ] Остановить распространение проблемы
   - [ ] Сохранить logs для анализа
   - [ ] Уведомить stakeholders

3. **Investigation** (15-30 минут)
   - [ ] Собрать детальные логи
   - [ ] Проверить recent changes
   - [ ] Проверить monitoring/alerting данные
   - [ ] Найти root cause

4. **Recovery** (30+ минут)
   - [ ] Применить решение из runbook
   - [ ] Восстановить нормальную работу
   - [ ] Проверить data integrity
   - [ ] Мониторить стабильность после recovery

5. **Post-Incident** (после восстановления)
   - [ ] Написать incident report
   - [ ] Провести post-mortem meeting
   - [ ] Обновить runbooks/documentation
   - [ ] Внедрить preventive measures
   - [ ] Update monitoring/alerting

---

## Useful Commands Reference

### Quick Diagnostic Commands

```bash
# Общий статус
ceph -s
ceph health detail
ceph df

# OSD status
ceph osd tree
ceph osd stat
ceph osd df tree

# MON status
ceph mon stat
ceph quorum_status

# PG status
ceph pg stat
ceph pg dump

# Pool info
ceph osd pool ls detail
ceph osd pool stats

# Performance
ceph osd perf
ceph tell osd.* bench

# Logs
kubectl logs -n rook-ceph deploy/rook-ceph-operator
kubectl logs -n rook-ceph -l app=rook-ceph-mon
kubectl logs -n rook-ceph -l app=rook-ceph-osd

# Events
kubectl get events -n rook-ceph --sort-by='.lastTimestamp'
```

### Configuration Commands

```bash
# Show config
ceph config dump
ceph config show osd.<id>

# Set config
ceph config set osd <parameter> <value>
ceph config set global <parameter> <value>

# Reset config
ceph config rm osd <parameter>
```

---

## Monitoring Key Metrics

### Critical Metrics to Watch

**Cluster Health:**
- Overall health status
- Number of down/out OSDs
- PG states distribution
- MON quorum status

**Capacity:**
- Used capacity percentage (alert at 75%, critical at 85%)
- Available capacity
- Growth rate
- Individual OSD usage

**Performance:**
- IOPS (read/write)
- Throughput (MB/s)
- Latency (commit/apply)
- Client I/O

**Network:**
- Bandwidth utilization
- Packet loss
- Latency between nodes

**Resource Usage:**
- CPU utilization
- Memory usage
- Disk I/O wait
- Network I/O

---

## Contact & Escalation

### Support Channels

1. **Documentation:** 
   - Rook: https://rook.io/docs/
   - Ceph: https://docs.ceph.com/

2. **Community:**
   - Rook Slack: https://rook.io/slack
   - Ceph Mailing Lists: https://ceph.io/community/

3. **Emergency Contacts:**
   - [Добавить контакты вашей команды]
   - [Добавить on-call rotation]
   - [Добавить escalation path]

---

