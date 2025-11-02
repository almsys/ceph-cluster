# Ceph: Backup, настройка кластера между DC

## Содержание

1. [Backup стратегии для Ceph](#Backup-стратегии-для-Ceph)
2. [Multi-Datacenter конфигурации](#Multi-Datacenter-конфигурации)


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

# [Troubleshooting]
### [Troubleshooting]
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
