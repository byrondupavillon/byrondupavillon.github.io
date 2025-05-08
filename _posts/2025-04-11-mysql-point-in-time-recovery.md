---
title: "MySQL Point-in-Time Recovery: Restore from Backup and Replay Binary Logs"
date: 2025-04-11
categories:
  - database
tags:
  - disaster recovery
  - mysql
  - backup
  - binlogs
---

This guide walks you through the process of **restoring a MySQL database** after a **drive failure** or **data loss** event, using a combination of **full backups** and **binary logs** to perform point-in-time recovery.

You can use this guide in one of two recovery situations:

- **Fresh install** (reinstalling MySQL from scratch)
- **Restoring a previous MySQL root drive installation** (from a snapshot)

---
## ⚙️ **Before You Begin**

> ⚠️ It is <strong>strongly recommended</strong> to perform the steps in this guide, inside a <a href="https://linux.die.net/man/1/screen">screen</a> or <a href="https://github.com/tmux/tmux/wiki">tmux</a> session. This helps prevent interruptions from dropped SSH connections during long-running processes.

---

## 🧱 **If Performing a Fresh MySQL Install**

Follow these steps **only if** you're setting up MySQL from scratch on a new server:

### 🔧 1. Install Percona Software for MySQL

You can either install the **latest available version** or a **specific pinned version** of MySQL or Percona XtraDB Cluster (PXC).

#### 📦 A. Install the Latest Version

1. **Install prerequisites and Percona APT repo**:
    
    ```bash
    sudo apt update
    sudo apt install -y wget gnupg2 lsb-release curl
    wget https://repo.percona.com/apt/percona-release_latest.generic_all.deb
    sudo dpkg -i percona-release_latest.generic_all.deb
    sudo apt update
    ```
    
