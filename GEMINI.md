# Gemini 專案指南 (test_db)

本專案提供了一個大型範例資料庫（「employees」資料庫），包含約 300,000 筆員工記錄和 280 萬筆薪資分錄。它包含一套完整的測試套件，用於驗證不同資料庫引擎之間的資料完整性。

## 專案概述

-   **目的：** 提供非平凡（non-trivial）的資料集，用於測試資料庫伺服器、應用程式和 SQL 查詢。
-   **主要技術：** MySQL (5.0+)、PostgreSQL (12+)、Shell Script。
-   **資料來源：** 由 Siemens Corporate Research 原始建立的虛構資料。

## 架構與綱要 (Schema)

資料庫由六個主要資料表組成：
-   `employees`: 員工個人核心資料。
-   `departments`: 部門定義。
-   `dept_emp`: 員工與部門分配的關聯表（多對多）。
-   `dept_manager`: 部門主管的關聯表（多對多）。
-   `titles`: 員工職稱的歷史記錄（一對多）。
-   `salaries`: 員工薪資的歷史記錄（一對多）。

## 建置與執行

### MySQL

將資料庫載入 MySQL 伺服器：
```bash
# 標準安裝
mysql < employees.sql

# MySQL 9.5+（需要 --commands 旗標）
mysql --commands < employees.sql

# 分區版本（對 salaries 和 titles 使用 RANGE 分區）
mysql < employees_partitioned.sql
```

### PostgreSQL

將資料庫載入 PostgreSQL 伺服器：
```bash
cd postgresql
bash load_employees_db.sh
```

### 補充物件 (Views/Procedures)

載入檢視表、儲存函式和程序：
-   **MySQL:** `mysql < objects.sql`
-   **PostgreSQL:** `psql -d employees < postgresql/objects.sql`

## 測試與驗證

### 完整性測試

使用基於校驗和（checksum）的完整性測試來驗證安裝。建議使用 SHA-256 以確保跨版本的相容性。

-   **MySQL:**
    ```bash
    mysql -t < test_employees_sha2.sql   # SHA-256 (相容 9.6+)
    mysql -t < test_employees_md5.sql    # MD5 (僅限 MySQL 8.0-9.5)
    ```
-   **PostgreSQL:**
    ```bash
    psql -d employees < postgresql/test_employees_sha2.sql
    ```

### Shell 驗證 (僅限 MySQL)

`sql_test.sh` 腳本使用 `CHECKSUM TABLE` 進行快速驗證：
```bash
bash sql_test.sh "mysql -u root"
```

## 開發慣例

-   **SQL 相容性：** 在可能的情況下，保持 MySQL 和 PostgreSQL 綱要之間的功能一致性。
-   **資料完整性：** 對綱要或資料載入過程的任何更改，都必須針對 `test_employees_*.sql` 檔案中的預期校驗和進行驗證。
-   **命名規範：** 遵循既有的全小寫資料表和欄位名稱（例如 `emp_no`, `first_name`）。
-   **儲存邏輯：** MySQL 程序使用 `DELIMITER //`，而 PostgreSQL 等效程式則使用帶有錢字號引用（dollar-quoting）的 PL/pgSQL。

## MySQL 版本矩陣

| 版本 | 需要 `--commands` | 支援 MD5/SHA |
| :--- | :--- | :--- |
| < 9.5 | 否 | 是 |
| 9.5 | 是 | 是 |
| 9.6+ | 是 | 否 (請使用 SHA2) |
| 10.x+ | 是 | 否 (請使用 SHA2) |
