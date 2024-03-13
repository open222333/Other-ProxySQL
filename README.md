# Other-ProxySQL

```
主從高可用
```

## 目錄

- [Other-ProxySQL](#other-proxysql)
  - [目錄](#目錄)
  - [參考資料](#參考資料)
    - [教學相關](#教學相關)
    - [腳本相關](#腳本相關)
- [基本操作](#基本操作)
  - [透過 ProxySQL 連接到已設定的 MySQL 伺服器](#透過-proxysql-連接到已設定的-mysql-伺服器)
- [常用查詢](#常用查詢)
  - [路由相關](#路由相關)
  - [查看監控](#查看監控)
- [測試意外宕機，故障轉移 (MGR)](#測試意外宕機故障轉移-mgr)
  - [恢復](#恢復)
    - [保持資料完整](#保持資料完整)
    - [啟動組](#啟動組)
    - [查看 各表格是否自動恢復](#查看-各表格是否自動恢復)

## 參考資料

[MySQL 工具 ProxySQL(高性能 高可用性的 MySQL 代理)](https://github.com/open222333/Other-Note/blob/main/03_%E4%BC%BA%E6%9C%8D%E5%99%A8%E6%9C%8D%E5%8B%99/DatabaseServer(%E8%B3%87%E6%96%99%E5%BA%AB%E4%BC%BA%E6%9C%8D%E5%99%A8)/MySQL/MySQL%20%E5%B7%A5%E5%85%B7%20ProxySQL(%E9%AB%98%E6%80%A7%E8%83%BD%20%E9%AB%98%E5%8F%AF%E7%94%A8%E6%80%A7%E7%9A%84%20MySQL%20%E4%BB%A3%E7%90%86).md)

[官方 ProxySQL Docker Image](https://hub.docker.com/r/proxysql/proxysql)

### 教學相關

[ProxySQL配置与高可用](https://www.yoyoask.com/?p=3560)

### 腳本相關


[ZzzCrazyPig/proxysql_groupreplication_checker - 設定proxysql故障轉移 腳本](https://github.com/ZzzCrazyPig/proxysql_groupreplication_checker)

[ZzzCrazyPig/proxysql_groupreplication_checker - 設定proxysql故障轉移 腳本說明](https://github.com/ZzzCrazyPig/proxysql_groupreplication_checker/blob/master/README_Chinese.md)

# 基本操作

## 透過 ProxySQL 連接到已設定的 MySQL 伺服器

ProxySQL 的管理端口（6032）

ProxySQL 的 MySQL 連接端口（通常是 6033）

--prompt 定制提示符的外觀和格式

```bash
mysql -uyour_username -pyour_password -h127.0.0.1 -P6032 --prompt='ProxySQL> '
```

```bash
mysql -uyour_username -pyour_password -h127.0.0.1 -P6033 --prompt='MySQL> '
```

MySQL master cnf

```ini
server-id = 1
log-bin = mysql-master-bin
```

MySQL slave cnf

```ini
[mysqld]
server-id = 2
log-bin = mysql-slave1-bin
```

MySQL 新增使用者

```sql
CREATE USER 'monitor'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON *.* TO 'monitor'@'%';
CREATE USER 'proxysql'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON *.* TO 'proxysql'@'%';
CREATE USER 'replication'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';
FLUSH PRIVILEGES;
```

ProxySQL 新增使用者

```sql
INSERT INTO mysql_users (username, password, active, default_hostgroup)
VALUES ('proxysql', 'password', 1, 1);
LOAD MYSQL USERS TO RUNTIME;
SAVE MYSQL USERS TO DISK;
```

ProxySQL 設置監控 MySQL 後端節點

```sql
SET mysql-monitor_username='monitor';
SET mysql-monitor_password='password';
LOAD MYSQL VARIABLES TO RUNTIME;
SAVE MYSQL VARIABLES TO DISK;
```

ProxySQL 新增路由

```sql
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply)
VALUES (1, 1, '^SELECT', 2, 1);
INSERT INTO mysql_query_rules (rule_id, active, match_pattern, destination_hostgroup, apply)
VALUES (2, 1, '.*', 1, 1);
```

# 常用查詢

`查看 ProxySQL 分組`

```sql
SELECT * FROM mysql_replication_hostgroups;
```

`查看 設定的 mysql servers`

```sql
SELECT * FROM mysql_servers;
```

`查看 設定的 mysql users`

```sql
SELECT * FROM mysql_users;
```

`統計各種SQL類型的執行次數與時間`

```sql
SELECT * FROM stats_mysql_commands_counters;
```

`查看連接後端MySQL的連接池信息`

```sql
SELECT * FROM stats_mysql_connection_pool;
```

`與MySQL相關的代理程式級別的全域統計`

```sql
SELECT * FROM stats_mysql_global;
```

## 路由相關

`查看路由規則表`

```sql
SELECT rule_id,active,match_pattern,destination_hostgroup,apply
FROM mysql_query_rules;
```

`統計路由命中次數`

```sql
SELECT * FROM stats_mysql_processlist;
```

`儲存monitor模組收集的信息，主要是對後端db的健康/延遲檢查`

`查看monitor資料庫中的表`

```sql
SHOW tables FROM monitor;
```

`查看請求路由資訊`

```sql
SELECT hostgroup,schemaname,username,digest_text,count_star FROM stats_mysql_query_digest;
```

`從 stats_mysql_query_digest 表中選擇一些關於查詢摘要統計資料的字段，並按 first_seen 降序排列，然後限制結果集的數量為 5 個。`

```sql
SELECT
  hostgroup AS hg,
  count_star,
  FROM_UNIXTIME(first_seen) AS first_seen_time,
  sum_time,
  digest,
  digest_text
FROM
  stats_mysql_query_digest
ORDER BY
  first_seen DESC
LIMIT 5;
```

hostgroup：查詢所屬的主機群組。

count_star：查詢執行的次數。

FROM_UNIXTIME(first_seen)：查詢首次執行的時間，已轉換為日期時間格式。

sum_time：查詢總共執行的時間（單位可能是微秒）。

digest：查詢摘要的雜湊值。

digest_text：查詢語句的文字形式。

## 查看監控

`檢查連接到MySQL的日誌`

```sql
SELECT * FROM monitor.mysql_server_ping_log
ORDER BY time_start_us
DESC LIMIT 6;
```

```sql
SELECT * FROM monitor.mysql_server_connect_log
ORDER BY time_start_us
DESC LIMIT 6;
```

`查看read_only的日誌監控`

```sql
SELECT * FROM mysql_server_read_only_log LIMIT 10;
```

# 測試意外宕機，故障轉移 (MGR)

`在 mysql 中查看目前group群組情況`

```sql
SELECT
    MEMBER_ID,
    MEMBER_HOST,
    MEMBER_PORT,
    MEMBER_STATE,
    IF(global_status.VARIABLE_NAME IS NOT NULL,
        'PRIMARY',
        'SECONDARY') AS MEMBER_ROLE
FROM
    performance_schema.replication_group_members
        LEFT JOIN
    performance_schema.global_status ON global_status.VARIABLE_NAME = 'group_replication_primary_member'
        AND global_status.VARIABLE_VALUE = replication_group_members.MEMBER_ID;
```

`查看 sql 請求路由訊息`

```sql
SELECT hostgroup,schemaname,username,digest_text,count_star
FROM stats_mysql_query_digest;
```

## 恢復

### 保持資料完整

如果其他節點不斷有資料寫入

因為之前宕機的是主(Master)

再重新加入群組要以 slave 的身份

reset master 以 slave 身份仍然不能正常加入

匯出一份完整數據(帶gtid的) 匯入當機的 mysql 後 直接啟動組

```bash
mysqldump -uroot -p --all-databases --triggers --routines --events --skip-lock-tables > all.sql
```

### 啟動組

注意這個允許本地不想交的指令必須執行

否則proxysql之前的查詢主從節點健康的視圖你會無法使用

自然proxysql就無法辨識已經恢復正常

```sql
-- 群組複製允許本地不相交
SET global group_replication_allow_local_disjoint_gtids_join=ON;
SET SESSION binlog_format = 'ROW';
SET GLOBAL binlog_format = 'ROW';
START GROUP_REPLICATION;
```

`查看 MGR 組狀態`

```sql
SELECT
    MEMBER_ID,
    MEMBER_HOST,
    MEMBER_PORT,
    MEMBER_STATE,
    IF(global_status.VARIABLE_NAME IS NOT NULL,
        'PRIMARY',
        'SECONDARY') AS MEMBER_ROLE
FROM
    performance_schema.replication_group_members
        LEFT JOIN
    performance_schema.global_status ON global_status.VARIABLE_NAME = 'group_replication_primary_member'
        AND global_status.VARIABLE_VALUE = replication_group_members.MEMBER_ID;
```

### 查看 各表格是否自動恢復

`sys.gr_member_routing_candidate_status` 表是MySQL Group Replication 中的一個系統表，用於提供有關Group Replication 成員（節點）的資訊。

以下是對該資料表的查詢以及欄位的說明：

```sql
SELECT * FROM sys.gr_member_routing_candidate_status;
```

此查詢傳回有關Group Replication 成員的目前狀態和路由候選狀態的資訊。

`sys.gr_member_routing_candidate_status` 表的主要欄位說明：

- **MEMBER_ID：** Group Replication 成員的識別碼。

- **MEMBER_HOST：** 成員的主機名稱或IP 位址。

- **MEMBER_PORT：** 成員的連接埠號碼。

- **MEMBER_STATE：** 成員的目前狀態，例如'ONLINE' 表示在線。

- **MEMBER_ROLE：** 成員的角色，例如'PRIMARY' 表示主節點。

- **ROUTING_SUPPORT：** 成員是否支援路由功能，如果支持，值為'YES'。

- **ROUTING_CANDIDATE_STATUS：** 成員在路由中的候選狀態，例如'ACTIVE' 表示是活躍的候選者。

這個表的查詢結果將顯示Group Replication 成員的詳細信息，包括其當前狀態和是否支援路由功能。
候選狀態可以用於了解成員是否可以用作路由目標。
請注意，使用這個表的查詢需要確保使用者俱有執行相關查詢的權限，而MySQL 版本必須支援`sys` schema 和Group Replication。

`查看 proxysql 節點是否恢復`

```sql
SELECT * FROM mysql_servers;
```
