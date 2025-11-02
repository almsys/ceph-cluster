# Ceph: Backup, настройка кластера между DC

## Содержание

1. [Backup стратегии для Ceph](#Backup-стратегии-для-Ceph)
2. [Multi-Datacenter конфигурации](#Multi-Datacenter-конфигурации)
3. [Требования к оборудованию](#требования-к-оборудованию)
4. [Подготовка инфраструктуры](#подготовка-инфраструктуры)
5. [Установка Ceph](#установка-ceph)
6. [Конфигурация и оптимизация](#конфигурация-и-оптимизация)
7. [Масштабирование кластера](#масштабирование-кластера)
8. [Мониторинг и алертинг](#мониторинг-и-алертинг)
9. [Эксплуатация и обслуживание](#эксплуатация-и-обслуживание)
10. [Troubleshooting](#troubleshooting)


---
## Backup стратегии для Ceph

### Зачем нужен backup если есть репликация?

⚠️ **КРИТИЧНО**: Репликация ≠ Backup!

**Репликация защищает от:**
- Отказа дисков
- Отказа узлов
- Отказа стоек

**Репликация НЕ защищает от:**
- Случайного удаления данных
- Повреждения данных приложением
- Вирусов/ransomware
- Ошибок администратора
- Катастрофы дата-центра

### Архитектура backup для Ceph

```
┌─────────────────────────────────────────────────────────────┐
│                   Production Ceph Cluster                    │
│                        (DC1 - Primary)                        │
│    ┌──────────┬──────────┬──────────┐                       │
│    │  node1   │  node2   │  node3   │                       │
│    │ 10 OSD   │ 10 OSD   │ 10 OSD   │                       │
│    └──────────┴──────────┴──────────┘                       │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ RBD Snapshots / RGW Sync
                            │ rados export / incremental backup
                            ↓
┌─────────────────────────────────────────────────────────────┐
│                    Backup Ceph Cluster                       │
│                     (DC2 - Secondary)                        │
│    ┌──────────┬──────────┬──────────┐                       │
│    │  backup1 │  backup2 │  backup3 │                       │
│    │ 10 OSD   │ 10 OSD   │ 10 OSD   │                       │
│    └──────────┴──────────┴──────────┘                       │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ Long-term archive
                            ↓
                    ┌──────────────┐
                    │ Tape / S3    │
                    │ Glacier      │
                    └──────────────┘
```

### Стратегия 1: RBD snapshots (для блочных устройств)

**Use case:** Виртуальные машины, Kubernetes PV

#### Создание и управление снапшотами

```bash
# ============================================
# Ручные snapshot
# ============================================

# Создание snapshot
rbd snap create mypool/myimage@snap1

# Список snapshots
rbd snap ls mypool/myimage

# Защита snapshot от удаления
rbd snap protect mypool/myimage@snap1

# Откат к snapshot
rbd snap rollback mypool/myimage@snap1

# Удаление snapshot
rbd snap unprotect mypool/myimage@snap1
rbd snap rm mypool/myimage@snap1

# Удаление всех snapshots
rbd snap purge mypool/myimage
```

#### Автоматизация через скрипты

```bash
#!/bin/bash
# /usr/local/bin/rbd-snapshot-backup.sh

POOL="rbd-pool"
RETENTION_DAYS=7

# Функция создания snapshot
create_snapshot() {
    local image=$1
    local snap_name="auto_$(date +%Y%m%d_%H%M%S)"
    
    echo "Creating snapshot: $image@$snap_name"
    rbd snap create $POOL/$image@$snap_name
    
    # Защита от случайного удаления
    rbd snap protect $POOL/$image@$snap_name
}

# Функция удаления старых snapshots
cleanup_old_snapshots() {
    local image=$1
    local retention_seconds=$((RETENTION_DAYS * 86400))
    local current_time=$(date +%s)
    
    rbd snap ls $POOL/$image --format json | jq -r '.[].name' | while read snap; do
        # Пропустить не-автоматические snapshots
        [[ ! "$snap" =~ ^auto_ ]] && continue
        
        # Получить время создания
        snap_time=$(rbd snap ls $POOL/$image --format json | jq -r ".[] | select(.name==\"$snap\") | .timestamp")
        snap_epoch=$(date -d "$snap_time" +%s)
        
        # Проверить возраст
        age=$((current_time - snap_epoch))
        
        if [ $age -gt $retention_seconds ]; then
            echo "Removing old snapshot: $image@$snap (age: $((age / 86400)) days)"
            rbd snap unprotect $POOL/$image@$snap 2>/dev/null
            rbd snap rm $POOL/$image@$snap
        fi
    done
}

# Обработка всех образов
rbd ls $POOL | while read image; do
    echo "Processing image: $image"
    create_snapshot $image
    cleanup_old_snapshots $image
done

echo "Backup completed: $(date)"
```

```bash
# Добавить в cron (каждый день в 02:00)
cat > /etc/cron.d/rbd-snapshot-backup <<EOF
0 2 * * * root /usr/local/bin/rbd-snapshot-backup.sh >> /var/log/rbd-backup.log 2>&1
EOF
```

#### Экспорт snapshots в другой кластер

```bash
# ============================================
# Полный экспорт образа
# ============================================

# На source кластере: экспорт
rbd export mypool/myimage - | \
  ssh backup-cluster 'rbd import - backup-pool/myimage'

# ============================================
# Инкрементальный экспорт (эффективнее!)
# ============================================

# Первый раз: полный экспорт
rbd export-diff mypool/myimage@snap1 - | \
  ssh backup-cluster 'rbd import-diff - backup-pool/myimage'

# Последующие: только изменения
rbd export-diff mypool/myimage@snap2 --from-snap snap1 - | \
  ssh backup-cluster 'rbd import-diff - backup-pool/myimage'

# ============================================
# Автоматизация инкрементального backup
# ============================================

#!/bin/bash
# /usr/local/bin/rbd-incremental-backup.sh

SOURCE_POOL="rbd-pool"
DEST_POOL="backup-pool"
DEST_CLUSTER="backup-cluster"

# Для каждого образа
rbd ls $SOURCE_POOL | while read image; do
    echo "Backing up: $image"
    
    # Создать новый snapshot
    SNAP_NAME="backup_$(date +%Y%m%d_%H%M%S)"
    rbd snap create $SOURCE_POOL/$image@$SNAP_NAME
    
    # Найти предыдущий snapshot
    PREV_SNAP=$(rbd snap ls $SOURCE_POOL/$image --format json | \
                jq -r '.[].name | select(startswith("backup_"))' | \
                tail -2 | head -1)
    
    if [ -z "$PREV_SNAP" ]; then
        # Первый backup: полный экспорт
        echo "Full export of $image@$SNAP_NAME"
        rbd export-diff $SOURCE_POOL/$image@$SNAP_NAME - | \
          ssh $DEST_CLUSTER "rbd import-diff - $DEST_POOL/$image"
    else
        # Инкрементальный export
        echo "Incremental export of $image from $PREV_SNAP to $SNAP_NAME"
        rbd export-diff $SOURCE_POOL/$image@$SNAP_NAME --from-snap $PREV_SNAP - | \
          ssh $DEST_CLUSTER "rbd import-diff - $DEST_POOL/$image"
    fi
done
```

### Стратегия 2: RADOS export/import (для пулов)

**Use case:** Полный backup пула, миграция данных

```bash
# ============================================
# Полный экспорт пула
# ============================================

# Экспорт всех объектов пула
rados export mypool /backup/mypool-$(date +%Y%m%d).tar

# С сжатием
rados export mypool - | gzip > /backup/mypool-$(date +%Y%m%d).tar.gz

# Экспорт в другой кластер через SSH
rados export mypool - | \
  ssh backup-cluster "cat > /backup/mypool-$(date +%Y%m%d).tar"

# ============================================
# Импорт обратно
# ============================================

# Импорт пула
rados import /backup/mypool-20250128.tar mypool

# С сжатием
gunzip -c /backup/mypool-20250128.tar.gz | rados import - mypool

# ============================================
# Backup конкретных объектов
# ============================================

# Список объектов в пуле
rados -p mypool ls

# Экспорт одного объекта
rados -p mypool get myobject /backup/myobject

# Импорт объекта
rados -p mypool put myobject /backup/myobject
```

#### Автоматизация RADOS backup

```bash
#!/bin/bash
# /usr/local/bin/rados-pool-backup.sh

BACKUP_DIR="/backup/ceph-rados"
RETENTION_DAYS=30
POOLS="mypool rgw-data cephfs-data"

mkdir -p $BACKUP_DIR

for POOL in $POOLS; do
    echo "Backing up pool: $POOL"
    
    BACKUP_FILE="$BACKUP_DIR/${POOL}_$(date +%Y%m%d_%H%M%S).tar.gz"
    
    # Экспорт с сжатием
    rados export $POOL - | gzip > $BACKUP_FILE
    
    # Проверка успешности
    if [ $? -eq 0 ]; then
        echo "Successfully backed up $POOL to $BACKUP_FILE"
        
        # Запись метаданных
        echo "Pool: $POOL" > ${BACKUP_FILE}.metadata
        echo "Date: $(date)" >> ${BACKUP_FILE}.metadata
        echo "Objects: $(rados -p $POOL ls | wc -l)" >> ${BACKUP_FILE}.metadata
        echo "Size: $(du -h $BACKUP_FILE | cut -f1)" >> ${BACKUP_FILE}.metadata
    else
        echo "ERROR: Failed to backup $POOL"
    fi
done

# Очистка старых backup
echo "Cleaning up old backups..."
find $BACKUP_DIR -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete
find $BACKUP_DIR -name "*.metadata" -mtime +$RETENTION_DAYS -delete

echo "Backup completed: $(date)"
```

### Стратегия 3: CephFS snapshots

**Use case:** Файловое хранилище, NFS/SMB shares

```bash
# ============================================
# Создание CephFS snapshot
# ============================================

# Snapshots создаются в специальной директории .snap
cd /mnt/cephfs/mydir
mkdir .snap/snapshot-$(date +%Y%m%d)

# Список snapshots
ls -la /mnt/cephfs/mydir/.snap/

# Восстановление из snapshot
cp -a /mnt/cephfs/mydir/.snap/snapshot-20250128/file.txt \
     /mnt/cephfs/mydir/file.txt

# Удаление snapshot
rmdir /mnt/cephfs/mydir/.snap/snapshot-20250128

# ============================================
# Автоматизация CephFS snapshots
# ============================================

#!/bin/bash
# /usr/local/bin/cephfs-snapshot-backup.sh

CEPHFS_MOUNT="/mnt/cephfs"
DIRS_TO_BACKUP="data home projects"
RETENTION_DAYS=14

for DIR in $DIRS_TO_BACKUP; do
    SNAP_DIR="$CEPHFS_MOUNT/$DIR/.snap"
    SNAP_NAME="auto_$(date +%Y%m%d_%H%M%S)"
    
    echo "Creating snapshot: $DIR -> $SNAP_NAME"
    mkdir -p "$SNAP_DIR/$SNAP_NAME"
    
    # Очистка старых snapshots
    find "$SNAP_DIR" -maxdepth 1 -type d -name "auto_*" -mtime +$RETENTION_DAYS | while read snap; do
        echo "Removing old snapshot: $snap"
        rmdir "$snap"
    done
done
```

### Стратегия 4: RadosGW (S3) replication

**Use case:** Объектное хранилище S3

#### Настройка multisite replication

```bash
# ============================================
# Конфигурация multisite (Primary DC)
# ============================================

# 1. Создать realm
radosgw-admin realm create --rgw-realm=production

# 2. Создать zonegroup (multi-site group)
radosgw-admin zonegroup create --rgw-zonegroup=eu \
  --master --default

# 3. Создать zone (DC1 - primary)
radosgw-admin zone create --rgw-zonegroup=eu \
  --rgw-zone=dc1 --master --default

# 4. Создать system user для sync
radosgw-admin user create --uid=sync-user \
  --display-name="Sync User" --system

# Сохранить access_key и secret_key!

# 5. Commit конфигурации
radosgw-admin period update --commit

# 6. Рестарт RGW
systemctl restart ceph-radosgw.target

# ============================================
# Конфигурация multisite (Secondary DC)
# ============================================

# На backup кластере:

# 1. Pull realm конфигурации
radosgw-admin realm pull --url=http://primary-rgw:8080 \
  --access-key=<ACCESS_KEY> --secret=<SECRET_KEY>

# 2. Pull period
radosgw-admin period pull --url=http://primary-rgw:8080 \
  --access-key=<ACCESS_KEY> --secret=<SECRET_KEY>

# 3. Создать secondary zone
radosgw-admin zone create --rgw-zonegroup=eu \
  --rgw-zone=dc2 \
  --access-key=<ACCESS_KEY> --secret=<SECRET_KEY> \
  --endpoints=http://backup-rgw:8080

# 4. Update period
radosgw-admin period update --commit

# 5. Запустить RGW
systemctl start ceph-radosgw.target

# ============================================
# Проверка sync
# ============================================

# На primary
radosgw-admin sync status

# Результат:
#   realm abc123 (production)
#   zonegroup def456 (eu)
#   zone ghi789 (dc1)
#   metadata sync syncing
#   data sync source: 1 shards
#   data sync source: 1 shards syncing
```

#### Мониторинг replication

```bash
# Статус синхронизации
radosgw-admin sync status

# Детальная информация
radosgw-admin sync info

# Ошибки синхронизации
radosgw-admin sync error list

# Повторная попытка для failed объектов
radosgw-admin sync error trim --shard-id=0
```

### Стратегия 5: Restic backup (универсальное решение)

**Use case:** Гибкий backup с дедупликацией, шифрованием

```bash
# ============================================
# Установка Restic
# ============================================

wget https://github.com/restic/restic/releases/download/v0.16.0/restic_0.16.0_linux_amd64.bz2
bunzip2 restic_0.16.0_linux_amd64.bz2
chmod +x restic_0.16.0_linux_amd64
mv restic_0.16.0_linux_amd64 /usr/local/bin/restic

# ============================================
# Инициализация репозитория
# ============================================

# S3 backend (можно использовать RGW)
export RESTIC_REPOSITORY=s3:http://rgw.backup.local/restic-backup
export RESTIC_PASSWORD=<strong-password>
export AWS_ACCESS_KEY_ID=<access-key>
export AWS_SECRET_ACCESS_KEY=<secret-key>

restic init

# ============================================
# Backup RBD образов
# ============================================

#!/bin/bash
# /usr/local/bin/restic-rbd-backup.sh

export RESTIC_REPOSITORY=s3:http://rgw.backup.local/restic-backup
export RESTIC_PASSWORD_FILE=/root/.restic-password

POOL="rbd-pool"
MOUNT_POINT="/mnt/rbd-backup"

rbd ls $POOL | while read image; do
    echo "Backing up RBD: $image"
    
    # Map RBD device
    rbd map $POOL/$image
    DEVICE=$(rbd showmapped | grep "$image" | awk '{print $5}')
    
    # Mount (если есть FS)
    mkdir -p $MOUNT_POINT/$image
    mount $DEVICE $MOUNT_POINT/$image
    
    # Backup через Restic
    restic backup $MOUNT_POINT/$image \
      --tag rbd \
      --tag pool:$POOL \
      --tag image:$image
    
    # Cleanup
    umount $MOUNT_POINT/$image
    rbd unmap $DEVICE
done

# Очистка старых snapshots (retention policy)
restic forget \
  --keep-daily 7 \
  --keep-weekly 4 \
  --keep-monthly 12 \
  --prune

echo "Backup completed: $(date)"
```

### Стратегия 6: rclone для миграции между кластерами

**Use case:** Копирование больших объемов данных между S3/RGW

```bash
# ============================================
# Установка rclone
# ============================================

curl https://rclone.org/install.sh | bash

# ============================================
# Конфигурация
# ============================================

cat > ~/.rclone.conf <<EOF
[source-ceph]
type = s3
provider = Ceph
env_auth = false
access_key_id = <SOURCE_ACCESS_KEY>
secret_access_key = <SOURCE_SECRET_KEY>
endpoint = http://source-rgw:8080
region = dc1

[backup-ceph]
type = s3
provider = Ceph
env_auth = false
access_key_id = <BACKUP_ACCESS_KEY>
secret_access_key = <BACKUP_SECRET_KEY>
endpoint = http://backup-rgw:8080
region = dc2
EOF

# ============================================
# Синхронизация bucket
# ============================================

# Полная синхронизация
rclone sync source-ceph:mybucket backup-ceph:mybucket \
  --progress \
  --transfers 32 \
  --checkers 16

# Только новые/измененные файлы
rclone sync source-ceph:mybucket backup-ceph:mybucket \
  --checksum \
  --progress

# Инкрементальный с логированием
rclone sync source-ceph:mybucket backup-ceph:mybucket \
  --log-file=/var/log/rclone-sync.log \
  --log-level INFO \
  --max-age 24h  # только файлы измененные за последние 24ч

# ============================================
# Автоматизация через cron
# ============================================

#!/bin/bash
# /usr/local/bin/rclone-backup.sh

BUCKETS="data backups archives"
LOG_FILE="/var/log/rclone-backup.log"

for BUCKET in $BUCKETS; do
    echo "[$(date)] Starting sync: $BUCKET" >> $LOG_FILE
    
    rclone sync source-ceph:$BUCKET backup-ceph:$BUCKET \
      --checksum \
      --transfers 32 \
      --log-file=$LOG_FILE \
      --log-level INFO
    
    if [ $? -eq 0 ]; then
        echo "[$(date)] Successfully synced: $BUCKET" >> $LOG_FILE
    else
        echo "[$(date)] ERROR syncing: $BUCKET" >> $LOG_FILE
        # Отправить алерт
        echo "rclone sync failed for $BUCKET" | mail -s "Backup Alert" admin@example.com
    fi
done
```

### Backup retention policy (3-2-1 правило)

**Правило 3-2-1:**
- **3** копии данных
- **2** разных типа носителей
- **1** копия offsite

```
┌────────────────────────────────────────────────────┐
│ Production Ceph (DC1)                              │
│ - Копия 1: Production data (replica=3)            │
└────────────────────────────────────────────────────┘
                     │
                     ├──────────────────────┐
                     │                      │
                     ↓                      ↓
┌─────────────────────────────┐  ┌──────────────────────┐
│ Backup Ceph (DC2)           │  │ Cold Storage         │
│ - Копия 2: Daily snapshots  │  │ - Копия 3: Archives  │
│ - Retention: 30 days        │  │ - Tape/S3 Glacier    │
└─────────────────────────────┘  │ - Retention: 7 years │
                                  └──────────────────────┘
```

**Пример retention политики:**

```bash
#!/bin/bash
# /usr/local/bin/backup-retention.sh

# ============================================
# Retention rules
# ============================================

# Ежедневные backup: храним 7 дней
DAILY_RETENTION=7

# Еженедельные backup: храним 4 недели
WEEKLY_RETENTION=28

# Ежемесячные backup: храним 12 месяцев
MONTHLY_RETENTION=365

# Ежегодные backup: храним 7 лет
YEARLY_RETENTION=2555

# ============================================
# Cleanup старых backup
# ============================================

BACKUP_DIR="/backup/ceph"

# Daily backups
find $BACKUP_DIR/daily -type f -mtime +$DAILY_RETENTION -delete

# Weekly backups (каждое воскресенье)
# Удаляем weekly backup старше 4 недель
find $BACKUP_DIR/weekly -type f -mtime +$WEEKLY_RETENTION -delete

# Monthly backups (1 числа месяца)
# Удаляем monthly backup старше 12 месяцев
find $BACKUP_DIR/monthly -type f -mtime +$MONTHLY_RETENTION -delete

# Yearly backups (1 января)
# Удаляем yearly backup старше 7 лет
find $BACKUP_DIR/yearly -type f -mtime +$YEARLY_RETENTION -delete

echo "Retention cleanup completed: $(date)"
```

### Тестирование backup (КРИТИЧНО!)

⚠️ **Backup без тестирования = нет backup!**

```bash
#!/bin/bash
# /usr/local/bin/test-backup-restore.sh

# ============================================
# Тест восстановления RBD
# ============================================

TEST_POOL="test-restore"
BACKUP_FILE="/backup/mypool/myimage-20250128.tar.gz"

echo "Creating test pool..."
ceph osd pool create $TEST_POOL 32 32

echo "Restoring from backup..."
gunzip -c $BACKUP_FILE | rbd import - $TEST_POOL/test-restore-image

echo "Checking restored image..."
rbd info $TEST_POOL/test-restore-image

# Map и проверка данных
rbd map $TEST_POOL/test-restore-image
DEVICE=$(rbd showmapped | grep test-restore-image | awk '{print $5}')

# Проверка FS
fsck -n $DEVICE

# Mount и проверка файлов
mkdir -p /mnt/test-restore
mount -o ro $DEVICE /mnt/test-restore
ls -laR /mnt/test-restore > /tmp/restore-test-files.txt

# Cleanup
umount /mnt/test-restore
rbd unmap $DEVICE
rbd rm $TEST_POOL/test-restore-image
ceph osd pool delete $TEST_POOL $TEST_POOL --yes-i-really-really-mean-it

echo "Restore test completed successfully!"
```

**Ежемесячный restore test (автоматизация):**

```bash
# Добавить в cron (1 числа каждого месяца)
cat > /etc/cron.d/backup-restore-test <<EOF
0 4 1 * * root /usr/local/bin/test-backup-restore.sh >> /var/log/backup-restore-test.log 2>&1
EOF
```

---

## Multi-Datacenter конфигурации

### Архитектура Ceph Multi-Site

```
┌────────────────────────────────────────────────────────────┐
│                  Datacenter 1 (Primary)                     │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Ceph Cluster DC1 (zone: dc1)                 │  │
│  │  ┌────────┬────────┬────────┐                        │  │
│  │  │ node1  │ node2  │ node3  │                        │  │
│  │  │ 10 OSD │ 10 OSD │ 10 OSD │                        │  │
│  │  └────────┴────────┴────────┘                        │  │
│  │  RGW (Active-Active for reads/writes)                │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
                            │
                            │ WAN Link (VPN/Direct Connect)
                            │ Metadata + Data sync
                            │ Latency: < 100ms рекомендуется
                            ↓
┌────────────────────────────────────────────────────────────┐
│                Datacenter 2 (Secondary/DR)                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Ceph Cluster DC2 (zone: dc2)                 │  │
│  │  ┌────────┬────────┬────────┐                        │  │
│  │  │ node4  │ node5  │ node6  │                        │  │
│  │  │ 10 OSD │ 10 OSD │ 10 OSD │                        │  │
│  │  └────────┴────────┴────────┘                        │  │
│  │  RGW (Active-Active for reads, sync for writes)      │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

### Вариант 1: Stretched Cluster (Single Ceph Cluster)

**Требования:**
- Latency < 5ms между DC
- Dedicated high-bandwidth link (10+ Gbit/s)
- 3+ датацентра для MON кворума

```
Topology:
┌─────────────┬─────────────┬─────────────┐
│    DC1      │    DC2      │    DC3      │
│  (Primary)  │ (Secondary) │ (Arbitrator)│
├─────────────┼─────────────┼─────────────┤
│ 3 MON       │ 3 MON       │ 1 MON (tiebreaker)│
│ 2 MGR       │ 2 MGR       │ -           │
│ 15 OSD      │ 15 OSD      │ -           │
└─────────────┴─────────────┴─────────────┘

Failure domain: datacenter
Replica: 3 (по 1 в каждом DC)
```

#### Настройка Stretched Cluster

```bash
# ============================================
# 1. Создание CRUSH buckets для DC
# ============================================

ceph osd crush add-bucket dc1 datacenter
ceph osd crush add-bucket dc2 datacenter
ceph osd crush add-bucket dc3 datacenter

# Создать стойки внутри DC
ceph osd crush add-bucket dc1-rack1 rack
ceph osd crush add-bucket dc1-rack2 rack
ceph osd crush add-bucket dc2-rack1 rack
ceph osd crush add-bucket dc2-rack2 rack

# Связать иерархию
ceph osd crush move dc1-rack1 datacenter=dc1
ceph osd crush move dc1-rack2 datacenter=dc1
ceph osd crush move dc2-rack1 datacenter=dc2
ceph osd crush move dc2-rack2 datacenter=dc2

ceph osd crush move dc1 root=default
ceph osd crush move dc2 root=default
ceph osd crush move dc3 root=default

# ============================================
# 2. Размещение хостов
# ============================================

# DC1 узлы
ceph osd crush move node1 rack=dc1-rack1
ceph osd crush move node2 rack=dc1-rack2

# DC2 узлы
ceph osd crush move node3 rack=dc2-rack1
ceph osd crush move node4 rack=dc2-rack2

# ============================================
# 3. Создание CRUSH rule для stretched cluster
# ============================================

ceph osd crush rule create-replicated stretched-rule \
  default datacenter host

# ============================================
# 4. Применение правила к пулам
# ============================================

# Для существующих пулов
ceph osd pool set mypool crush_rule stretched-rule

# Для новых пулов
ceph osd pool create stretched-pool 128 128 replicated stretched-rule

# Настройка размера репликации
ceph osd pool set stretched-pool size 3
ceph osd pool set stretched-pool min_size 2

# ============================================
# 5. Настройка MON для stretched cluster
# ============================================

# Размещение MON (нечетное число для кворума)
# 3 MON в DC1, 3 MON в DC2, 1 MON в DC3
ceph orch apply mon --placement="7 dc1-node1 dc1-node2 dc1-node3 dc2-node1 dc2-node2 dc2-node3 dc3-node1"

# Настройка mon weights (для балансировки кворума)
ceph mon set_weight dc1-mon1 3
ceph mon set_weight dc1-mon2 3
ceph mon set_weight dc1-mon3 3
ceph mon set_weight dc2-mon1 3
ceph mon set_weight dc2-mon2 3
ceph mon set_weight dc2-mon3 3
ceph mon set_weight dc3-mon1 1  # tiebreaker

# ============================================
# 6. Настройка сети
# ============================================

# Public network (клиенты в каждом DC)
ceph config set mon public_network 10.0.1.0/24,172.16.1.0/24,192.168.1.0/24

# Cluster network (WAN link между DC)
ceph config set mon cluster_network 10.0.2.0/24,172.16.2.0/24

# Увеличить таймауты для WAN
ceph config set mon mon_osd_down_out_interval 7200  # 2 часа
ceph config set osd osd_heartbeat_grace 30
ceph config set osd osd_heartbeat_interval 10
```

**Проверка конфигурации:**

```bash
# Проверить топологию
ceph osd crush tree

# Результат:
# ID   CLASS  WEIGHT   TYPE NAME              
# -1          90.00    root default           
# -2          30.00        datacenter dc1     
# -3          15.00            rack dc1-rack1 
# -4          15.00                host node1 
# -5          15.00            rack dc1-rack2 
# -6          15.00                host node2 
# -7          30.00        datacenter dc2     
# -8          15.00            rack dc2-rack1 
# -9          15.00                host node3 

# Проверить размещение данных
ceph pg dump | grep -E 'pgid|acting'

# Проверить MON кворум
ceph mon stat
ceph mon dump
```

**Преимущества Stretched Cluster:**
- ✅ Прозрачность для клиентов (один кластер)
- ✅ Автоматический failover
- ✅ Синхронная репликация
- ✅ Локальное чтение в каждом DC

**Недостатки:**
- ❌ Требует низкую latency (< 5ms)
- ❌ Требует 3 датацентра для кворума
- ❌ Высокие требования к каналу связи
- ❌ Сложнее в обслуживании

### Вариант 2: Multi-Site через RGW (Async Replication)

**Требования:**
- Любая latency (работает через интернет)
- Только для S3/Swift объектов
- Независимые кластеры

```
Architecture:
┌─────────────────────────────────┐
│   Datacenter 1 (Primary)        │
│   Ceph Cluster 1                │
│   RGW Zone: dc1                 │
│   - Accepts writes              │
│   - Syncs to DC2                │
└─────────────────────────────────┘
            │
            │ Async metadata + data sync
            │ Can tolerate high latency
            ↓
┌─────────────────────────────────┐
│   Datacenter 2 (Secondary)      │
│   Ceph Cluster 2                │
│   RGW Zone: dc2                 │
│   - Accepts reads               │
│   - Can accept writes (optional)│
└─────────────────────────────────┘
```

#### Полная настройка RGW Multi-Site

**На Primary DC (DC1):**

```bash
# ============================================
# Шаг 1: Создать Realm (глобальный контейнер)
# ============================================

radosgw-admin realm create \
  --rgw-realm=production \
  --default

# ============================================
# Шаг 2: Создать ZoneGroup (группа зон)
# ============================================

radosgw-admin zonegroup create \
  --rgw-zonegroup=global \
  --master \
  --default \
  --endpoints=http://dc1-rgw.example.com:8080

# ============================================
# Шаг 3: Создать Zone (DC1 - primary/master)
# ============================================

radosgw-admin zone create \
  --rgw-zonegroup=global \
  --rgw-zone=dc1 \
  --master \
  --default \
  --endpoints=http://dc1-rgw.example.com:8080

# ============================================
# Шаг 4: Создать System User для sync
# ============================================

radosgw-admin user create \
  --uid=sync-user \
  --display-name="Synchronization User" \
  --system \
  --access-key=SYNC_ACCESS_KEY \
  --secret=SYNC_SECRET_KEY

# ВАЖНО: Сохранить access_key и secret_key!

# ============================================
# Шаг 5: Добавить system user в zone
# ============================================

radosgw-admin zone modify \
  --rgw-zone=dc1 \
  --access-key=SYNC_ACCESS_KEY \
  --secret=SYNC_SECRET_KEY

# ============================================
# Шаг 6: Commit конфигурации
# ============================================

radosgw-admin period update --commit

# ============================================
# Шаг 7: Перезапустить RGW
# ============================================

systemctl restart ceph-radosgw.target

# Проверка
radosgw-admin period get
radosgw-admin sync status
```

**На Secondary DC (DC2):**

```bash
# ============================================
# Шаг 1: Pull realm конфигурации с Primary
# ============================================

radosgw-admin realm pull \
  --url=http://dc1-rgw.example.com:8080 \
  --access-key=SYNC_ACCESS_KEY \
  --secret=SYNC_SECRET_KEY \
  --default

# ============================================
# Шаг 2: Pull period
# ============================================

radosgw-admin period pull \
  --url=http://dc1-rgw.example.com:8080 \
  --access-key=SYNC_ACCESS_KEY \
  --secret=SYNC_SECRET_KEY

# ============================================
# Шаг 3: Создать Secondary Zone
# ============================================

radosgw-admin zone create \
  --rgw-zonegroup=global \
  --rgw-zone=dc2 \
  --endpoints=http://dc2-rgw.example.com:8080 \
  --access-key=SYNC_ACCESS_KEY \
  --secret=SYNC_SECRET_KEY

# ============================================
# Шаг 4: Delete default zone (если была)
# ============================================

radosgw-admin zone delete --rgw-zone=default
radosgw-admin zonegroup delete --rgw-zonegroup=default
radosgw-admin period update --commit

# ============================================
# Шаг 5: Update period и restart
# ============================================

radosgw-admin period update --commit
systemctl restart ceph-radosgw.target

# ============================================
# Проверка
# ============================================

radosgw-admin sync status

# Результат:
#   realm abc123 (production)
#       zonegroup def456 (global)
#           zone 789ghi (dc2)
#   metadata sync syncing
#       full sync: 0/10 shards
#       incremental sync: 10/10 shards
#   data sync source: dc1
#       syncing
#       full sync: 0/128 shards
#       incremental sync: 128/128 shards
```

#### Настройка Active-Active (оба DC принимают writes)

```bash
# На Primary DC:
radosgw-admin zonegroup modify \
  --rgw-zonegroup=global \
  --endpoints=http://dc1-rgw.example.com:8080,http://dc2-rgw.example.com:8080

radosgw-admin period update --commit

# На Secondary DC:
radosgw-admin zone modify \
  --rgw-zone=dc2 \
  --read-only=false

radosgw-admin period update --commit
systemctl restart ceph-radosgw.target
```

#### Мониторинг sync

```bash
# ============================================
# Основные команды мониторинга
# ============================================

# Общий статус sync
radosgw-admin sync status

# Детальная информация
radosgw-admin sync info

# Метрики sync
radosgw-admin sync counters

# Проблемы с sync
radosgw-admin sync error list

# Повторить failed sync
radosgw-admin sync error trim --shard-id=0

# ============================================
# Автоматизация мониторинга
# ============================================

#!/bin/bash
# /usr/local/bin/monitor-rgw-sync.sh

ALERT_EMAIL="admin@example.com"

# Проверка статуса sync
SYNC_STATUS=$(radosgw-admin sync status 2>&1)

# Проверка на ошибки
if echo "$SYNC_STATUS" | grep -qi "ERROR\|failed\|behind"; then
    echo "RGW Sync Issue Detected!"
    echo "$SYNC_STATUS"
    
    # Детальная информация
    radosgw-admin sync error list
    
    # Отправить алерт
    echo "$SYNC_STATUS" | mail -s "RGW Sync Alert - DC2" $ALERT_EMAIL
fi

# Проверка отставания (lag)
META_LAG=$(radosgw-admin sync status | grep "metadata sync" -A 5 | grep "behind" || echo "0")
DATA_LAG=$(radosgw-admin sync status | grep "data sync" -A 5 | grep "behind" || echo "0")

# Если отставание > 1000 объектов - алерт
if [ ! -z "$META_LAG" ] || [ ! -z "$DATA_LAG" ]; then
    echo "RGW Sync Lag Detected: Meta=$META_LAG, Data=$DATA_LAG" | \
      mail -s "RGW Sync Lag Alert" $ALERT_EMAIL
fi
```

```bash
# Добавить в cron (каждые 15 минут)
cat > /etc/cron.d/monitor-rgw-sync <<EOF
*/15 * * * * root /usr/local/bin/monitor-rgw-sync.sh >> /var/log/rgw-sync-monitor.log 2>&1
EOF
```

### Вариант 3: RBD Mirroring (для блочных устройств)

**Use case:** Асинхронная репликация RBD образов между кластерами

```
Architecture:
┌─────────────────────────────────┐
│   Datacenter 1 (Primary)        │
│   Ceph Cluster 1                │
│   Pool: rbd-mirror              │
│   - Primary images              │
│   - Journal для changes         │
└─────────────────────────────────┘
            │
            │ Async journal replay
            │ RPO: несколько секунд
            ↓
┌─────────────────────────────────┐
│   Datacenter 2 (DR)             │
│   Ceph Cluster 2                │
│   Pool: rbd-mirror              │
│   - Secondary (read-only)       │
│   - Failover при катастрофе     │
└─────────────────────────────────┘
```

#### Настройка RBD Mirroring

**На обоих кластерах:**

```bash
# ============================================
# Шаг 1: Включить mirroring на пуле
# ============================================

# Primary кластер
ceph osd pool create rbd-mirror 128 128
rbd pool init rbd-mirror

# Enable mirroring (image mode - per-image mirroring)
rbd mirror pool enable rbd-mirror image

# ИЛИ pool mode (все образы в пуле)
rbd mirror pool enable rbd-mirror pool

# Secondary кластер (то же самое)
ceph osd pool create rbd-mirror 128 128
rbd pool init rbd-mirror
rbd mirror pool enable rbd-mirror image

# ============================================
# Шаг 2: Создать bootstrap token на Primary
# ============================================

# На Primary кластере
rbd mirror pool peer bootstrap create \
  --site-name dc1 rbd-mirror > /tmp/bootstrap-token-dc1

# Токен будет выглядеть так:
# eyJmc2lkIjoiYWJjMTIz...

# ============================================
# Шаг 3: Import token на Secondary
# ============================================

# На Secondary кластере
rbd mirror pool peer bootstrap import \
  --site-name dc2 \
  --direction rx-tx \
  rbd-mirror /tmp/bootstrap-token-dc1

# ============================================
# Шаг 4: Включить mirroring на конкретных образах
# ============================================

# На Primary кластере
# Создать образ
rbd create rbd-mirror/myimage --size 100G

# Включить journaling (необходимо для mirroring!)
rbd feature enable rbd-mirror/myimage exclusive-lock journaling

# Включить mirroring для образа
rbd mirror image enable rbd-mirror/myimage

# ============================================
# Проверка статуса
# ============================================

# Статус mirroring пула
rbd mirror pool status rbd-mirror

# Статус конкретного образа
rbd mirror image status rbd-mirror/myimage

# Результат:
# myimage:
#   global_id:   abc123-def456
#   state:       up+replaying
#   description: replaying, master_position=[object_number=1234, tag_tid=1, entry_tid=5678], 
#                mirror_position=[object_number=1230, tag_tid=1, entry_tid=5670], 
#                entries_behind_master=8
#   last_update: 2025-01-28 10:15:30
```

#### RBD Mirror Daemon

```bash
# ============================================
# Установка и запуск rbd-mirror демона
# ============================================

# На Secondary кластере (или на обоих для bidirectional)

# Через cephadm
ceph orch apply rbd-mirror --placement="dc2-node1 dc2-node2"

# Проверка
ceph orch ps --daemon-type rbd-mirror

# Статус демона
rbd mirror pool status --verbose rbd-mirror

# ============================================
# Failover на Secondary DC (в случае disaster)
# ============================================

# На Secondary кластере
# Промотировать образ в primary
rbd mirror image promote rbd-mirror/myimage

# Образ теперь доступен для записи на Secondary
rbd bench --io-type write rbd-mirror/myimage --io-size 4K --io-total 1G

# ============================================
# Failback на Primary DC (после восстановления)
# ============================================

# На Secondary (теперь primary)
rbd mirror image demote rbd-mirror/myimage

# На Primary (было secondary)
rbd mirror image promote rbd-mirror/myimage

# Mirroring продолжится в обратном направлении
```

#### Мониторинг RBD Mirroring

```bash
#!/bin/bash
# /usr/local/bin/monitor-rbd-mirror.sh

POOL="rbd-mirror"
ALERT_EMAIL="admin@example.com"

# Проверка статуса пула
POOL_STATUS=$(rbd mirror pool status $POOL 2>&1)

# Проверка демона
DAEMON_STATUS=$(ceph orch ps --daemon-type rbd-mirror --format json | jq -r '.[].status_desc')

if echo "$DAEMON_STATUS" | grep -qi "error\|stopped"; then
    echo "RBD Mirror Daemon Issue!" | mail -s "RBD Mirror Alert" $ALERT_EMAIL
fi

# Проверка каждого образа
rbd ls $POOL | while read image; do
    # Проверить включен ли mirroring
    MIRROR_ENABLED=$(rbd info $POOL/$image | grep "mirroring state" | grep -q "enabled" && echo "yes" || echo "no")
    
    if [ "$MIRROR_ENABLED" = "yes" ]; then
        # Проверить статус
        STATUS=$(rbd mirror image status $POOL/$image)
        
        # Проверить lag
        LAG=$(echo "$STATUS" | grep "entries_behind_master" | awk -F= '{print $2}' | tr -d ' ')
        
        if [ ! -z "$LAG" ] && [ "$LAG" -gt 1000 ]; then
            echo "RBD Mirror Lag for $image: $LAG entries behind" | \
              mail -s "RBD Mirror Lag Alert" $ALERT_EMAIL
        fi
        
        # Проверить состояние
        if echo "$STATUS" | grep -qi "error\|unknown"; then
            echo "RBD Mirror Error for $image:" | \
              mail -s "RBD Mirror Error Alert" $ALERT_EMAIL
            echo "$STATUS" | mail -s "RBD Mirror Error Alert" $ALERT_EMAIL
        fi
    fi
done
```

### Вариант 4: CephFS Multi-DC (Experimental)

⚠️ **Экспериментально! Не рекомендуется для production**

CephFS не имеет встроенной multi-DC репликации. Альтернативы:

**Решение 1: Rsync через cron**
```bash
# Простая односторонняя синхронизация
rsync -avz --delete /mnt/cephfs-dc1/ backup-server:/backup/cephfs/
```

**Решение 2: Lsyncd (real-time sync)**
```bash
# Установка
apt install lsyncd

# Конфигурация /etc/lsyncd/lsyncd.conf.lua
settings {
    logfile = "/var/log/lsyncd/lsyncd.log",
    statusFile = "/var/log/lsyncd/lsyncd.status"
}

sync {
    default.rsync,
    source = "/mnt/cephfs-dc1",
    target = "backup-server:/mnt/cephfs-dc2",
    delay = 5,
    rsync = {
        archive = true,
        compress = true,
        whole_file = false
    }
}
```

### Сравнение Multi-DC решений

| Критерий | Stretched Cluster | RGW Multi-Site | RBD Mirroring | Rsync/Backup |
|----------|------------------|----------------|---------------|--------------|
| **Latency требования** | < 5ms | Любая | Любая | Любая |
| **Тип данных** | Все | S3/Swift только | RBD только | Все |
| **Синхронность** | Синхронная | Асинхронная | Асинхронная | Асинхронная |
| **RPO** | 0 | Секунды | Секунды | Минуты-часы |
| **RTO** | Мгновенно | Минуты | Минуты | Часы |
| **Active-Active** | Да | Да | Нет | Нет |
| **Сложность** | Высокая | Средняя | Средняя | Низкая |
| **Стоимость WAN** | Высокая | Средняя | Средняя | Низкая |

### Best Practices для Multi-DC

1. **Выбор решения:**
   - Latency < 5ms + все типы данных → Stretched Cluster
   - S3 объекты → RGW Multi-Site
   - VM диски, K8s PV → RBD Mirroring
   - Файлы, архивы → Rsync + Backup

2. **Сетевые требования:**
   - Dedicated канал между DC
   - QoS для Ceph трафика
   - Шифрование (VPN/IPSec)
   - Мониторинг latency и packet loss

3. **Тестирование:**
   - Регулярные failover тесты
   - Проверка RPO/RTO
   - Тест отказа WAN канала
   - Проверка split-brain сценариев

4. **Мониторинг:**
   - Latency между DC (< 5ms для stretched)
   - Sync lag (RGW, RBD mirror)
   - Bandwidth утилизация
   - Алерты на sync errors

5. **Документация:**
   - Процедура failover
   - Процедура failback
   - Контакты ISP/NOC
   - Runbook для disaster scenarios

---

## Disaster Recovery Scenarios

### Сценарий 1: Полная потеря Primary DC

**Дано:**
- Stretched Cluster или RGW Multi-Site
- Primary DC полностью недоступен
- Secondary DC работает

**Действия:**

```bash
# ============================================
# Для Stretched Cluster
# ============================================

# На Secondary DC:

# 1. Проверить состояние
ceph -s
# Будет HEALTH_WARN (потеря кворума MON)

# 2. Force создать новый MON кворум (КРИТИЧНО!)
# На одном из выживших MON:
systemctl stop ceph-mon.target

ceph-mon -i dc2-mon1 --extract-monmap /tmp/monmap
monmaptool /tmp/monmap --rm dc1-mon1
monmaptool /tmp/monmap --rm dc1-mon2
monmaptool /tmp/monmap --rm dc1-mon3
ceph-mon -i dc2-mon1 --inject-monmap /tmp/monmap

systemctl start ceph-mon.target

# 3. Добавить новые MON взамен потерянных
ceph orch apply mon 5  # новое нечетное число

# 4. Проверить OSD
ceph osd tree
# Пометить потерянные OSD как out
for osd in {0..14}; do  # OSD из DC1
    ceph osd out osd.$osd
done

# 5. Дождаться recovery
ceph -w

# ============================================
# Для RGW Multi-Site
# ============================================

# На Secondary DC:

# 1. Промотировать Secondary zone в Master
radosgw-admin zone modify --rgw-zone=dc2 --master
radosgw-admin period update --commit

# 2. Перенаправить DNS на Secondary RGW
# В DNS: rgw.example.com → dc2-rgw.example.com

# 3. Проверить доступность
aws s3 ls s3://mybucket --endpoint-url=http://dc2-rgw.example.com

# ============================================
# Для RBD Mirroring
# ============================================

# На Secondary DC:

# Промотировать все образы
rbd ls rbd-mirror | while read image; do
    echo "Promoting $image"
    rbd mirror image promote rbd-mirror/$image --force
done

# Проверить статус
rbd mirror pool status rbd-mirror
```

### Сценарий 2: Split-Brain (потеря связи между DC)

**Проблема:** DC изолированы, оба думают что другой упал

**Решение для Stretched Cluster:**

```bash
# Использовать третий DC (arbitrator) для кворума
# MON конфигурация должна быть:
# DC1: 3 MON
# DC2: 3 MON  
# DC3: 1 MON (tie-breaker)

# При split между DC1<->DC2:
# - DC с DC3 получит кворум (3+1=4 из 7)
# - Другой DC потеряет кворум и станет read-only

# Проверка кворума
ceph mon stat
```

### Сценарий 3: Восстановление после disaster

**После восстановления Primary DC:**

```bash
# ============================================
# Возврат на Stretched Cluster
# ============================================

# 1. Восстановить связь между DC
# Проверить сеть
ping dc1-node1 -c 10

# 2. Добавить DC1 узлы обратно
ceph orch host add dc1-node1 10.0.1.11
ceph orch host add dc1-node2 10.0.1.12
ceph orch host add dc1-node3 10.0.1.13

# 3. Восстановить CRUSH topology
ceph osd crush move dc1-node1 rack=dc1-rack1
ceph osd crush move dc1-node2 rack=dc1-rack2

# 4. Добавить OSD
ceph orch apply osd --all-available-devices --hostname=dc1-node1

# 5. Recovery начнется автоматически
ceph -w

# ============================================
# Failback для RGW
# ============================================

# На текущем Master (DC2):
radosgw-admin zone modify --rgw-zone=dc2 --master=false
radosgw-admin period update --commit

# На Primary (DC1):
radosgw-admin zone modify --rgw-zone=dc1 --master=true
radosgw-admin period update --commit

systemctl restart ceph-radosgw.target

# Sync произойдет автоматически
radosgw-admin sync status

# ============================================
# Failback для RBD Mirroring
# ============================================

# На Secondary (DC2) - demote
rbd ls rbd-mirror | while read image; do
    rbd mirror image demote rbd-mirror/$image
done

# На Primary (DC1) - promote
rbd ls rbd-mirror | while read image; do
    rbd mirror image promote rbd-mirror/$image
done
```

**КОНЕЦ ДОКУМЕНТА**# Если диск умирает - заменить
# См. раздел "Замена неисправного диска"
```

**Случай 2: Нехватка ресурсов**
```bash
# Проверить CPU/RAM/Network
top
free -h
iftop -i bond0

# Увеличить лимиты OSD временно
systemctl edit ceph-osd@5.service

[Service]
MemoryMax=12G
CPUQuota=300%

systemctl daemon-reload
systemctl restart ceph-osd@5
```

**Случай 3: Слишком агрессивный recovery**
```bash
# Замедлить recovery
ceph config set osd osd_max_backfills 1
ceph config set osd osd_recovery_max_active 1
ceph config set osd osd_recovery_sleep 0.2
ceph config set osd osd_recovery_op_priority 1
```

### Проблема 4: Заполнение дисков (nearfull/full)

**Симптомы:**
```bash
ceph health detail
# HEALTH_WARN 1 nearfull osd(s)
# osd.15 is near full at 82%
```

**Диагностика:**
```bash
# Проверить заполненность всех OSD
ceph osd df tree | sort -k6 -n

# Проверить распределение по пулам
ceph df detail

# Проверить большие объекты
rados -p mypool ls | while read obj; do
  rados -p mypool stat $obj
done | sort -k2 -n | tail -20
```

**Решения:**

**Краткосрочные:**
```bash
# 1. Увеличить порог nearfull (временно!)
ceph osd set-nearfull-ratio 0.90
ceph osd set-full-ratio 0.95

# 2. Удалить ненужные данные
# RBD snapshots
rbd snap ls mypool/myimage
rbd snap rm mypool/myimage@snapname

# Orphan objects
rados -p mypool ls-inconsistent-obj

# 3. Принудительно ребалансировать
ceph osd reweight osd.15 0.95  # снизить вес переполненного OSD
```

**Долгосрочные:**
```bash
# 1. Добавить диски/узлы
# См. раздел "Масштабирование кластера"

# 2. Включить автоматическую балансировку
ceph balancer on
ceph balancer mode upmap

# 3. Настроить автоочистку
# Для RGW
radosgw-admin gc process

# 4. Компрессия (если не включена)
ceph osd pool set mypool compression_mode aggressive
ceph osd pool set mypool compression_algorithm snappy
```

### Проблема 5: Несогласованность данных (inconsistent PG)

**Симптомы:**
```bash
ceph health detail
# HEALTH_ERR 1 scrub errors
# pg 2.1a has 3 inconsistent objects
```

**Диагностика:**
```bash
# Детали проблемной PG
ceph pg 2.1a query

# Список несогласованных объектов
rados list-inconsistent-obj 2.1a --format=json-pretty

# Детали каждого объекта
rados list-inconsistent-obj 2.1a | jq '.inconsistents[]'
```

**Решения:**

**Автоматический ремонт:**
```bash
# Попытка автоматического исправления
ceph pg repair 2.1a

# Мониторинг ремонта
watch -n 5 'ceph pg 2.1a query | grep state'

# Если ремонт завершился успешно
ceph health detail
# Должно быть HEALTH_OK
```

**Ручной ремонт (если автоматический не помог):**
```bash
# 1. Найти проблемные объекты
rados list-inconsistent-obj 2.1a --format=json-pretty > /tmp/inconsistent.json

# 2. Для каждого объекта посмотреть детали
cat /tmp/inconsistent.json | jq '.inconsistents[0]'

# Пример вывода:
# {
#   "object": {
#     "name": "myobject",
#     "nspace": "",
#     "locator": "",
#     "snap": "head",
#     "version": 123
#   },
#   "errors": ["size_mismatch"],
#   "shards": [
#     {
#       "osd": 5,
#       "size": 4096,
#       "omap_digest": "0xabcd1234",
#       "data_digest": "0xef567890"
#     },
#     {
#       "osd": 12,
#       "size": 4096,
#       "omap_digest": "0xabcd1234",
#       "data_digest": "0xef567890"
#     },
#     {
#       "osd": 8,
#       "size": 8192,  ← РАЗНЫЙ РАЗМЕР!
#       "omap_digest": "0xabcd1234",
#       "data_digest": "0x12345678"  ← РАЗНЫЙ DIGEST!
#     }
#   ]
# }

# 3. Определить правильную копию (большинство голосов)
# В данном случае OSD 5 и 12 согласны, OSD 8 - неправильный

# 4. Удалить неправильную копию
ceph-objectstore-tool --data-path /var/lib/ceph/osd/ceph-8 \
  --pgid 2.1a \
  --op remove \
  myobject

# 5. Запустить ремонт
ceph pg repair 2.1a
```

### Проблема 6: MON не может сформировать кворум

**Симптомы:**
```bash
ceph -s
# Error connecting to cluster: ObjectNotFound

systemctl status ceph-mon@node1
# active but cluster unreachable
```

**Диагностика:**
```bash
# Проверка на каждом MON узле
systemctl status ceph-mon@node1
journalctl -u ceph-mon@node1 -n 100

# Проверка времени
chronyc tracking
# System time offset должен быть < 50ms

# Проверка сети между MON
ping node2
ping node3

# Проверка портов
ss -tlnp | grep 6789
ss -tlnp | grep 3300
```

**Решения:**

**Случай 1: Рассинхронизация времени**
```bash
# На ВСЕХ узлах:
systemctl restart chrony
chronyc makestep

# Подождать 1-2 минуты
systemctl restart ceph-mon.target
```

**Случай 2: Сетевые проблемы**
```bash
# Проверить firewall
systemctl status firewalld
iptables -L -n

# Проверить маршрутизацию
ip route
traceroute node2

# Временно отключить firewall для теста
systemctl stop firewalld

# Если помогло - настроить правила
firewall-cmd --permanent --add-service=ceph-mon
firewall-cmd --reload
```

**Случай 3: Потеря большинства MON**
```bash
# КРАЙНИЙ СЛУЧАЙ!
# Если потеряли > 50% MON (например, 2 из 3)

# На выжившем MON:
systemctl stop ceph-mon.target

# Создать новый monmap
ceph-mon -i node1 --extract-monmap /tmp/monmap

# Посмотреть текущий monmap
monmaptool --print /tmp/monmap

# Удалить мертвые MON
monmaptool /tmp/monmap --rm node2
monmaptool /tmp/monmap --rm node3

# Inject новый monmap
ceph-mon -i node1 --inject-monmap /tmp/monmap

# Запустить MON
systemctl start ceph-mon@node1

# Проверка
ceph -s

# НЕМЕДЛЕННО добавить новые MON!
ceph orch apply mon --placement="3 node1 node4 node5"
```

### Проблема 7: High network latency

**Симптомы:**
```bash
ceph health detail
# HEALTH_WARN slow ops, oldest one blocked for 15 sec

# Ping latency высокая
ping node2
# 64 bytes: time=10ms  ← должно быть < 1ms в локальной сети
```

**Диагностика:**
```bash
# Проверка latency между узлами
for node in node{1..7}; do
  echo "$node:"
  ping -c 10 $node | grep avg
done

# Проверка сетевых интерфейсов
ethtool bond0
ethtool bond1

# Проверка ошибок на интерфейсах
ip -s link show bond0
ip -s link show bond1
# Смотреть на TX/RX errors

# Проверка загрузки сети
iftop -i bond0
iftop -i bond1

# Проверка QoS/traffic shaping
tc qdisc show
```

**Решения:**

**Случай 1: Проблемы с bond/LACP**
```bash
# Проверить состояние bond
cat /proc/net/bonding/bond0

# Если есть проблемы с LACP
# Перезапустить интерфейс
ip link set bond0 down
ip link set bond0 up

# Или полностью пересоздать
systemctl restart networking
```

**Случай 2: Перегрузка сети**
```bash
# Замедлить recovery (если идет)
ceph config set osd osd_max_backfills 1
ceph config set osd osd_recovery_max_active 1
ceph config set osd osd_recovery_sleep 0.2

# Включить QoS для приоритизации клиентского трафика
# На коммутаторах настроить приоритет для public network
```

**Случай 3: MTU mismatch**
```bash
# Проверить MTU на всех узлах
ip link show bond0 | grep mtu
ip link show bond1 | grep mtu

# Если разные - выровнять (обычно 9000 для jumbo frames)
ip link set bond0 mtu 9000
ip link set bond1 mtu 9000

# Постоянная настройка в netplan/ifcfg
```

### Проблема 8: Degraded objects не восстанавливаются

**Симптомы:**
```bash
ceph -s
# degraded: 123456/3456789 objects degraded (3.571%)
# И число не уменьшается часами
```

**Диагностика:**
```bash
# Проверить состояние PG
ceph pg dump | grep -E 'degraded|undersized|peered'

# Проверить recovery
ceph pg dump | grep recovery

# Проверить проблемные OSD
ceph osd tree | grep down

# Проверить флаги
ceph osd dump | grep flags
```

**Решения:**

```bash
# 1. Проверить что recovery не заблокирован
ceph osd unset norecover
ceph osd unset nobackfill

# 2. Увеличить приоритет recovery (осторожно!)
ceph config set osd osd_recovery_op_priority 3
ceph config set osd osd_max_backfills 2
ceph config set osd osd_recovery_max_active 2

# 3. Принудительно запустить recovery на конкретных PG
ceph pg force-recovery 1.a
ceph pg force-backfill 1.a

# 4. Если есть down OSD - поднять их
ceph osd in osd.X
systemctl start ceph-osd@X

# 5. Проверить нехватку места
ceph osd df tree
# Если OSD > 85% заполнен - нужно добавить места
```

---

## Чеклисты и Best Practices

### Pre-Production Checklist

Перед запуском в production убедитесь:

**Оборудование:**
- [ ] Минимум 3 узла в разных стойках/зонах доступности
- [ ] SSD диски с power-loss protection (конденсаторы)
- [ ] 8+ GB RAM на каждый OSD
- [ ] CPU с высокой частотой (3.0+ GHz), 16+ ядер
- [ ] Две сети: public (25+ Gbit/s) и cluster (25+ Gbit/s)
- [ ] Bonded сетевые интерфейсы (LACP)
- [ ] Все диски < 4 лет

**Операционная система:**
- [ ] Ubuntu 22.04 LTS или RHEL 9.x
- [ ] Синхронизация времени через chrony (offset < 50ms)
- [ ] Настройка ядра (sysctl) применена
- [ ] Отключено энергосбережение CPU (performance governor)
- [ ] Отключены Transparent Huge Pages
- [ ] Увеличены лимиты (nofile, nproc, memlock)
- [ ] SMART мониторинг настроен

**Ceph конфигурация:**
- [ ] Ceph версия Reef (18.x) или Quincy (17.x)
- [ ] Фактор репликации size=3, min_size=2
- [ ] CRUSH Map соответствует физической топологии
- [ ] Автоматическое управление PG включено
- [ ] Настроены лимиты ресурсов через systemd (8GB RAM на OSD)
- [ ] Recovery параметры настроены консервативно
- [ ] Таймаут отказа OSD = 3600 сек (1 час)
- [ ] Scrubbing настроен на ночное время

**Мониторинг:**
- [ ] Dashboard включен и доступен
- [ ] Prometheus экспорт настроен
- [ ] Grafana с дашбордами Ceph
- [ ] Alertmanager с правилами для Ceph
- [ ] Email уведомления о критичных событиях
- [ ] SMART мониторинг дисков
- [ ] Ежедневные health check скрипты

**Документация:**
- [ ] Backup конфигурации сделан
- [ ] Процедура замены диска задокументирована
- [ ] Процедура добавления узлов задокументирована
- [ ] Список контактов для эскалации
- [ ] План disaster recovery

**Тестирование:**
- [ ] Отказ одного OSD (замена диска)
- [ ] Отказ одного узла (перезагрузка)
- [ ] Отказ сети (разрыв соединения)
- [ ] Производительность (rados bench)
- [ ] Процедура восстановления из backup

### Production Best Practices

#### Общие правила

1. **Никогда не используйте size=1**
   - Минимум size=3, min_size=2
   - Для критичных данных: size=3, min_size=3

2. **Всегда используйте две сети**
   - Public для клиентов
   - Cluster для репликации
   - Минимум 25 Gbit/s каждая

3. **Регулярно заменяйте диски**
   - План замены дисков > 3 лет
   - Мониторинг SMART еженедельно
   - Плановая замена лучше аварийной

4. **Флаги перед обслуживанием**
   ```bash
   # ВСЕГДА устанавливать перед:
   # - Заменой дисков
   # - Перезагрузкой узлов
   # - Сетевыми работами
   ceph osd set noout norebalance nobackfill
   ```

5. **Контролируемый recovery**
   ```bash
   # Не позволяйте recovery убивать клиентов
   osd_max_backfills = 1
   osd_recovery_max_active = 1
   osd_recovery_sleep = 0.1
   ```

6. **Мониторинг времени**
   - Синхронизация < 50ms обязательна
   - Автоматические алерты при drift
   - Резервные NTP серверы

7. **Backup конфигурации**
   - Еженедельный backup config/crushmap/auth
   - Хранить минимум 4 последних копии
   - Проверять возможность восстановления

8. **Capacity planning**
   - Не допускать заполнения > 75%
   - Планировать расширение за 6 месяцев
   - Учитывать рост на 20% в год

#### Масштабирование

1. **Добавление узлов:**
   - Устанавливайте флаги ПЕРЕД добавлением
   - Добавляйте четное количество узлов
   - Мониторьте нагрузку на клиентов

2. **Стратегия расширения:**
   - Постепенно: безопаснее, дольше
   - Ночью: минимальное влияние, очень долго
   - В выходные: быстро, нужно дежурить

3. **MON демоны:**
   - Держите нечетное число (3, 5, 7)
   - Не более 7 MON в большинстве случаев
   - Размещайте на разных узлах/стойках

#### Обслуживание

1. **Замена дисков:**
   - Проверяйте SMART перед заменой
   - Используйте идентичные диски
   - Очищайте диск перед добавлением (wipefs)

2. **Обновления:**
   - Читайте Release Notes!
   - Тестируйте на dev кластере
   - Делайте backup перед обновлением
   - Обновляйте в maintenance window

3. **Scrubbing:**
   - Настройте на ночное время
   - Не отключайте надолго
   - Deep scrub минимум раз в 2 недели

#### Производительность

1. **Диски:**
   - Используйте SSD с PLP
   - 1+ DWPD для production
   - Отключите планировщик I/O (none для SSD)

2. **Сеть:**
   - Включите jumbo frames (MTU 9000)
   - Используйте LACP bonding
   - Настройте QoS на коммутаторах

3. **Память:**
   - 8 GB на OSD минимум
   - Ограничивайте через systemd
   - Мониторьте OOM

4. **CPU:**
   - Performance governor обязательно
   - Отключите C-states
   - Минимум 2 ядра на OSD

### Типичные ошибки и как их избежать

❌ **Ошибка 1: Фактор репликации 1**
```bash
# НЕПРАВИЛЬНО
ceph osd pool set mypool size 1

# ПРАВИЛЬНО
ceph osd pool set mypool size 3 min_size 2
```

❌ **Ошибка 2: Добавление узлов без флагов**
```bash
# НЕПРАВИЛЬНО - начнется немедленный ребаланс!
ceph orch host add node4 ...

# ПРАВИЛЬНО
ceph osd set noout norebalance nobackfill
ceph orch host add node4 ...
# ... добавление дисков ...
ceph osd unset norebalance nobackfill noout
```

❌ **Ошибка 3: Игнорирование SMART предупреждений**
```bash
# Проверяйте SMART еженедельно!
smartctl -a /dev/sdc | grep -E 'Reallocated|Pending|Uncorrectable'

# Если > 0 - планируйте замену!
```

❌ **Ошибка 4: Одна сеть для всего**
```bash
# НЕПРАВИЛЬНО
# public_network = cluster_network = 10.0.1.0/24

# ПРАВИЛЬНО
# public_network = 10.0.1.0/24
# cluster_network = 10.0.2.0/24
```

❌ **Ошибка 5: Нет ограничений памяти**
```bash
# НЕПРАВИЛЬНО - OSD может съесть всю память

# ПРАВИЛЬНО
systemctl edit ceph-osd@0.service
[Service]
MemoryMax=8G
MemoryHigh=7G
```

❌ **Ошибка 6: Увеличение PG без тестирования**
```bash
# НЕПРАВИЛЬНО - можно застрять в ребалансировке
ceph osd pool set mypool pg_num 2048

# ПРАВИЛЬНО
# 1. Тестировать в dev окружении
# 2. Использовать pg_autoscale
ceph osd pool set mypool pg_autoscale_mode on
```

❌ **Ошибка 7: Игнорирование рассинхронизации времени**
```bash
# Проверяйте ВСЕГДА
chronyc tracking
# System time offset должен быть < 0.05 секунд

# Если больше - НЕМЕДЛЕННО исправлять!
systemctl restart chrony
chronyc makestep
```

---

## Полезные команды

### Быстрая диагностика

```bash
# Состояние кластера
ceph -s
ceph health detail

# Состояние OSD
ceph osd tree
ceph osd stat
ceph osd df tree

# Состояние пулов
ceph df
ceph osd pool ls detail

# Производительность
ceph osd perf
rados bench -p mypool 10 write
rados bench -p mypool 10 seq
rados bench -p mypool 10 rand

# Проблемные PG
ceph pg dump | grep -v active+clean
ceph pg ls-by-osd 5  # PG на конкретном OSD

# Логи
journalctl -u ceph-osd@5 -f
journalctl -u ceph-mon@node1 -f
ceph log last 100
```

### Аварийные команды

```bash
# Экстренная остановка recovery (если убивает клиентов)
ceph osd set norecover
ceph osd set nobackfill

# Пометить OSD как lost (КРИТИЧНО!)
ceph osd lost 5 --yes-i-really-mean-it

# Принудительный ремонт PG
ceph pg repair 1.a

# Форсировать recovery
ceph pg force-recovery 1.a
ceph pg force-backfill 1.a

# Пометить PG как lost (потеря данных!)
ceph pg 1.a mark_unfound_lost revert
```

### Maintenance mode

```bash
# Вход в maintenance
ceph osd set noout
ceph osd set norecover
ceph osd set nobackfill
ceph osd set norebalance
ceph osd set noscrub
ceph osd set nodeep-scrub

# Выход из maintenance
ceph osd unset noout
ceph osd unset norecover
ceph osd unset nobackfill
ceph osd unset norebalance
ceph osd unset noscrub
ceph osd unset nodeep-scrub
```

---

## Заключение

Ceph - это мощная и надежная система хранения данных, но она требует:
- **Правильного оборудования** (SSD, две сети, достаточно RAM)
- **Правильной настройки** (репликация, топология, параметры)
- **Постоянного мониторинга** (health, SMART, производительность)
- **Регулярного обслуживания** (замена дисков, обновления)

При соблюдении этих требований Ceph обеспечит:
- Высокую доступность (99.99%+)
- Отличную производительность
- Линейное масштабирование
- Защиту от потери данных

### Дополнительные ресурсы

**Документация:**
- Официальная документация: https://docs.ceph.com/
- Red Hat Ceph Storage: https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/
- Ceph Blog: https://ceph.io/en/news/blog/

**Сообщество:**
- Ceph Users mailing list: ceph-users@ceph.io
- Ceph Dev mailing list: dev@ceph.io
- IRC: #ceph on OFTC
- Slack: https://ceph.io/slack

**Обучение:**
- Ceph Foundation Training
- Red Hat Ceph Administration (DO370)

**Полезные инструменты:**
- ceph-ansible: Ansible playbooks для Ceph
- Rook: Ceph operator для Kubernetes
- ceph-deploy: Legacy deployment tool

---

**Версия документа:** 1.0  
**Последнее обновление:** 2025-01-28  
**Тестировано на:** Ceph Reef (18.2.x), Ubuntu 22.04 LTS

**Авторы:** Based on production experience and Ceph community best practices

---

## Приложение: Примеры конфигураций

### Пример 1: Малый production кластер (3 узла)

```yaml
Конфигурация:
  Узлов: 3
  OSD на узел: 10
  Емкость: 120 TB raw (40 TB usable)
  RAM на узел: 96 GB
  CPU: 24 cores @ 3.0 GHz
  Сеть: 2 × 25 Gbit/s

Параметры Ceph:
  size: 3
  min_size: 2
  pg_autoscale: on
  osd_max_backfills: 1
  osd_recovery_max_active: 1
  mon_osd_down_out_interval: 3600

Use case:
  - Средние базы данных
  - Хранилище для виртуальных машин
  - S3 объектное хранилище
```

### Пример 2: Большой production кластер (10 узлов)

```yaml
Конфигурация:
  Узлов: 10
  OSD на узел: 12
  Емкость: 480 TB raw (160 TB usable)
  RAM на узел: 128 GB
  CPU: 32 cores @ 3.2 GHz
  Сеть: 2 × 100 Gbit/s

Параметры Ceph:
  size: 3
  min_size: 2
  pg_autoscale: on
  osd_max_backfills: 2
  osd_recovery_max_active: 2
  mon_osd_down_out_interval: 3600
  MON: 5 демонов

Use case:
  - Большие базы данных
  - Cloud storage платформа
  - Big Data / Analytics
  - High-performance computing
```

**КОНЕЦ ДОКУМЕНТА**

**Диагностика:**

```bash
# 1. Проверка логов
journalctl -u ceph-osd@5 -n 100 --no-pager

# Частые ошибки:
# - "Input/output error" → проблема с диском
# - "Cannot allocate memory" → нехватка RAM
# - "Authentication failed" → проблема с ключами
# - "failed to load OSD map" → поврежденный OSD

# 2. Проверка диска
lsblk | grep -E 'sdb|sdc'
smartctl -a /dev/sdc

# 3. Проверка файловой системы
df -h | grep osd
mount | grep osd

# 4. Проверка прав доступа
ls -la /var/lib/ceph/osd/ceph-5/
```

**Решения:**

**Случай 1: Поврежденный BlueStore**
```bash
# Попытка ремонта
ceph-objectstore-tool --data-path /var/lib/ceph/osd/ceph-5 --op fsck

# Если не помогло - пересоздать OSD
ceph osd destroy osd.5 --yes-i-really-mean-it
ceph-volume lvm zap /dev/sdc --destroy
ceph orch daemon add osd node2:/dev/sdc
```

**Случай 2: Проблемы с RocksDB/LevelDB**
```bash
# Попытка восстановления
ceph-kvstore-tool bluestore-kv /var/lib/ceph/osd/ceph-5 repair

# Если не помогло - см. Случай 1
```

**Случай 3: Нехватка памяти**
```bash
# Временно увеличить лимиты
systemctl edit ceph-osd@5.service

[Service]
MemoryMax=16G

systemctl daemon-reload
systemctl restart ceph-osd@5
```

### Проблема 2: PG stuck в состоянии inactive/incomplete

**Симптомы:**
```bash
ceph health detail
# HEALTH_WARN 5 pgs incomplete
# pg 1.3a is incomplete, acting [2,4]
```

**Диагностика:**
```bash
# Посмотреть детали проблемных PG
ceph pg 1.3a query

# Проверить где должны быть данные
ceph pg map 1.3a

# Проверить состояние OSD
ceph osd tree
```

**Решения:**

**Случай 1: Отсутствует OSD с данными**
```bash
# Если OSD с данными точно потерян
# КРИТИЧНО: Это приведет к потере данных!

# Пометить PG как lost
ceph pg 1.3a mark_unfound_lost revert

# ИЛИ удалить (если данные не критичны)
ceph pg 1.3a mark_unfound_lost delete
```

**Случай 2: OSD есть, но не в acting set**
```bash
# Принудительно включить OSD
ceph osd in osd.X

# Форсировать recovery
ceph pg force-recovery 1.3a
ceph pg force-backfill 1.3a
```

### Проблема 3: Медленные запросы (slow ops/requests)

**Симптомы:**
```bash
ceph health detail
# HEALTH_WARN 150 slow ops, oldest one blocked for 30 sec
```

**Диагностика:**
```bash
# Посмотреть медленные запросы
ceph daemon osd.5 dump_historic_slow_ops

# Топ медленных OSD
ceph osd perf

# Проверка latency
ceph osd perf | sort -k3 -n

# Проверка очереди
for osd in {0..29}; do
  echo "OSD $osd:"
  ceph daemon osd.$osd dump_ops_in_flight
done
```

**Решения:**

**Случай 1: Проблемы с диском**
```bash
# Проверить диски
smartctl -a /dev/sdc
iotop -o  # посмотреть нагрузку I/O

# Если диск умирает - заменить
# См. раздел "Замена неисправ# Ceph Production-Ready: Полное руководство по развертыванию и эксплуатации

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
11. [Чеклисты и best practices](#чеклисты-и-best-practices)

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

# НЕМЕДЛЕННО добавить новые MON!
```

---

## Troubleshooting

### Проблема 1: OSD не стартует

**Симптомы:**
```bash
systemctl status ceph-osd@5
# Failed to start Ceph OSD Daemon
