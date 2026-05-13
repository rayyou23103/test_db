# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 專案簡介

這是一個員工範例資料庫（約 30 萬筆員工、280 萬筆薪資記錄），內含完整性測試套件，支援 MySQL 5.0+ 與 PostgreSQL 12+，用於測試資料庫伺服器與應用程式。

## 載入資料庫

### MySQL

```bash
# 標準安裝
mysql < employees.sql

# MySQL 9.5+ 需加 --commands 旗標
mysql --commands < employees.sql

# 分區版本（salaries 與 titles 表使用 RANGE 分區）
mysql < employees_partitioned.sql
```

### PostgreSQL

```bash
cd postgresql
bash load_employees_db.sh

# 指定自訂 psql（例如 dbdeployer sandbox）
PSQL=/path/to/psql bash load_employees_db.sh
```

## 完整性測試

建議優先使用 `sha2`，相容所有版本（包含 MySQL 9.6+）：

```bash
# MySQL
mysql -t < test_employees_sha2.sql   # SHA-256 — 所有版本皆可用
mysql -t < test_employees_md5.sql    # MD5 — 僅 MySQL 8.0–9.5
mysql -t < test_employees_sha.sql    # SHA-1 — 僅 MySQL 8.0–9.5

# PostgreSQL
psql -d employees < postgresql/test_employees_sha2.sql
```

測試輸出每張表的 `records_match` 與 `crc_match`，正確時均顯示 `OK`。MySQL 與 PostgreSQL 的 SHA-256 checksum 值相同。

`sql_test.sh` 是另一種使用 `CHECKSUM TABLE` 的 shell 式驗證工具：

```bash
bash sql_test.sh "mysql -u root"
```

## 預存程序 / 函式

```bash
# MySQL
mysql < objects.sql

# PostgreSQL
psql -d employees < postgresql/objects.sql
```

可用函式：`emp_name()`、`emp_dept_name()`、`emp_dept_id()`、`current_manager()`、`show_departments()`、`employees_help()`。  
MySQL 呼叫：`CALL show_departments();`；PostgreSQL 呼叫：`SELECT * FROM show_departments();`

## 資料庫結構

六張表：`employees`、`departments`、`dept_emp`、`dept_manager`、`titles`、`salaries`。  
關係：`employees` → `dept_emp` → `departments`（透過 dept_emp/dept_manager 形成多對多）；`employees` → `titles`、`salaries`（一對多，每筆以日期區間記錄）。

## MySQL 版本相容性

| 版本 | 需要 `--commands` | MD5/SHA 可用 |
|------|-------------------|--------------|
| < 9.5 | 否              | 是           |
| 9.5   | 是              | 是           |
| 9.6+  | 是              | 否（改用 SHA2）|

## PostgreSQL 與 MySQL 的差異

- `ENUM('M','F')` → `CHAR(1) CHECK (gender IN ('M','F'))`
- 預存程序使用 dollar-quoting（`$$...$$`）而非 `DELIMITER //`
- `show_departments()` 是回傳 TABLE 的函式，非 procedure
- 完整性測試透過 PL/pgSQL helper 函式實作；SHA/MD5 測試需要 `pgcrypto` 擴充套件

## CI

`.github/workflows/` 下有四個工作流程（`ci-mysql.yml`、`ci-percona.yml`、`ci-mariadb.yml`、`ci-postgresql.yml`），使用 [dbdeployer](https://github.com/ProxySQL/dbdeployer) 建立 sandbox 實例，在每次 push、PR 及每週定期執行。
