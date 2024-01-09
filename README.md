# Other-ProxySQL

```
我的專案樣本
```

## 目錄

- [Other-ProxySQL](#other-proxysql)
  - [目錄](#目錄)
  - [參考資料](#參考資料)
- [配置文檔](#配置文檔)
- [用法](#用法)

## 參考資料

[MySQL 工具 ProxySQL(高性能 高可用性的 MySQL 代理)](https://github.com/open222333/Other-Note/blob/main/03_%E4%BC%BA%E6%9C%8D%E5%99%A8%E6%9C%8D%E5%8B%99/DatabaseServer(%E8%B3%87%E6%96%99%E5%BA%AB%E4%BC%BA%E6%9C%8D%E5%99%A8)/MySQL/MySQL%20%E5%B7%A5%E5%85%B7%20ProxySQL(%E9%AB%98%E6%80%A7%E8%83%BD%20%E9%AB%98%E5%8F%AF%E7%94%A8%E6%80%A7%E7%9A%84%20MySQL%20%E4%BB%A3%E7%90%86).md)

[官方 ProxySQL Docker Image](https://hub.docker.com/r/proxysql/proxysql)

# 配置文檔

`官方 dockerhub 範例`

```ini
datadir="/var/lib/proxysql"

admin_variables=
{
	admin_credentials="admin:admin;radmin:radmin"
	mysql_ifaces="0.0.0.0:6032"
}

mysql_variables=
{
	threads=4
	max_connections=2048
	default_query_delay=0
	default_query_timeout=36000000
	have_compress=true
	poll_timeout=2000
	interfaces="0.0.0.0:6033"
	default_schema="information_schema"
	stacksize=1048576
	server_version="5.5.30"
	connect_timeout_server=3000
	monitor_username="monitor"
	monitor_password="monitor"
	monitor_history=600000
	monitor_connect_interval=60000
	monitor_ping_interval=10000
	monitor_read_only_interval=1500
	monitor_read_only_timeout=500
	ping_interval_server_msec=120000
	ping_timeout_server=500
	commands_stats=true
	sessions_sort=true
	connect_retries_on_failure=10
}
```

```ini
# 基本設定
datadir="/var/lib/proxysql"  # ProxySQL 數據目錄
logfile="/var/log/proxysql.log"  # ProxySQL 日誌文件
pidfile="/var/run/proxysql/proxysql.pid"  # ProxySQL PID 文件
admin_variables= {
    admin_credentials="admin:adminadmin"  # 管理員憑據
    mysql_ifaces="0.0.0.0:6032"  # MySQL 接口地址和端口
    refresh_interval=2000  # 管理器刷新間隔（毫秒）
    web_enabled=true  # 啟用 Web 界面
    web_port=6080  # Web 界面端口
    web_user=admin  # Web 界面用戶名
    web_passwd=admin  # Web 界面密碼
}

# MySQL 伺服器組配置
# 默認連接池配置
mysql_servers = (
    {
        hostgroup_id=10,  # 默認 Hostgroup ID
        hostname="mysql_server1_ip",  # 默認連接的 MySQL 伺服器 IP 地址
        port=3306,  # 默認連接的 MySQL 伺服器端口
        max_connections=100,  # 最大連接數
        weight=100  # 默認權重
    },
    {
        hostgroup_id=20,  # 默認 Hostgroup ID
        hostname="mysql_server2_ip",  # 默認連接的 MySQL 伺服器 IP 地址
        port=3306,  # 默認連接的 MySQL 伺服器端口
        max_connections=100,  # 最大連接數
        weight=100  # 默認權重
    },
)

# MySQL 伺服器組配置
mysql_groups = (
    {
        writer_hostgroup=10,  # 寫入操作的 Hostgroup
        backup_hostgroup=20,  # 備份操作的 Hostgroup
        reader_hostgroup=30,  # 讀取操作的 Hostgroup
        offline_hostgroup=9999,  # 下線操作的 Hostgroup
        max_writers=1,  # 最大寫入數
        writer_is_also_reader=1  # 寫入操作是否同時是讀取操作
    },
)

# 監聽端口配置
mysql_variables = (
    {
        variable_name="admin_variables.admin_credentials",  # 參數名稱
        variable_value="admin:adminadmin"  # 參數值
    },
)

# 查詢攔截和重寫配置
; 這個查詢規則的作用是，當有 SQL 查詢匹配正則表達式 ^SELECT.*FOR UPDATE$ 時，將該查詢發送到 Hostgroup 20。
; 這可能用於特定類型的查詢進行路由或處理。
mysql_query_rules = (
    {
        rule_id=1,  # 規則 ID
        match_digest="^SELECT.*FOR UPDATE$",  # 匹配的 SQL 語句
        destination_hostgroup=20,  # 目標 Hostgroup
        apply=1  # 是否應用此規則
    },
)
```

# 用法

```bash
# admin 只能從本機
mysql -uradmin -p -h127.0.0.1 -P6032
```
