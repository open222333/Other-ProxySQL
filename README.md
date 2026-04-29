# Other-ProxySQL

```
ProxySQL + MySQL 主從高可用練習環境
```

## MySQL 版本分支

| 分支 | ProxySQL 版本 | 支援 MySQL 範圍 | 說明 |
|------|--------------|----------------|------|
| [feat/mysql80-support](https://github.com/open222333/Other-ProxySQL/tree/feat/mysql80-support) | 2.5.5 | **5.7 ～ 8.0** | 需用 `mysql_native_password` 建立帳號（8.4 已移除此插件） |
| [feat/mysql84-support](https://github.com/open222333/Other-ProxySQL/tree/feat/mysql84-support) | 2.7.1 | **8.0 ～ 8.4** | 原生支援 `caching_sha2_password`，直接建立帳號即可 |

---

## 目錄

- [專案說明](#專案說明)
- [架構概覽](#架構概覽)
- [各模式說明](#各模式說明)
  - [Master-Slave 主從複製模式](#master-slave-主從複製模式)
  - [MGR 群組複製模式](#mgr-群組複製模式)
- [設定檔說明](#設定檔說明)
  - [proxysql.cnf](#proxysqlcnf)
  - [MySQL cnf](#mysql-cnf)
- [執行流程](#執行流程)
  - [Master-Slave 環境啟動流程](#master-slave-環境啟動流程)
  - [MGR 環境啟動流程](#mgr-環境啟動流程)
  - [故障轉移流程 (MGR)](#故障轉移流程-mgr)
  - [節點恢復流程 (MGR)](#節點恢復流程-mgr)
- [使用方法](#使用方法)
  - [環境準備](#環境準備)
  - [啟動環境](#啟動環境)
  - [連接 ProxySQL](#連接-proxysql)
  - [初始化 ProxySQL 設定](#初始化-proxysql-設定)
  - [MySQL 主從設定](#mysql-主從設定)
  - [負載平衡設定](#負載平衡設定)
  - [gr_sw_mode_checker.sh 故障轉移腳本](#gr_sw_mode_checkersh-故障轉移腳本)
- [實際伺服器部署（Master-Slave）](#實際伺服器部署master-slave)
  - [部署架構](#部署架構)
  - [模板 vs 實際部署差異](#模板-vs-實際部署差異)
  - [Step 1：MySQL Master 準備](#step-1mysql-master-準備)
  - [Step 2：MySQL Slave 準備](#step-2mysql-slave-準備)
  - [Step 3：ProxySQL 主機初始化](#step-3proxysql-主機初始化)
  - [Step 4：調整設定檔](#step-4調整設定檔)
  - [Step 5：啟動 ProxySQL](#step-5啟動-proxysql)
  - [Step 6：設定後端節點與路由](#step-6設定後端節點與路由)
  - [Step 7：驗證](#step-7驗證)
- [常用查詢](#常用查詢)
  - [伺服器與使用者](#伺服器與使用者)
  - [路由相關](#路由相關)
  - [監控相關](#監控相關)
  - [MGR 群組狀態](#mgr-群組狀態)
- [建議注意事項](#建議注意事項)
- [參考資料](#參考資料)

---

## 專案說明

本專案提供兩種高可用模式的 Docker 練習環境：

1. **Master-Slave 主從複製** — ProxySQL 搭配傳統 MySQL 主從，讀寫分離
2. **MGR 群組複製** — ProxySQL 搭配 MySQL Group Replication，自動故障轉移

---

## 架構概覽

```
Client
  │
  ▼
ProxySQL (6033)          ← 應用程式連接埠（讀寫分離代理）
  │   ProxySQL Admin (6032) ← 管理介面
  │   SQLite Web  (8080)    ← ProxySQL DB 瀏覽器（選用）
  │   Web UI      (6080)    ← ProxySQL Web 介面（選用）
  │
  ├──► Hostgroup 1 (Writer) → MySQL Master / Primary
  └──► Hostgroup 2 (Reader) → MySQL Slave / Secondary
```

**連接埠說明：**

| 埠號 | 用途 |
|------|------|
| 6032 | ProxySQL 管理介面（Admin） |
| 6033 | ProxySQL MySQL 代理連接（應用程式使用） |
| 6080 | ProxySQL Web UI |
| 8080 | SQLite Web（ProxySQL DB 瀏覽器） |
| 3306 | MySQL Master |
| 3307 | MySQL Slave1 |

---

## 各模式說明

### Master-Slave 主從複製模式

位於 `mysql_replication/`，使用傳統 binlog 主從複製。

- **Master** 負責寫入，**Slave** 負責讀取
- ProxySQL 透過 `mysql_replication_hostgroups` 設定讀寫分離
- 路由規則：`INSERT / UPDATE / SELECT...FOR UPDATE` → Hostgroup 1（Master），`SELECT` → Hostgroup 2（Slave）
- 主節點宕機需手動切換，無自動故障轉移

### MGR 群組複製模式

使用 MySQL Group Replication，配合 `script/gr_sw_mode_checker.sh` 腳本實現自動故障轉移。

- **Primary** 負責寫入，**Secondary** 可讀取（Single-Primary 模式）
- ProxySQL 透過 `mysql_group_replication_hostgroups` 設定分組
- 搭配 checker 腳本定期偵測節點健康狀態，自動調整 ProxySQL 路由
- 主節點宕機後，腳本自動將存活的 Secondary 提升為 Primary 並更新 ProxySQL 設定

**Hostgroup 分組（MGR 模式）：**

| Hostgroup ID | 角色 |
|---|---|
| 10 | Writer（Primary） |
| 20 | Backup Writer |
| 30 | Reader（Secondary） |
| 9999 | Offline |

---

## 設定檔說明

### proxysql.cnf

從 `conf/proxysql.cnf.default` 複製為 `conf/proxysql.cnf`，並將 `CHANGE_ME` 取代為實際密碼：

```ini
datadir="/var/lib/proxysql"
logfile="/logs/proxysql/proxysql.log"

admin_variables= {
    # 格式: user:password;user2:password2，請修改為強密碼
    admin_credentials="admin:CHANGE_ME;radmin:CHANGE_ME2"
    mysql_ifaces="0.0.0.0:6032"
    web_enabled=true
    web_port=6080
}

# MGR 模式分組設定
mysql_groups = (
    {
        writer_hostgroup=10
        backup_hostgroup=20
        reader_hostgroup=30
        offline_hostgroup=9999
        max_writers=1
        writer_is_also_reader=1
    }
)

mysql_servers = (
    {
        hostgroup_id=10
        hostname="mysql_server1_ip"
        port=3306
    },
    {
        hostgroup_id=20
        hostname="mysql_server2_ip"
        port=3306
    }
)
```

> 修改 `mysql_server1_ip` / `mysql_server2_ip` 為實際 MySQL 節點 IP 或容器名稱。

### MySQL cnf

**Master：**
```ini
server-id = 1
log-bin = mysql-master-bin
```

**Slave：**
```ini
[mysqld]
server-id = 2
log-bin = mysql-slave1-bin
```

---

## 執行流程

### Master-Slave 環境啟動流程

```
1. 複製設定檔
   conf/proxysql.cnf.default → conf/proxysql.cnf
        │
        ▼
2. 啟動 MySQL 主從 + ProxySQL
   cd mysql_replication/
   docker compose up -d
        │
        ▼
3. 設定 MySQL Master
   建立 replication 使用者
   SHOW MASTER STATUS → 取得 binlog 檔名與 position
        │
        ▼
4. 設定 MySQL Slave
   CHANGE MASTER TO ...（填入 Master 的 binlog 資訊）
   START SLAVE
        │
        ▼
5. 連接 ProxySQL Admin（6032）
   建立 monitor 使用者、proxysql 使用者
   設定監控帳密、路由規則、Host Groups
   LOAD / SAVE 生效
        │
        ▼
6. 透過 6033 連接驗證讀寫分離
```

### MGR 環境啟動流程

```
1. 複製設定檔
   conf/proxysql.cnf.default → conf/proxysql.cnf
        │
        ▼
2. 啟動 ProxySQL + SQLite Web
   docker compose up -d
        │
        ▼
3. 連接 ProxySQL Admin（6032）
   設定 MySQL 節點（mysql_servers）
   設定分組（mysql_group_replication_hostgroups）
   設定監控帳密、使用者、路由規則
   LOAD / SAVE 生效
        │
        ▼
4. 啟動 gr_sw_mode_checker.sh 腳本
   持續監控節點健康，自動調整路由
```

### 故障轉移流程 (MGR)

```
主節點宕機
    │
    ▼
gr_sw_mode_checker.sh 偵測到 Primary 離線
    │
    ▼
腳本查詢 sys.gr_member_routing_candidate_status
找出可接管的 Secondary
    │
    ▼
更新 ProxySQL mysql_servers：
  - 原 Primary → Hostgroup 9999 (Offline)
  - 新 Primary → Hostgroup 10 (Writer)
LOAD MYSQL SERVERS TO RUNTIME
SAVE MYSQL SERVERS TO DISK
    │
    ▼
流量自動切換至新 Primary
```

### 節點恢復流程 (MGR)

```
1. 確認其他節點仍持續有資料寫入
        │
        ▼
2. 匯出完整資料（含 GTID）
   mysqldump --all-databases --triggers --routines --events --skip-lock-tables > all.sql
        │
        ▼
3. 將備份匯入已恢復的節點
        │
        ▼
4. 設定允許本地不相交（必要步驟）
   SET global group_replication_allow_local_disjoint_gtids_join=ON;
   SET GLOBAL binlog_format = 'ROW';
   START GROUP_REPLICATION;
        │
        ▼
5. 確認節點狀態為 ONLINE
   查詢 performance_schema.replication_group_members
        │
        ▼
6. 確認 ProxySQL mysql_servers 已自動更新
   SELECT * FROM mysql_servers;
```

---

## 使用方法

### 環境準備

**1. 建立 Docker 網路（首次使用）：**

```bash
docker network create mysql_network
```

**2. 建立環境變數檔：**

```bash
cp .env.example .env
# 編輯 .env，將所有 changeme 取代為實際密碼
```

**3. 建立 ProxySQL 設定檔：**

```bash
cp conf/proxysql.cnf.default conf/proxysql.cnf
# 編輯 conf/proxysql.cnf，將 CHANGE_ME 取代為實際密碼
# 並將 mysql_server1_ip / mysql_server2_ip 改為實際 MySQL 節點位址
```

---

### 啟動環境

```bash
# 啟動 ProxySQL + SQLite Web（主目錄）
docker compose up -d

# 啟動 MySQL 主從環境（Master-Slave 模式）
cd mysql_replication/
docker compose up -d

# 停止
docker compose down
```

> **SQLite Web（8080）** 已啟用密碼保護，密碼為 `.env` 中 `SQLITE_WEB_PASSWORD` 的值。

---

### 連接 ProxySQL

```bash
# 連接 ProxySQL 管理介面（Admin）
# 密碼為 conf/proxysql.cnf 中 admin_credentials 設定的值
mysql -uadmin -p -h127.0.0.1 -P6032 --prompt='ProxySQL> '

# 透過 ProxySQL 代理連接 MySQL（應用程式用）
mysql -uyour_username -pyour_password -h127.0.0.1 -P6033 --prompt='MySQL> '
```

### 初始化 ProxySQL 設定

**1. 建立 MySQL 使用者（在 MySQL Master 執行）：**

```sql
-- 監控帳號（ProxySQL 健康檢查用）
CREATE USER 'monitor'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON *.* TO 'monitor'@'%';

-- 應用程式帳號（透過 ProxySQL 連線用）
CREATE USER 'proxysql'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON *.* TO 'proxysql'@'%';

-- 主從複製帳號
CREATE USER 'replication'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';
FLUSH PRIVILEGES;
```

**2. 在 ProxySQL Admin 設定（6032）：**

```sql
-- 設定監控帳密
SET mysql-monitor_username='monitor';
SET mysql-monitor_password='password';
LOAD MYSQL VARIABLES TO RUNTIME;
SAVE MYSQL VARIABLES TO DISK;

-- 新增應用程式使用者
INSERT INTO mysql_users (username, password, active, default_hostgroup)
VALUES ('proxysql', 'password', 1, 1);
LOAD MYSQL USERS TO RUNTIME;
SAVE MYSQL USERS TO DISK;

-- 新增 MySQL 伺服器節點
INSERT INTO mysql_servers (hostgroup_id, hostname, port) VALUES (1, 'master', 3306);
INSERT INTO mysql_servers (hostgroup_id, hostname, port) VALUES (2, 'slave1', 3306);
LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;

-- 設定 Host Groups（主從模式）
INSERT INTO mysql_replication_hostgroups (writer_hostgroup, reader_hostgroup) VALUES (1, 2);
LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;

-- 設定路由規則（寫入 → Master，讀取 → Slave）
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply)
VALUES (1, 1, '^INSERT', 1, 1);
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply)
VALUES (2, 1, '^UPDATE', 1, 1);
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply)
VALUES (3, 1, '^SELECT.*FOR UPDATE$', 1, 1);
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply)
VALUES (4, 1, '^SELECT', 2, 1);
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply)
VALUES (5, 1, '.*', 1, 1);
LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL QUERY RULES TO DISK;
```

### MySQL 主從設定

**Master（取得 binlog 資訊）：**

```bash
docker exec -ti master mysql -uroot -p
```

```sql
SHOW MASTER STATUS;
-- 記錄 File（binlog 檔名）與 Position
```

**Slave（設定主從）：**

```bash
docker exec -ti slave1 mysql -uroot -p
```

```sql
CHANGE MASTER TO
  MASTER_HOST='master',
  MASTER_PORT=3306,
  MASTER_USER='replication',
  MASTER_PASSWORD='replicationpassword',
  MASTER_LOG_FILE='{binlog 檔名}',
  MASTER_LOG_POS={position};

START SLAVE;
SHOW SLAVE STATUS\G
```

### 負載平衡設定

ProxySQL 對同一 Hostgroup 內的多台節點依 `weight` 比例分配流量，可直接在 Reader Hostgroup 新增多台 Slave。

**新增多台 Slave 至 Reader Hostgroup：**

```sql
-- 連接 ProxySQL Admin（6032）後執行
INSERT INTO mysql_servers (hostgroup_id, hostname, port, weight, max_connections)
VALUES (2, 'slave1', 3306, 100, 200);

INSERT INTO mysql_servers (hostgroup_id, hostname, port, weight, max_connections)
VALUES (2, 'slave2', 3306, 100, 200);

INSERT INTO mysql_servers (hostgroup_id, hostname, port, weight, max_connections)
VALUES (2, 'slave3', 3306, 200, 200);
-- slave3 的 weight 為 200，流量為 slave1/slave2 的兩倍

LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```

**調整負載平衡演算法：**

```sql
-- 查看目前演算法（0=RANDOM 依 weight 隨機，1=LEAST_CONNECTIONS 優先送往連線數最少的節點）
SELECT * FROM global_variables WHERE variable_name = 'mysql-default_query_routing_algorithm';

-- 改為 LEAST_CONNECTIONS
SET mysql-default_query_routing_algorithm = 1;
LOAD MYSQL VARIABLES TO RUNTIME;
SAVE MYSQL VARIABLES TO DISK;
```

**暫時下線單台節點（維護用）：**

```sql
-- 將 slave2 設為 OFFLINE_SOFT（等現有連線結束後不再分配新連線）
UPDATE mysql_servers SET status='OFFLINE_SOFT' WHERE hostname='slave2';
LOAD MYSQL SERVERS TO RUNTIME;

-- 維護完畢後恢復
UPDATE mysql_servers SET status='ONLINE' WHERE hostname='slave2';
LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```

**驗證負載平衡是否生效：**

```sql
-- 1. 確認節點狀態（ConnUsed=目前連線數，Queries=已處理請求數）
SELECT hostgroup, srv_host, srv_port, status, weight, ConnUsed, Queries
FROM stats_mysql_connection_pool
WHERE hostgroup = 2;

-- 2. 查看各節點的流量分佈
SELECT hostgroup, srv_host, count_star, sum_time
FROM stats_mysql_query_digest_reset
GROUP BY hostgroup, srv_host;

-- 3. 實際測試：對 6033 執行多次 SELECT，觀察 server_id 是否輪替
-- （在應用程式或 mysql client 執行）
SELECT @@hostname, @@server_id;
```

> `Queries` 欄位統計各節點累計處理的請求數，若分配均勻則各節點數量應接近 weight 比例。

---

### gr_sw_mode_checker.sh 故障轉移腳本

用於 MGR 模式，持續監控節點健康並自動調整 ProxySQL 路由。

```bash
# 用法
./script/gr_sw_mode_checker.sh <writer_hostgroup_id> <reader_hostgroup_id> [write_can_read] [log_file]

# 範例：Writer=1, Reader=2，Writer 也可讀，記錄到 checker.log
./script/gr_sw_mode_checker.sh 1 2 1 ./logs/checker.log
```

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `writer_hostgroup_id` | 寫入分組 ID | 1 |
| `reader_hostgroup_id` | 讀取分組 ID | 2 |
| `write_can_read` | Writer 是否也可讀（1=是，0=否） | 1 |
| `log_file` | 日誌檔案路徑 | `./checker.log` |

> 腳本預設連接 `127.0.0.1:6032`，帳密為腳本內 `proxysql_username` / `proxysql_password` 變數，修改 ProxySQL 管理密碼後需同步更新腳本。

---

## 實際伺服器部署（Master-Slave）

本節說明將 ProxySQL 搭配外部實際 MySQL 主從伺服器（非 Docker 練習環境）的部署步驟與注意事項。

### 部署架構

```
Client
  │
  ▼
ProxySQL 主機（Docker 容器）
  ├── 6032：Admin 管理介面
  ├── 6033：MySQL 代理連線（應用程式使用）
  └── 6080：ProxySQL Web UI
        │
        ├──► HG 1（寫入）→ MySQL Master 主機  :3306
        └──► HG 2（讀取）→ MySQL Master :3306（weight=5）
                           MySQL Slave  :3306（weight=3）
```

> ProxySQL 透過主機名稱或 IP 連接外部 MySQL，兩者需在同一網路或有防火牆規則允許 3306 互通。

---

### 模板 vs 實際部署差異

從模板複製後**必須手動調整**的項目：

#### conf/proxysql.cnf

| 設定項 | 模板預設值 | 實際部署值 | 說明 |
|--------|-----------|-----------|------|
| `logfile` | `/logs/proxysql/proxysql.log` | `/var/log/proxysql.log` | 模板掛 volume，實際部署改用容器內路徑 |
| `admin_credentials` | `admin:CHANGE_ME;radmin:CHANGE_ME` | 實際密碼 | 必須替換強密碼 |
| `monitor_password` | `CHANGE_ME` | 實際密碼 | 必須替換強密碼 |
| `server_version` | `5.5.30` | 對應 MySQL 版本號 | MySQL 8 建議改為 `8.0.x` |

#### docker-compose.yml

| 項目 | 模板 | 實際部署 | 說明 |
|------|------|---------|------|
| `networks` 區塊 | 有 `mysql_network` (external) | 可移除或保留 | 實際部署視網路環境決定 |
| `phpmyadmin` ports | `"80:80"` | `"8081:80"` | 避免與其他服務衝突 |
| `sqlite-web` 密碼 | 有 `--password` | 可移除 | 視安全需求決定 |
| `phpmyadmin` restart | `restart: always` | 視需求 | 僅 proxysql 服務必要 |

#### conf/phpmyadmin/config.user.inc.php

實際部署須改為透過 ProxySQL 代理連線，而非直連 MySQL：

```php
$cfg['Servers'][1]['host'] = 'proxysql';
$cfg['Servers'][1]['port'] = '6033';
$cfg['Servers'][1]['verbose'] = 'proxysql';
$cfg['DefaultServer'] = 1;
```

---

### Step 1：MySQL Master 準備

在 **MySQL Master 主機**執行，建立 ProxySQL 所需帳號：

**MySQL 5.7：**

```sql
CREATE USER 'monitor'@'%' IDENTIFIED BY '<MYSQL_MONITOR_PASSWORD>';
GRANT ALL PRIVILEGES ON *.* TO 'monitor'@'%';

CREATE USER 'proxysql'@'%' IDENTIFIED BY '<MYSQL_PROXYSQL_PASSWORD>';
GRANT ALL PRIVILEGES ON *.* TO 'proxysql'@'%';

CREATE USER 'replication'@'%' IDENTIFIED BY '<MYSQL_REPLICATION_PASSWORD>';
GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';

FLUSH PRIVILEGES;
```

**MySQL 8（必須指定 `mysql_native_password`）：**

```sql
-- MySQL 8 預設認證插件為 caching_sha2_password，ProxySQL 不支援
-- 建立帳號時必須指定 mysql_native_password，否則監控與連線會失敗
CREATE USER 'monitor'@'%' IDENTIFIED WITH mysql_native_password BY '<MYSQL_MONITOR_PASSWORD>';
GRANT ALL PRIVILEGES ON *.* TO 'monitor'@'%';

CREATE USER 'proxysql'@'%' IDENTIFIED WITH mysql_native_password BY '<MYSQL_PROXYSQL_PASSWORD>';
GRANT ALL PRIVILEGES ON *.* TO 'proxysql'@'%';

CREATE USER 'replication'@'%' IDENTIFIED WITH mysql_native_password BY '<MYSQL_REPLICATION_PASSWORD>';
GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';

FLUSH PRIVILEGES;
```

取得 Master binlog 位置（設定 Slave 時使用）：

```sql
SHOW MASTER STATUS;
-- 記錄 File（binlog 檔名）與 Position
```

---

### Step 2：MySQL Slave 準備

在 **MySQL Slave 主機**設定主從複製，指向 Master：

**MySQL 5.7：**

```sql
CHANGE MASTER TO
  MASTER_HOST='<master_hostname_or_ip>',
  MASTER_PORT=3306,
  MASTER_USER='replication',
  MASTER_PASSWORD='<MYSQL_REPLICATION_PASSWORD>',
  MASTER_LOG_FILE='<binlog 檔名>',
  MASTER_LOG_POS=<position>;

START SLAVE;
SHOW SLAVE STATUS\G
-- 確認 Slave_IO_Running: Yes 且 Slave_SQL_Running: Yes
```

**MySQL 8（語法不同）：**

```sql
CHANGE REPLICATION SOURCE TO
  SOURCE_HOST='<master_hostname_or_ip>',
  SOURCE_PORT=3306,
  SOURCE_USER='replication',
  SOURCE_PASSWORD='<MYSQL_REPLICATION_PASSWORD>',
  SOURCE_LOG_FILE='<binlog 檔名>',
  SOURCE_LOG_POS=<position>;

START REPLICA;
SHOW REPLICA STATUS\G
-- 確認 Replica_IO_Running: Yes 且 Replica_SQL_Running: Yes
```

> **注意**：Slave 的 `server-id` 必須與 Master 不同，且每個節點唯一。  
> Slave 建議設定 `read_only = ON`（或 `super_read_only = ON`），防止誤寫。

---

### Step 3：ProxySQL 主機初始化

在部署 ProxySQL 的主機上 clone 專案並複製設定檔範本：

```bash
git clone https://bitbucket.org/avnight/proxysql.git
cd proxysql

cp docker-compose.yml.default docker-compose.yml
cp conf/proxysql.cnf.default conf/proxysql.cnf
cp conf/phpmyadmin/config.user.inc.php.default conf/phpmyadmin/config.user.inc.php
mkdir -p logs/proxysql
```

---

### Step 4：調整設定檔

**conf/proxysql.cnf**（依「模板 vs 實際部署差異」修改）：

```ini
datadir="/var/lib/proxysql"
logfile="/var/log/proxysql.log"
pidfile="/var/run/proxysql/proxysql.pid"

admin_variables=
{
    admin_credentials="admin:<ADMIN_PASSWORD>;radmin:<RADMIN_PASSWORD>"
    mysql_ifaces="0.0.0.0:6032"
    web_enabled=true
    web_port=6080
}

mysql_variables=
{
    threads=4
    max_connections=2048
    server_version="8.0.36"
    monitor_username="monitor"
    monitor_password="<MONITOR_PASSWORD>"
    ...
}
```

**docker-compose.yml**（移除 `networks` 外部依賴，調整 port）：

```yaml
version: "3"
services:
  proxysql:
    build:
      context: ./docker
      dockerfile: Dockerfile.sqlite
    image: proxysql_sqlite
    container_name: proxysql
    hostname: proxysql
    volumes:
      - ./conf/proxysql.cnf:/etc/proxysql.cnf
      - ./data/proxysql:/var/lib/proxysql
      - ./logs:/logs/proxysql
    ports:
      - "6032:6032"
      - "6033:6033"
      - "6080:6080"
    restart: always

  sqlite-web:
    image: coleifer/sqlite-web
    container_name: sqlite-web
    ports:
      - "8080:8080"
    volumes:
      - ./data/proxysql:/data
    command: ["sqlite_web", "/data/proxysql.db", "--host", "0.0.0.0"]

  phpmyadmin:
    image: phpmyadmin/phpmyadmin
    container_name: phpmyadmin
    ports:
      - "8081:80"
    volumes:
      - ./conf/phpmyadmin/config.user.inc.php:/etc/phpmyadmin/config.user.inc.php
      - ./data/phpmyadmin/:/srv/phpmyadmin/
```

**conf/phpmyadmin/config.user.inc.php**：

```php
<?php
$cfg['Servers'][1]['host'] = 'proxysql';
$cfg['Servers'][1]['port'] = '6033';
$cfg['Servers'][1]['verbose'] = 'proxysql';
$cfg['DefaultServer'] = 1;
?>
```

---

### Step 5：啟動 ProxySQL

```bash
docker-compose up -d
docker-compose ps
# 確認 proxysql、sqlite-web、phpmyadmin 全部 running
```

---

### Step 6：設定後端節點與路由

連入 ProxySQL Admin：

```bash
mysql -uadmin -p<ADMIN_PASSWORD> -h127.0.0.1 -P6032 --prompt='ProxySQL> '
```

建立 Hostgroup 主從對應：

```sql
INSERT INTO mysql_replication_hostgroups (writer_hostgroup, reader_hostgroup, check_type)
VALUES (1, 2, 'read_only');
LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```

加入後端節點（以 master weight=5、slave weight=3 分配讀取流量）：

```sql
-- HG1：寫入，僅 master
INSERT INTO mysql_servers (hostgroup_id, hostname, port, weight, max_connections)
VALUES (1, '<master_hostname_or_ip>', 3306, 1, 10000);

-- HG2：讀取，master 高權重 + slave 低權重
INSERT INTO mysql_servers (hostgroup_id, hostname, port, weight, max_connections)
VALUES (2, '<master_hostname_or_ip>', 3306, 5, 10000);
INSERT INTO mysql_servers (hostgroup_id, hostname, port, weight, max_connections)
VALUES (2, '<slave_hostname_or_ip>', 3306, 3, 10000);

LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```

設定監控帳號：

```sql
SET mysql-monitor_username='monitor';
SET mysql-monitor_password='<MONITOR_PASSWORD>';
LOAD MYSQL VARIABLES TO RUNTIME;
SAVE MYSQL VARIABLES TO DISK;
```

設定應用連線帳號：

```sql
INSERT INTO mysql_users (username, password, active, default_hostgroup)
VALUES ('proxysql', '<PROXYSQL_PASSWORD>', 1, 1);
LOAD MYSQL USERS TO RUNTIME;
SAVE MYSQL USERS TO DISK;
```

設定讀寫分離路由規則：

> **注意**：rule_id 3（SELECT FOR UPDATE）必須在 rule_id 4（SELECT）之前，否則排他鎖查詢會被誤導至 Slave。

```sql
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply)
VALUES (1, 1, '^INSERT', 1, 1);
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply)
VALUES (2, 1, '^UPDATE', 1, 1);
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply)
VALUES (3, 1, '^SELECT.*FOR UPDATE$', 1, 1);
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply)
VALUES (4, 1, '^SELECT', 2, 1);
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply)
VALUES (5, 1, '.*', 1, 1);
LOAD MYSQL QUERY RULES TO RUNTIME;
SAVE MYSQL QUERY RULES TO DISK;
```

---

### Step 7：驗證

確認後端節點狀態（`ONLINE` 表示正常）：

```sql
SELECT hostgroup_id, hostname, port, status, weight FROM mysql_servers;
```

確認監控可正常連線：

```sql
SELECT * FROM monitor.mysql_server_connect_log ORDER BY time_start_us DESC LIMIT 10;
SELECT * FROM monitor.mysql_server_ping_log ORDER BY time_start_us DESC LIMIT 10;
-- error_msg 欄位應為 NULL，若有 Access denied 表示 monitor 帳號密碼有誤
```

確認路由規則：

```sql
SELECT rule_id, active, match_pattern, destination_hostgroup, apply
FROM mysql_query_rules ORDER BY rule_id;
```

測試讀寫分離（透過 ProxySQL 6033）：

```bash
mysql -uproxysql -p<PROXYSQL_PASSWORD> -h127.0.0.1 -P6033 --prompt='MySQL> '
```

```sql
-- 執行數次，觀察 hostname 是否在 master/slave 之間切換
SELECT @@hostname, @@server_id;

-- 查看路由命中分佈
SELECT hostgroup, digest_text, count_star
FROM stats_mysql_query_digest
ORDER BY count_star DESC LIMIT 10;
```

---

## 常用查詢

### 伺服器與使用者

```sql
-- 查看 ProxySQL 分組設定
SELECT * FROM mysql_replication_hostgroups;

-- 查看設定的 MySQL 伺服器
SELECT * FROM mysql_servers;

-- 查看設定的使用者
SELECT * FROM mysql_users;

-- 統計各 SQL 類型執行次數與時間
SELECT * FROM stats_mysql_commands_counters;

-- 查看連線池資訊
SELECT * FROM stats_mysql_connection_pool;

-- MySQL 代理全域統計
SELECT * FROM stats_mysql_global;
```

### 路由相關

```sql
-- 查看路由規則
SELECT rule_id, active, match_pattern, destination_hostgroup, apply
FROM mysql_query_rules;

-- 查看請求路由分佈
SELECT hostgroup, schemaname, username, digest_text, count_star
FROM stats_mysql_query_digest;

-- 最近 5 筆查詢統計（含執行時間）
SELECT
  hostgroup AS hg,
  count_star,
  FROM_UNIXTIME(first_seen) AS first_seen_time,
  sum_time,
  digest_text
FROM stats_mysql_query_digest
ORDER BY first_seen DESC
LIMIT 5;
```

### 監控相關

```sql
-- 查看 ping 監控日誌
SELECT * FROM monitor.mysql_server_ping_log
ORDER BY time_start_us DESC LIMIT 6;

-- 查看連線監控日誌
SELECT * FROM monitor.mysql_server_connect_log
ORDER BY time_start_us DESC LIMIT 6;

-- 查看 read_only 監控日誌
SELECT * FROM mysql_server_read_only_log LIMIT 10;

-- 查看 monitor 資料庫所有表
SHOW TABLES FROM monitor;
```

### MGR 群組狀態

```sql
-- 查看 MGR 各節點狀態與角色（在 MySQL 執行）
SELECT
    MEMBER_ID,
    MEMBER_HOST,
    MEMBER_PORT,
    MEMBER_STATE,
    IF(global_status.VARIABLE_NAME IS NOT NULL, 'PRIMARY', 'SECONDARY') AS MEMBER_ROLE
FROM performance_schema.replication_group_members
LEFT JOIN performance_schema.global_status
    ON global_status.VARIABLE_NAME = 'group_replication_primary_member'
    AND global_status.VARIABLE_VALUE = replication_group_members.MEMBER_ID;

-- 查看路由候選狀態（確認節點是否可作為路由目標）
SELECT * FROM sys.gr_member_routing_candidate_status;

-- 確認 ProxySQL 節點是否恢復（在 ProxySQL Admin 執行）
SELECT * FROM mysql_servers;
```

**`sys.gr_member_routing_candidate_status` 欄位說明：**

| 欄位 | 說明 |
|------|------|
| MEMBER_ID | 節點識別碼 |
| MEMBER_HOST | 節點主機名稱或 IP |
| MEMBER_PORT | 節點埠號 |
| MEMBER_STATE | 目前狀態（`ONLINE` / `OFFLINE` 等） |
| MEMBER_ROLE | 角色（`PRIMARY` / `SECONDARY`） |
| ROUTING_SUPPORT | 是否支援路由（`YES` / `NO`） |
| ROUTING_CANDIDATE_STATUS | 路由候選狀態（`ACTIVE` = 可用） |

---

## 建議注意事項

### 環境設定

- 啟動前依序執行：`docker network create mysql_network` → 複製 `.env.example` → 複製 `conf/proxysql.cnf.default`，填入實際密碼與 MySQL 節點位址
- `.env` 與 `conf/proxysql.cnf` 已加入 `.gitignore`，不會被提交至版本控制
- `docker-compose.yml` 使用外部網路 `mysql_network`，需先手動建立，否則 compose 會失敗
- **所有 `CHANGE_ME` 佔位符必須替換為強密碼後才能啟動**

### Master-Slave 模式

- `server-id` 在每個節點必須唯一，重複會導致複製失敗
- 設定 Slave 前需先從 Master 取得正確的 `MASTER_LOG_FILE` 與 `MASTER_LOG_POS`，取得後避免在 Master 執行其他寫入操作
- `SHOW SLAVE STATUS\G`（MySQL 5.7）或 `SHOW REPLICA STATUS\G`（MySQL 8）中 `Seconds_Behind_Master` / `Seconds_Behind_Source` 為複製延遲，若持續增大需排查原因
- Slave 建議設定 `read_only = ON`（或 `super_read_only = ON`），防止誤寫入 Slave 導致主從不一致
- 主從模式**無自動故障轉移**，Master 宕機需手動將 Slave 提升為新 Master 並更新 ProxySQL 設定
- **MySQL 8 認證插件**：MySQL 8 預設使用 `caching_sha2_password`，ProxySQL 目前不支援此插件。建立 `monitor`、`proxysql`、`replication` 等帳號時必須指定 `IDENTIFIED WITH mysql_native_password`，否則 ProxySQL 監控與連線會出現 `Authentication plugin 'caching_sha2_password' cannot be loaded` 錯誤
- **MySQL 8 語法變更**：MySQL 8.0.23+ 已廢棄 `CHANGE MASTER TO` / `START SLAVE` / `SHOW SLAVE STATUS`，改用 `CHANGE REPLICATION SOURCE TO` / `START REPLICA` / `SHOW REPLICA STATUS`
- 路由規則中 `^SELECT.*FOR UPDATE$` 必須排在 `^SELECT` 之前（rule_id 較小），否則排他鎖查詢會被誤導至 Slave，造成鎖等待或資料不一致

### MGR 模式

- 恢復宕機節點時，**必須先執行** `SET global group_replication_allow_local_disjoint_gtids_join=ON`，否則 ProxySQL 的健康視圖無法正常使用，導致 ProxySQL 無法辨識節點已恢復
- `gr_sw_mode_checker.sh` 腳本的 ProxySQL 連線帳密預設為 `admin:admin`，修改 ProxySQL 管理密碼後需同步修改腳本內的變數
- MGR 節點宕機後重新加入，建議先用 `mysqldump` 匯出最新資料再匯入，避免 GTID 不一致導致加入失敗

### ProxySQL 操作

- 所有設定變更需執行 `LOAD ... TO RUNTIME` 才會立即生效，執行 `SAVE ... TO DISK` 才能在重啟後持續有效，兩者需配合使用
- 修改設定後建議透過 `stats_mysql_connection_pool` 確認後端節點連線狀態正常
- SQLite Web（8080）提供 ProxySQL 資料庫的圖形化瀏覽，僅用於檢視，不建議直接修改

### 安全性

- 密碼管理：複製 `.env.example` → `.env`，在 `.env` 中設定所有密碼；`conf/proxysql.cnf` 中的密碼亦需一併修改，兩者需保持一致
- `monitor` 使用者的帳密明文儲存於 ProxySQL 設定中，正式環境請限制其權限（最小化原則）
- `proxysql.cnf` 與 `.env` 已加入 `.gitignore`，確保不會提交至版本控制
- 正式環境不應對外暴露 6032（Admin）埠號，建議改綁內網 IP 並透過防火牆限制來源
- SQLite Web（8080）已啟用密碼保護，正式環境仍建議限制存取來源或關閉此服務

---

## 參考資料

- [MySQL 工具 ProxySQL（高性能 高可用性的 MySQL 代理）](https://github.com/open222333/Other-Note/blob/main/03_%E4%BC%BA%E6%9C%8D%E5%99%A8%E6%9C%8D%E5%8B%99/DatabaseServer(%E8%B3%87%E6%96%99%E5%BA%AB%E4%BC%BA%E6%9C%8D%E5%99%A8)/MySQL/MySQL%20%E5%B7%A5%E5%85%B7%20ProxySQL(%E9%AB%98%E6%80%A7%E8%83%BD%20%E9%AB%98%E5%8F%AF%E7%94%A8%E6%80%A7%E7%9A%84%20MySQL%20%E4%BB%A3%E7%90%86).md)
- [官方 ProxySQL Docker Image](https://hub.docker.com/r/proxysql/proxysql)
- [ProxySQL 配置与高可用](https://www.yoyoask.com/?p=3560)
- [ZzzCrazyPig/proxysql_groupreplication_checker — 故障轉移腳本](https://github.com/ZzzCrazyPig/proxysql_groupreplication_checker)
- [proxysql_groupreplication_checker 說明（中文）](https://github.com/ZzzCrazyPig/proxysql_groupreplication_checker/blob/master/README_Chinese.md)
