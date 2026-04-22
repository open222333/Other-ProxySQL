# Other-ProxySQL

```
ProxySQL + MySQL 主從高可用練習環境
```

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
- `SHOW SLAVE STATUS\G` 中 `Seconds_Behind_Master` 為複製延遲，若持續增大需排查原因
- 主從模式**無自動故障轉移**，Master 宕機需手動將 Slave 提升為新 Master 並更新 ProxySQL 設定

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