2. **Install [Percona Server for MySQL](https://docs.percona.com/percona-server/8.0/installation.html)**:
    
    ```bash
    sudo percona-release setup ps80
    sudo apt install -y percona-server-server
    ```
    
3. **(Alternative) Install [Percona XtraDB Cluster](https://docs.percona.com/percona-xtradb-cluster/8.0/quickstart-overview.html)**:
    
    ```bash
    sudo percona-release setup pxc80 
    sudo apt install -y percona-xtradb-cluster
    ```
    

#### 📦 B. Install a Specific Version

If you need to pin a specific version (e.g., for compatibility or replication parity):

```bash
# Example: Install Percona XtraDB Cluster 8.0.36-28-1
wget https://downloads.percona.com/downloads/Percona-XtraDB-Cluster-80/Percona-XtraDB-Cluster-8.0.36/binary/debian/jammy/x86_64/percona-xtradb-cluster_8.0.36-28-1.jammy_amd64.deb
sudo dpkg -i percona-xtradb-cluster_8.0.36-28-1.jammy_amd64.deb

# Fix dependencies if prompted
sudo apt --fix-broken install

# Prevent this package from being auto-upgraded
sudo apt-mark hold percona-xtradb-cluster
```

---

### ⚙️ 2. Configure `my.cnf` for Source/Replica Setup

To set up binary logging and server IDs, either copy the `my.cnf` from the source server or configure it manually as follows:

### **Source server :**

```
#bind-address            = 127.0.0.1
server-id               = 1
log_bin                 = /var/log/mysql/mysql-bin.log
# If you have a different location you want your bin logs to be stored, replace the above.
expire_logs_days        = 10
max_binlog_size         = 100M
```

### **Replica server :**

```
#bind-address            = 127.0.0.1
server-id               = 2
log_bin                 = /var/log/mysql/mysql-bin.log
# If you have a different location you want your bin logs to be stored, replace the above.
expire_logs_days        = 10
max_binlog_size         = 100M
```

---

## 🚚 **Getting Backup Files onto the New Server**

To begin the recovery process, you’ll need to transfer the **backup and binary log files** to the new MySQL server.

Choose the option below that best **suits** your setup:

- **Option 1:** Attach the **backup drive** to the new server, this is often the fastest method.
    - **AWS Only:**
    Maximize the throughput of the attached backup volume to reduce recovery time.
        
        ![aws-ec2-create-volume.jpg](/assets/img/posts/2025-04-11-mysql-point-in-time-recovery/aws-ec2-create-volume.jpg)
        
    - **Forgot to set throughput during restore?**
    Use the following AWS CLI command to update the volume's performance settings:
        
        ```bash
        aws ec2 modify-volume --volume-id *volume-id* --throughput 1000
        ```
        
- **Option 2:** Use [Rsync](https://linux.die.net/man/1/rsync) to securely **copy the files over the network** between servers

---

## 💾 Preparing Backup

### 📌 Initialise Amazon EBS Volume (Pre-Warming) using fio— AWS Only

> 💡 Estimated time for this section: `4 hrs 35 min` based on a 2TB drive.

When attaching an **EBS volume** that contains a backup (especially large volumes or snapshots), **Amazon recommends pre-warming** to avoid I/O latency during initial access. This step ensures consistent performance during restore.

> 🔥 Important:
> 
> 
> If you need to prewarm **multiple drives**, do **not** run `fio` commands in **parallel** sessions on the same server—this will **slow down the entire process**. Instead, **run them sequentially**, one drive at a time.
> 

Installing `fio`

```bash
apt install fio
```

### 🛠️ Recommended `fio` Usage for Prewarming

Use `fio` with **random read** mode (`--rw=randread`), as it was found to be **faster** than using sequential reads (`--rw=read`) during benchmarking.

```bash
sudo fio \
	--filename=/dev/nvme1n1 \
	--rw=randread \
	--bs=1M \
	--iodepth=256 \
	--ioengine=libaio \
	--direct=1 \
	--name=volume-initialize
```

> Replace `/dev/nvme1n1` with the appropriate device path for your EBS volume.

![example-fio-output.jpg](/assets/img/posts/2025-04-11-mysql-point-in-time-recovery/example-fio-output.jpg)

---

### 🧰 **Prepare the Backup for Restoration**

> 💡 Estimated time for this section: `2 min`

Before you can restore the data, you must **prepare the backup** to ensure data consistency. This step handles:

- **Rolling back uncommitted transactions**, and
- **Replaying committed transactions** from the transaction logs.

Percona XtraBackup handles this using the `--apply-log` process, making the backup ready for immediate use.

#### 📦 **Step 1**: Install Percona XtraBackup

```bash
# For MySQL 8.0+
sudo percona-release setup pxb-80
apt-get install percona-xtrabackup-80
```

#### 🔐 Step 2: Decrypt the Backup (If Encrypted)

If your backup is encrypted, you’ll need to decrypt it first:

> ⚠️ Important:
> 
> 
> When creating the encryption key file, use `echo -n "key"` (with `-n`) to avoid including a trailing newline, which can cause decryption errors.
> 

```bash
# xtrabackup (Recommended for MySQL 8.0+)

xtrabackup \
	--decrypt=AES256 \
	--parallel=8 \
	--encrypt-key="ThisIsAnEncryptionKey" \
	--target-dir=/mnt/backups/mysql/2025-04-11_15-33-52/ \
	--remove-origina
```

#### 🛠️ Step 3: Apply the Transaction Logs (Preparing the backup)

```bash
# xtrabackup (Recommended for MySQL 8.0+)

xtrabackup \
	--prepare \
	--target-dir=/mnt/backups/mysql/2025-04-11_15-33-52
```

If the operation completes successfully, the final lines of the output will look similar to this:

```bash
2025-04-11T11:29:17 [InnoDB] FTS optimize thread exiting.
2025-04-11T11:29:18 [InnoDB] Starting shutdown...
2025-04-11T11:29:19 [InnoDB] Log background threads are being closed
2025-04-11T11:29:21 [InnoDB] Shutdown completed; log sequence number 123456789
2025-04-11T11:29:23 [InnoDB] [Xtrabackup] completed OK!
```

This confirms the backup is now ready to be restored and used.

---

## 📂 **Restore Backup files to Data Drive**

> 💡 Estimated time for this section: `1 hr 15 min`, based on a 1.8TB backup.

After preparing the backup, the next step is to **restore the data files to the MySQL data directory**. This process involves **stopping MySQL**, cleaning up old data, copying the prepared backup files, and ensuring the proper permissions.

1. Ensure MySQL is Not Running

    Before making any changes to the data directory, stop the MySQL service:
    
    ```bash
    systemctl stop mysql
    ```
    
2. Clean Up Old MySQL Data
    
    > ⚠️ **WARNING:** The following commands **permanently delete** all existing data and binary logs in MySQL.
    > 
    > 
    > Make sure you update the paths to your actual **data** and **binlog** directories before running these commands.
    > 
    
    ```bash
    rm -R /mnt/data/mysql/*
    rm -R /mnt/binlogs/mysql/*
    ```
    
3. Copy Backup Files to Data Directory

    Now copy the prepared backup files into the MySQL data directory.
    
    ```bash
    # For MySQL 8.0+
    
    xtrabackup --copy-back \
      --parallel=32 \
      --use-memory=100G \
      --target-dir=/mnt/backups/mysql/2025-04-11_15-33-52
    ```
    
    > 💡**Tip:** Always match the number of parallel threads in `xtrabackup --copy-back` with your available CPU cores for maximum throughput.
    > 
4. Clear Binlogs Again

    After copying the data files, it's a good idea to ensure the binlog directory is clean:
    
    ```bash
    rm -R /mnt/binlogs/mysql/*
    ```
    
5. Fix File Ownership and Permissions

    Ensure the MySQL process has full ownership of the data directory:
    
    ```bash
    chown -R mysql:mysql /mnt/data/mysql
    ```
    
6. Start MySQL Service
    
    ```bash
    systemctl start mysql
    ```
    
    > 💡 **Tip:** Always ensure MySQL is running properly after starting it by checking the status:
    > 
    > ```bash
    > systemctl status mysql
    > ```

---

## 📜 **Applying Binary Logs**

> 💡 Estimated time for this section: `1 hr 50 min`, based on a 30GB binary log set (approximately 300 ± binlog files) backup.


After restoring your backup, the final step in point-in-time recovery is to **replay the binary logs**. This ensures your MySQL instance reflects all the committed transactions that occurred after the backup was taken.

### 🔍 **Determine Starting Point for Binary Log Replay**

Before applying binary logs, you need to identify **where the backup left off**. This ensures that you replay only what’s needed — not older transactions that have already been applied.

> ⚠️ Note: These reference files are only created after running `xtrabackup --prepare`.
> 

---

### **📌 Restoring from the Source Server**

Locate and read the `xtrabackup_binlog_info` file in your backup directory:

```bash
nano /mnt/backups/mysql/2025-04-11_15-33-52/xtrabackup_binlog_info
```

Example output: 

```bash
mysql-bin.733993 4983
```

- `mysql-bin.733993` — The last binary log file included in the backup
- `4983` — The **position** in the log file where the backup ended

Make a note of both values — you'll need them later.

---

### 📌 **Restoring from the  Replica Server**

If restoring from a replica, use `xtrabackup_slave_info` to determine the binlog position from the source at the time of backup:

```bash
nano /mnt/backups/mysql/2025-04-11_15-33-52/xtrabackup_slave_info
```

Example output: 

```sql
CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin.733993', MASTER_LOG_POS=4983;
```

This is useful if:

- You're restoring the replica **as a new replica**
- You're using this replica as the **new source** and applying logs locally

---

### 🧾 **Update `my.cnf` Configuration Before Applying Binary Logs**

Before beginning the binary log replay process, ensure the following parameters are set in your **`my.cnf`** file. These settings help avoid unintended replication behaviors and ensure a consistent environment for log application: 

```sql
[mysqld]

# ✅ Allows events with the same server ID (useful when replaying logs from the same instance)
replicate-same-server-id = 1

# ⚠️ Above requires log_replica_updates to be disabled 
log_replica_updates = OFF

# 🛑 Prevents replication threads from auto-starting on server boot — required for controlled replay
skip-replica-start

# ⚙️ Improves recovery speed at the cost of slightly reduced durability (safe on OS crash, not on full power loss)
innodb_flush_log_at_trx_commit = 2

# ⏱️ Controls how often InnoDB flushes the redo logs to disk
innodb_flush_log_at_timeout = 5

# 📴 Disables binary logging on this server (we don’t want to log replayed binlog events)
skip-log-bin
```

📍 **Why these settings matter:**

- `replicate-same-server-id`: Needed because the binary logs you’re applying were originally generated by the same server.
- `skip-replica-start`: Gives you full control over when to start applying the relay logs.
- `innodb_flush_log_at_trx_commit = 2`: Speeds up recovery while still being safe on OS-level crashes.
- `innodb_flush_log_at_timeout`: Reduces disk I/O during bulk log replay.
- `skip-log-bin`: Prevents your replayed transactions from being written to new binlogs.

> 💡 Configure max_allowed_packet size to allow for binary log application, this may need to be a higher value than configured in original server conf file.
> In the `my.cnf` file just double whats there, for example
> max_allowed_packet = 128M
> 

✅ Next Step: After updating `my.cnf`, restart MySQL before beginning the log replay process.

```bash
systemctl restart mysql
```

---

### 🛠️ **Binary Log Recovery Options**

Depending on your recovery scenario, choose one of the two options below:

#### 🔄 **Option 1: Replay All Binary Logs (Up to Latest State)**

Use this option if you're recovering from a **server loss or crash**, and want to restore the server to its most recent state by **replaying all binlogs after the backup**.

✅ Steps:

1. Reset the replication state:
    
    ```sql
    mysql> STOP REPLICA;
    mysql> RESET REPLICA ALL;
    ```
    
2. Remove old relay logs (if present):
    
    ```bash
    rm /mnt/data/mysql/*relay*
    ```
    
3. Rename and copy binary logs to relay logs:
    
    ```bash
    for i in /mnt/backups/bin_logs/*.7*; do
      ext=$(echo $i | cut -d'.' -f2)
      cp "$i" "servername-relay-bin.$ext"
    done
    ```
    
    > 📝 Replace `servername` with the name of your target server.
    > 
4. Create the relay log index file:
    
    ```bash
    cd /mnt/data/mysql
    ls ./servername-relay-bin.7* > servername-relay-bin.index
    ```
    
5. Fix permissions:
    
    ```bash
    chown -R mysql:mysql /mnt/data/mysql/*relay*
    ```
    
6. Start replication using the relay logs:
    
    ```sql
    mysql> SET GLOBAL REPLICA_PARALLEL_WORKERS=32;
    
    mysql> CHANGE REPLICATION SOURCE TO
            SOURCE_HOST='dummy', 
            SOURCE_USER='fake', 
            SOURCE_LOG_FILE='mysql-bin.000001', 
            SOURCE_LOG_POS=4, 
            RELAY_LOG_FILE='servername-relay-bin.733993', 
            RELAY_LOG_POS=4983;
    				
    mysql> START REPLICA SQL_THREAD; 
    ```
    
    > 📈 You can monitor progress by checking relay logs on disk or using:
    > 
    
    ```sql
    mysql> SHOW REPLICA STATUS;
    ```
    
7. Finalize: Reset Replica Info
Once caught up:
    
    ```sql
    mysql> STOP REPLICA;
    mysql> RESET REPLICA ALL;
    ```
    

#### ⏸️ Option 2: Replay Up to a Specific Point in Time (Point-in-Time Recovery)

Use this option when you need to **recover up to a specific moment** — for example, just before a table was dropped or a destructive query was executed.

> ⚠️ Important: You will need to determine the exact relay log file and position to stop at. This can be found by inspecting the binary logs using `mysqlbinlog`, application logs, or MySQL general logs.
> 

#### 🔍 **Step 1: Find the Stop Position**

To determine where to stop replaying logs, inspect the relevant binary logs and look for the destructive or unwanted statement.

You can use `mysqlbinlog` to convert relay log content into readable SQL:

```bash
mysqlbinlog -v /mnt/backups/bin_logs/servername-relay-bin.744231 > mybinlog.sql
```

Then open the output file (`mybinlog.sql`) and scan for the problematic query.

**🧠 Example Log Output:**

```sql
... COMMIT/*!*/;
# at 14260
#190310 20:15:56 server id 1 end_log_pos 14910 CRC32 0x96b04ba6 Anonymous_GTID last_committed=1 sequence_number=2 rbr_only=no
SET @@SESSION.GTID_NEXT= ‘ANONYMOUS’/*!*/;
# at 14910
#190310 20:15:56 server id 1 end_log_pos 15720 CRC32 0x299fea70 Query thread_id=27 exec_time=0 error_code=0
SET TIMESTAMP=1552245356/*!*/;
BEGIN/*!*/;
#190310 20:15:53 server id 1  end_log_pos 13950 CRC32 0x7f6ee5e8         Update_rows: table id 125 flags: STMT_END_F
...
### DELETE `important_table`
```

> 🎯 In this example, use log **position `14910`** as your `RELAY_LOG_POS` to **stop the recovery just before the destructive DELETE** occurs.
> 

#### ⚙️ **Step 2: Apply Binlogs with Stop Condition (PITR)**

Follow **Steps 1–5** from **Option 1**, including renaming the binary logs to relay logs, building the index, and fixing permissions.

Then, when you get to **Step 6 (Starting replication)**, instead of running `START REPLICA SQL_THREAD`, you'll use the `UNTIL` clause to stop at the exact position:

```sql
mysql> SET GLOBAL REPLICA_PARALLEL_WORKERS=32;

mysql> CHANGE REPLICATION SOURCE TO
        SOURCE_HOST='dummy', 
        SOURCE_USER='fake', 
        SOURCE_LOG_FILE='mysql-bin.000001', 
        SOURCE_LOG_POS=4, 
        RELAY_LOG_FILE='servername-relay-bin.733993', 
        RELAY_LOG_POS=4983;
				
mysql> START REPLICA SQL_THREAD UNTIL
        RELAY_LOG_FILE = 'servername-relay-bin.744231',
        RELAY_LOG_POS = 14910;
```

> 🛑 MySQL will automatically stop once the relay log has reached the specified position. This allows you to safely verify data before restarting normal operations.
> 

Finalize: Reset Replica Info
Once caught up:

```sql
mysql> STOP REPLICA;
mysql> RESET REPLICA ALL;
```

---

## ✅ Final Verification

Once the recovery process is complete or the replica has fully caught up:

- **Check MySQL logs**:
    
    Inspect `/var/log/mysql/error.log` for warnings, crash recovery messages, or replication issues.
    
- **Validate application connectivity**:
    
    Confirm that your applications can successfully connect and query the database.

> 💡 Tip: Consider snapshotting the volume now — a clean, restorable point post-recovery.
> 

---

# 🔁 Restore Script

The script below automates Steps 3 and 4 outlined above. It's especially useful for repeated runs, allowing you to easily test different configurations or settings.

```bash
#!/bin/bash

########################################
# 🛠️ MySQL Restore & Binlog Replay Script
# --------------------------------------
# This script restores a MySQL backup using Percona XtraBackup,
# and replays binary logs for point-in-time recovery.
########################################

# === Configuration ===
USER="root"
PASSWORD="password"
HOST="127.0.0.1"
DATABASE="test"
BACKUP_DIR="/mnt/backups/mysql/2025-04-11_15-33-52"
BINLOG_DIR="/mnt/backups/bin_logs"
DATA_DIR="/mnt/data/mysql"
BINLOG_TARGET_DIR="/mnt/binlogs/mysql"
RELAY_LOG_PREFIX="servername-relay-bin"

# === 1. Stop MySQL Service ===
echo "Stopping MySQL..."
systemctl stop mysql

# === 2. Clean Up Data & Binlog Directories ===
echo "🧹 Cleaning existing MySQL data and binlogs..."
rm -rf "$DATA_DIR"/*
rm -rf "$BINLOG_TARGET_DIR"/*

# === 3. Restore Backup with XtraBackup ===
echo "📦 Restoring backup from $BACKUP_DIR..."
xtrabackup --copy-back \
  --parallel=32 \
  --use-memory=100G \
  --target-dir="$BACKUP_DIR"

# === 4. Clean Binlog Directory Again (for safety) ===
echo "🧹 Re-cleaning binlog directory..."
rm -rf "$BINLOG_TARGET_DIR"/*

# === 5. Fix File Permissions ===
echo "🔧 Setting ownership for MySQL data directory..."
chown -R mysql:mysql "$DATA_DIR"

# === 6. Start MySQL Service ===
echo "🚀 Starting MySQL..."
systemctl start mysql

########################################
# 📜 BINLOG REPLAY SECTION
########################################

# === 7. Reset Replica Metadata ===
echo "🔄 Resetting MySQL replica state..."
mysql -u"$USER" -p"$PASSWORD" -h"$HOST" "$DATABASE" <<EOF
STOP REPLICA;
RESET REPLICA ALL;
EOF

# === 8. Prepare Relay Logs ===
echo "📁 Preparing relay logs..."
cd "$DATA_DIR" || exit 1

# Remove any existing relay logs
rm -f *relay*

# Copy and rename binary logs as relay logs
for i in "$BINLOG_DIR"/*.7*; do
  ext=\$(echo "\$i" | awk -F. '{print \$2}')
  cp "\$i" "${RELAY_LOG_PREFIX}.\$ext"
done

# Create relay log index file
ls ./${RELAY_LOG_PREFIX}.7* > "${RELAY_LOG_PREFIX}.index"

# Fix permissions
chown -R mysql:mysql *relay*

# === 9. Start SQL Thread for Replay ===
echo "🚀 Replaying binary logs..."
mysql -u"$USER" -p"$PASSWORD" -h"$HOST" "$DATABASE" <<EOF
SET GLOBAL REPLICA_PARALLEL_WORKERS=32;

CHANGE REPLICATION SOURCE TO
    SOURCE_HOST='dummy', 
    SOURCE_USER='fake', 
    SOURCE_LOG_FILE='mysql-bin.000001', 
    SOURCE_LOG_POS=4, 
    RELAY_LOG_FILE='${RELAY_LOG_PREFIX}.733993', 
    RELAY_LOG_POS=4983;

START REPLICA SQL_THREAD;
EOF

# === (Optional) Final Replica Reset ===
# Uncomment the lines below if you'd like to fully reset the replica state after replay
# echo "🧹 Final reset of replica state..."
# mysql -u"$USER" -p"$PASSWORD" -h"$HOST" "$DATABASE" <<EOF
# STOP REPLICA;
# RESET REPLICA ALL;
# EOF

echo "✅ MySQL restore and binlog replay completed!"
```
