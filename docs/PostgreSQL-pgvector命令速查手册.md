# PostgreSQL / pgvector 命令速查手册（全量版）

> 适用版本：PostgreSQL 14–18，pgvector 0.7 / 0.8  
> 本机验证环境：PostgreSQL 17.11 + pgvector 0.8.7，Docker 容器 `pgvector`，数据库 `vectordb`  
> 配套文档：`Windows-Docker-pgvector-安装指南.md`、`PostgreSQL-Docker-安装与SpringBoot对接指南.md`  
> 整理日期：2026-10-18

本文把 PostgreSQL 日常开发、SQL、权限、运维、备份恢复和 pgvector 向量检索命令整合到一份文档里。示例默认使用：

```text
容器名：pgvector
数据库：vectordb
用户：postgres
宿主机端口：5432
```

> 约定：`[可选]` 表示可省略；`<必填>` 表示需要替换成实际值。所有 SQL 都要以分号 `;` 结尾。

---

## 目录

1. 连接数据库与容器
2. psql 元命令
3. 数据类型
4. 数据库操作
5. Schema 操作
6. 角色、用户与权限
7. 表操作（DDL）
8. 约束
9. 索引
10. 数据操作（DML）
11. 查询语法
12. 事务、锁与并发
13. 视图、物化视图、序列
14. 函数、存储过程、触发器
15. 运维监控
16. 配置与调优
17. 备份、恢复与迁移
18. 扩展管理
19. pgvector 向量检索
20. 常见报错与 SQLSTATE
21. 一页速查

---

## 一、连接数据库与容器

### 1.1 确认容器状态

```powershell
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Ports}}\t{{.Status}}"
docker ps -a --filter "name=pgvector"
```

### 1.2 进入 psql（最常用）

```powershell
docker exec -it pgvector psql -U postgres -d vectordb
```

| 参数 | 含义 |
|---|---|
| `-i` | 保持标准输入打开 |
| `-t` | 分配伪终端，交互式操作必需 |
| `-U` | 数据库用户 |
| `-d` | 数据库名 |

退出：

```sql
\q
```

### 1.3 不进容器，单条执行

```powershell
docker exec pgvector psql -U postgres -d vectordb -c "SELECT version();"
docker exec pgvector psql -U postgres -d vectordb -c "\dt"
```

执行多条 SQL：

```powershell
docker exec pgvector psql -U postgres -d vectordb -c "SELECT now();" -c "SELECT current_user;"
```

### 1.4 执行 SQL 文件

文件在 Windows 宿主机：

```powershell
docker cp .\init.sql pgvector:/tmp/init.sql
docker exec -it pgvector psql -U postgres -d vectordb -f /tmp/init.sql
```

或者通过标准输入：

```powershell
Get-Content .\init.sql -Raw | docker exec -i pgvector psql -U postgres -d vectordb
```

带错误即停：

```powershell
docker exec -it pgvector psql -U postgres -d vectordb -v ON_ERROR_STOP=1 -f /tmp/init.sql
```

### 1.5 进入容器 shell

```powershell
docker exec -it pgvector bash
```

容器内常用：

```bash
psql -U postgres -d vectordb
pg_isready -U postgres -d vectordb
pg_config --version
psql --version
```

### 1.6 宿主机直接连接（装了 psql 客户端时）

```powershell
psql -h localhost -p 5432 -U postgres -d vectordb
```

连接串：

```text
postgresql://postgres:postgres@localhost:5432/vectordb
postgresql://postgres:postgres@localhost:5432/vectordb?sslmode=disable
```

JDBC：

```text
jdbc:postgresql://localhost:5432/vectordb
jdbc:postgresql://localhost:5432/vectordb?currentSchema=public&ApplicationName=ai-rag
```

### 1.7 psql 常用启动参数

| 参数 | 作用 |
|---|---|
| `-h <host>` | 主机 |
| `-p <port>` | 端口 |
| `-U <user>` | 用户 |
| `-d <db>` | 数据库 |
| `-c "<sql>"` | 执行一条 SQL 后退出 |
| `-f <file>` | 执行文件 |
| `-l` | 列出数据库 |
| `-v name=value` | 设置 psql 变量 |
| `-X` | 不读取 `.psqlrc` |
| `-q` | 安静模式 |
| `-A` | 非对齐输出，适合脚本 |
| `-t` | 只输出元组，不输出表头/行数 |
| `-F <sep>` | 字段分隔符 |
| `--csv` | CSV 输出 |
| `-W` | 强制输入密码 |
| `--single-transaction` | 把 `-f` 文件包在一个事务里 |
| `--set=ON_ERROR_STOP=1` | 出错立即停止 |

脚本化导出示例：

```powershell
docker exec pgvector psql -U postgres -d vectordb -At -F "," -c "SELECT id, username FROM app_user;" > users.csv
```

### 1.8 密码与环境变量

临时免交互：

```powershell
$env:PGPASSWORD = "postgres"
psql -h localhost -p 5432 -U postgres -d vectordb
Remove-Item Env:PGPASSWORD
```

Linux / WSL 使用 `.pgpass`：

```text
localhost:5432:vectordb:postgres:postgres
```

```bash
chmod 600 ~/.pgpass
```

Windows 的密码文件默认位置：

```text
%APPDATA%\postgresql\pgpass.conf
```

常用环境变量：

| 变量 | 作用 |
|---|---|
| `PGHOST` | 主机 |
| `PGPORT` | 端口 |
| `PGUSER` | 用户 |
| `PGDATABASE` | 数据库 |
| `PGPASSWORD` | 密码，不建议写进脚本 |
| `PGSSLMODE` | `disable` / `require` / `verify-full` |
| `PGCONNECT_TIMEOUT` | 连接超时秒数 |
| `PGAPPNAME` | 应用名，出现在 `pg_stat_activity` |
| `PGOPTIONS` | 连接时附加参数 |

### 1.9 连接健康检查

```powershell
docker exec pgvector pg_isready -U postgres -d vectordb
docker exec pgvector pg_isready -h 127.0.0.1 -p 5432
Test-NetConnection localhost -Port 5432
```

容器内查看监听端口：

```bash
ss -lntp | grep 5432
```

### 1.10 图形客户端连接参数

| 项 | 值 |
|---|---|
| Host | `localhost` |
| Port | 宿主机映射端口，本机为 `5432` |
| Database | `vectordb` |
| User | 你的 `POSTGRES_USER` |
| Password | 你的 `POSTGRES_PASSWORD` |
| SSL | 本地 Docker 一般关闭 |

---

## 二、psql 元命令

元命令以反斜杠开头，不需要分号。

### 2.1 查看类

| 命令 | 作用 |
|---|---|
| `\l` | 列出所有数据库 |
| `\l+` | 数据库列表，含大小和描述 |
| `\c <db>` | 切换数据库 |
| `\c <db> <user>` | 切换数据库和用户 |
| `\conninfo` | 当前连接信息 |
| `\dt` | 当前 schema 的表 |
| `\dt *.*` | 所有 schema 的表 |
| `\dt+` | 表列表，含大小和描述 |
| `\dt <pattern>` | 按名称过滤，如 `\dt app.*` |
| `\d <table>` | 表结构：列、索引、约束 |
| `\d+ <table>` | 更详细，含存储、压缩、描述 |
| `\ds` | 序列 |
| `\dv` | 视图 |
| `\dm` | 物化视图 |
| `\di` | 索引 |
| `\di+ <index>` | 索引大小和定义 |
| `\dn` | schema 列表 |
| `\dn+` | schema 列表，含 owner 和权限 |
| `\du` | 角色 / 用户 |
| `\du+` | 角色详细信息 |
| `\dg` | 角色组 |
| `\dp` | 表、视图、序列的权限 |
| `\dp <table>` | 指定对象权限 |
| `\ddp` | 默认权限 |
| `\df` | 函数列表 |
| `\df+ <function>` | 函数定义和属性 |
| `\dx` | 已安装扩展 |
| `\dx+ <ext>` | 扩展包含的对象 |
| `\dT` | 数据类型 |
| `\dT+` | 数据类型详情 |
| `\do` | 运算符 |
| `\do+` | 运算符详情 |
| `\z` | 同 `\dp` |
| `\sf <function>` | 查看函数源码 |
| `\sv <view>` | 查看视图定义 |
| `\s` | 命令历史 |

### 2.2 操作类

| 命令 | 作用 |
|---|---|
| `\x` | 切换纵向/横向显示 |
| `\x auto` | 自动选择显示方式 |
| `\timing` | 开关执行耗时 |
| `\timing on` | 开启耗时 |
| `\e` | 用编辑器编辑当前 SQL |
| `\ef <func>` | 编辑函数 |
| `\ev <view>` | 编辑视图 |
| `\i <file>` | 执行 SQL 文件 |
| `\ir <file>` | 执行文件，路径相对于当前脚本 |
| `\o <file>` | 后续查询结果输出到文件 |
| `\o` | 恢复输出到终端 |
| `\copy ...` | 客户端导入导出 CSV |
| `\watch <秒>` | 每隔 N 秒重复执行上一条 SQL |
| `\g` | 执行当前缓冲区 |
| `\gx` | 纵向执行并显示 |
| `\gset` | 把结果存入 psql 变量 |
| `\gexec` | 把查询结果的每一行当作 SQL 执行 |
| `\r` | 清空当前查询缓冲区 |
| `\p` | 打印当前缓冲区 |
| `\encoding` | 查看/设置客户端编码 |
| `\! <cmd>` | 执行 shell 命令 |
| `\cd <dir>` | 切换工作目录 |
| `\password [user]` | 交互式修改密码 |
| `\q` | 退出 |

### 2.3 帮助

| 命令 | 作用 |
|---|---|
| `\?` | 所有元命令帮助 |
| `\h` | SQL 语法帮助列表 |
| `\h CREATE TABLE` | 具体语法帮助 |
| `\h ALTER TABLE` | 查看 ALTER TABLE 语法 |

### 2.4 psql 变量

```sql
\set tenant_id 1001
SELECT * FROM orders WHERE tenant_id = :tenant_id;
\set ON_ERROR_STOP on
\set VERBOSITY verbose
\set AUTOCOMMIT off
```

把查询结果存入变量：

```sql
SELECT id AS user_id FROM app_user WHERE username = 'alice' \gset
SELECT :'user_id';
```

### 2.5 输出格式

```sql
\x
\x auto
\pset border 2
\pset null '(NULL)'
\pset format aligned
\pset format unaligned
\pset format csv
\pset footer off
\timing on
```

### 2.6 常见提示符

| 提示符 | 含义 |
|---|---|
| `vectordb=#` | 正常，等待 SQL |
| `vectordb-#` | 续行，上一条语句没结束 |
| `vectordb(#` | 括号没闭合 |
| `vectordb'#` | 单引号没闭合 |
| `vectordb"#` | 双引号没闭合 |
| `vectordb$#` | dollar-quote 没闭合 |

续行状态按 `Ctrl+C` 或输入 `\r` 清空缓冲区。

---

## 三、数据类型

### 3.1 常用类型总表

| 类别 | 类型 | 说明 |
|---|---|---|
| 整数 | `smallint` / `int2` | 2 字节，-32768 ~ 32767 |
| 整数 | `integer` / `int` / `int4` | 4 字节，常用 |
| 整数 | `bigint` / `int8` | 8 字节，主键常用 |
| 自增 | `serial` / `bigserial` | 伪类型，自动建序列 |
| 自增 | `GENERATED ... AS IDENTITY` | 标准写法，推荐 |
| 精确小数 | `numeric(p,s)` / `decimal` | 金额、精确计算 |
| 浮点 | `real` / `float4` | 4 字节，近似值 |
| 浮点 | `double precision` / `float8` | 8 字节，近似值 |
| 字符 | `char(n)` | 定长，不足补空格 |
| 字符 | `varchar(n)` | 变长，有上限 |
| 字符 | `text` | 变长，无上限，推荐 |
| 布尔 | `boolean` / `bool` | `true` / `false` / `NULL` |
| 日期 | `date` | 日期 |
| 时间 | `time` | 一天中的时间 |
| 时间 | `timetz` | 带时区的时间 |
| 时间戳 | `timestamp` | 不带时区 |
| 时间戳 | `timestamptz` | 带时区，推荐 |
| 间隔 | `interval` | 时间间隔 |
| UUID | `uuid` | 通用唯一标识 |
| JSON | `json` | 保留原始文本 |
| JSON | `jsonb` | 二进制，可索引，推荐 |
| 数组 | `type[]` | 任意类型数组 |
| 二进制 | `bytea` | 字节数组 |
| 网络 | `inet` / `cidr` / `macaddr` | IP / 网段 / MAC |
| 范围 | `int4range` / `tsrange` 等 | 区间类型 |
| 枚举 | 自定义 `enum` | 固定取值 |
| 向量 | `vector(n)` | pgvector 浮点向量 |
| 半精度向量 | `halfvec(n)` | 2 字节/维 |
| 二值向量 | `bit(n)` | 位向量 |
| 稀疏向量 | `sparsevec(n)` | 只存非零元素 |

### 3.2 建表时的类型示例

```sql
CREATE TABLE type_demo (
    id           bigint GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    name         varchar(64) NOT NULL,
    description  text,
    price        numeric(12, 2) NOT NULL DEFAULT 0,
    quantity     integer NOT NULL DEFAULT 0,
    enabled      boolean NOT NULL DEFAULT true,
    payload      jsonb NOT NULL DEFAULT '{}'::jsonb,
    tags         text[] NOT NULL DEFAULT '{}',
    user_id      uuid NOT NULL DEFAULT gen_random_uuid(),
    created_at   timestamptz NOT NULL DEFAULT now(),
    updated_at   timestamptz NOT NULL DEFAULT now(),
    expire_at    date
);
```

`gen_random_uuid()` 在 PostgreSQL 13+ 内置，不需要 `uuid-ossp`。

### 3.3 自增主键的三种写法

```sql
-- 1. serial（老写法，自动创建序列）
id bigserial PRIMARY KEY

-- 2. identity（推荐）
id bigint GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY

-- 3. 强制由数据库生成，插入时必须用 OVERRIDING SYSTEM VALUE 才能指定
id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY
```

### 3.4 枚举类型

```sql
CREATE TYPE order_status AS ENUM ('pending', 'paid', 'shipped', 'closed');

CREATE TABLE orders (
    id      bigserial PRIMARY KEY,
    status  order_status NOT NULL DEFAULT 'pending'
);

-- 增加枚举值（不能删除，不能改顺序）
ALTER TYPE order_status ADD VALUE IF NOT EXISTS 'refunded';

-- 查看枚举值
SELECT enumlabel
FROM pg_enum
WHERE enumtypid = 'order_status'::regtype
ORDER BY enumsortorder;
```

### 3.5 复合类型与域

```sql
-- 复合类型
CREATE TYPE address AS (
    province text,
    city     text,
    street   text
);

-- 域：带约束的类型
CREATE DOMAIN positive_int AS integer
    CHECK (VALUE > 0);

CREATE TABLE customer (
    id      bigserial PRIMARY KEY,
    addr    address,
    age     positive_int
);
```

### 3.6 类型转换

```sql
SELECT '123'::integer;
SELECT CAST('2026-10-18' AS date);
SELECT '{"a":1}'::jsonb;
SELECT ARRAY[1,2,3]::text[];
SELECT '[1,2,3]'::vector;
SELECT 1::text || 'abc';
```

---

## 四、数据库操作

### 4.1 查看数据库

```sql
\l
\l+

SELECT datname,
       pg_size_pretty(pg_database_size(datname)) AS size,
       datcollate,
       datctype
FROM pg_database
ORDER BY pg_database_size(datname) DESC;
```

当前数据库：

```sql
SELECT current_database();
SELECT current_user, session_user;
```

### 4.2 创建数据库

```sql
CREATE DATABASE mydb;

CREATE DATABASE mydb
    WITH OWNER = postgres
         ENCODING = 'UTF8'
         TEMPLATE = template0
         CONNECTION LIMIT = 100;
```

基于现有库复制（要求没有其他连接）：

```sql
CREATE DATABASE mydb_copy TEMPLATE mydb;
```

### 4.3 修改数据库

```sql
ALTER DATABASE mydb RENAME TO mydb2;
ALTER DATABASE mydb OWNER TO appuser;
ALTER DATABASE mydb CONNECTION LIMIT 50;
ALTER DATABASE mydb ALLOW_CONNECTIONS false;
ALTER DATABASE mydb ALLOW_CONNECTIONS true;
ALTER DATABASE mydb SET timezone = 'Asia/Shanghai';
ALTER DATABASE mydb SET search_path = app, public;
ALTER DATABASE mydb RESET search_path;
```

### 4.4 删除数据库

```sql
DROP DATABASE mydb;
DROP DATABASE IF EXISTS mydb;
DROP DATABASE mydb WITH (FORCE);   -- PostgreSQL 13+
```

删除前断开连接：

```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'mydb'
  AND pid <> pg_backend_pid();

DROP DATABASE mydb;
```

注意：

- 不能在连接到目标库的会话里删除它，先 `\c postgres`。
- `DROP DATABASE` 不能在事务块里执行。
- `WITH (FORCE)` 会强制断开其他连接，生产慎用。

### 4.5 数据库大小与连接数

```sql
SELECT pg_size_pretty(pg_database_size('mydb'));

SELECT datname,
       numbackends,
       xact_commit,
       xact_rollback,
       blks_read,
       blks_hit,
       deadlocks,
       temp_files,
       pg_size_pretty(temp_bytes) AS temp_size
FROM pg_stat_database
ORDER BY numbackends DESC;
```

### 4.6 数据库注释

```sql
COMMENT ON DATABASE mydb IS '业务主库';
```

### 4.7 数据库级默认配置

```sql
ALTER DATABASE vectordb SET statement_timeout = '30s';
ALTER DATABASE vectordb SET idle_in_transaction_session_timeout = '60s';
ALTER DATABASE vectordb SET hnsw.ef_search = 100;

-- 只对某个角色在某个库生效
ALTER ROLE appuser IN DATABASE vectordb SET search_path = app, public;
```

---

## 五、Schema 操作

### 5.1 查看 schema

```sql
\dn
\dn+

SELECT nspname AS schema_name,
       pg_get_userbyid(nspowner) AS owner
FROM pg_namespace
WHERE nspname NOT LIKE 'pg_%'
  AND nspname <> 'information_schema'
ORDER BY nspname;
```

当前 schema 和搜索路径：

```sql
SELECT current_schema();
SHOW search_path;
```

### 5.2 创建 schema

```sql
CREATE SCHEMA app;
CREATE SCHEMA IF NOT EXISTS app;
CREATE SCHEMA app AUTHORIZATION appuser;
CREATE SCHEMA IF NOT EXISTS app AUTHORIZATION appuser;
```

### 5.3 修改 schema

```sql
ALTER SCHEMA app RENAME TO business;
ALTER SCHEMA business OWNER TO appuser;
```

改名后检查当前会话：

```sql
SHOW search_path;
SET search_path TO business, public;
ALTER ROLE appuser SET search_path TO business, public;
```

### 5.4 删除 schema

```sql
DROP SCHEMA app;
DROP SCHEMA IF EXISTS app;
DROP SCHEMA app CASCADE;    -- 连同里面的对象一起删除
DROP SCHEMA app RESTRICT;   -- 有对象就报错，默认行为
```

### 5.5 schema 与对象

```sql
-- 在指定 schema 建表
CREATE TABLE app.app_user (...);

-- 把表移动到另一个 schema
ALTER TABLE app.app_user SET SCHEMA business;

-- 查看某 schema 下的表
\dt app.*

SELECT table_schema, table_name
FROM information_schema.tables
WHERE table_schema = 'app'
ORDER BY table_name;
```

### 5.6 search_path 规则

```sql
SHOW search_path;
SET search_path TO app, public;
SET LOCAL search_path TO app, public;   -- 只对当前事务生效
RESET search_path;
```

不写 schema 的表名按 `search_path` 从左到右查找。生产环境建议显式写 schema，或者把默认 schema 固定在连接配置里。

---

## 六、角色、用户与权限

### 6.1 角色与用户的区别

- `CREATE ROLE` 创建角色，默认不能登录。
- `CREATE USER` 等价于 `CREATE ROLE ... LOGIN`。
- 角色既可以当用户，也可以当权限组。

### 6.2 创建角色

```sql
-- 可登录用户
CREATE ROLE appuser LOGIN PASSWORD 'StrongPass123';

-- 带更多属性
CREATE ROLE appuser
    LOGIN
    PASSWORD 'StrongPass123'
    NOSUPERUSER
    NOCREATEDB
    NOCREATEROLE
    NOINHERIT
    CONNECTION LIMIT 20
    VALID UNTIL '2027-12-31';

-- 只读角色
CREATE ROLE readonly NOLOGIN;

-- 可建库、可建角色
CREATE ROLE devuser LOGIN PASSWORD 'devpass' CREATEDB CREATEROLE;

-- 复制用户
CREATE ROLE replicator REPLICATION LOGIN PASSWORD 'replpass';
```

查看角色：

```sql
\du
\du+

SELECT rolname,
       rolsuper,
       rolcanlogin,
       rolcreatedb,
       rolcreaterole,
       rolreplication,
       rolconnlimit,
       rolvaliduntil
FROM pg_roles
ORDER BY rolname;
```

### 6.3 修改角色

```sql
ALTER ROLE appuser WITH PASSWORD 'NewPass456';
ALTER ROLE appuser WITH CONNECTION LIMIT 50;
ALTER ROLE appuser WITH VALID UNTIL 'infinity';
ALTER ROLE appuser NOSUPERUSER NOCREATEDB NOCREATEROLE;
ALTER ROLE appuser RENAME TO app_user;

-- 角色级默认配置
ALTER ROLE appuser SET search_path TO app, public;
ALTER ROLE appuser SET statement_timeout = '60s';
ALTER ROLE appuser RESET statement_timeout;

-- 只对某个数据库生效
ALTER ROLE appuser IN DATABASE vectordb SET search_path TO app, public;
```

### 6.4 删除角色

```sql
DROP ROLE appuser;
DROP ROLE IF EXISTS appuser;
```

角色拥有对象时不能直接删，先转移或清理：

```sql
REASSIGN OWNED BY appuser TO postgres;
DROP OWNED BY appuser;
DROP ROLE appuser;
```

`REASSIGN OWNED` 和 `DROP OWNED` 需要在**每个数据库**里分别执行。

### 6.5 角色成员关系

```sql
-- 把角色授予用户
GRANT readonly TO alice;
GRANT readonly TO bob WITH ADMIN OPTION;

-- 收回
REVOKE readonly FROM alice;

-- 查看成员关系
\du
SELECT r.rolname AS role_name,
       m.rolname AS member_name,
       am.admin_option
FROM pg_auth_members am
JOIN pg_roles r ON r.oid = am.roleid
JOIN pg_roles m ON m.oid = am.member
ORDER BY r.rolname, m.rolname;
```

临时切换角色：

```sql
SET ROLE appuser;
SELECT current_user;
RESET ROLE;
```

### 6.6 数据库权限

```sql
GRANT CONNECT ON DATABASE vectordb TO appuser;
GRANT CREATE, TEMPORARY ON DATABASE vectordb TO appuser;
GRANT ALL PRIVILEGES ON DATABASE vectordb TO appuser;

REVOKE CREATE ON DATABASE vectordb FROM appuser;
REVOKE ALL PRIVILEGES ON DATABASE vectordb FROM appuser;
```

### 6.7 Schema 权限

```sql
GRANT USAGE ON SCHEMA app TO appuser;
GRANT CREATE ON SCHEMA app TO appuser;
GRANT USAGE, CREATE ON SCHEMA app TO appuser;

REVOKE CREATE ON SCHEMA app FROM appuser;
```

PostgreSQL 15 起，普通用户默认没有 `public` schema 的 `CREATE` 权限。需要可写 schema 时建议单独建：

```sql
CREATE SCHEMA app AUTHORIZATION appuser;
GRANT USAGE, CREATE ON SCHEMA app TO appuser;
```

### 6.8 表与序列权限

```sql
GRANT SELECT ON ALL TABLES IN SCHEMA app TO readonly;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app TO appuser;
GRANT TRUNCATE, REFERENCES, TRIGGER ON ALL TABLES IN SCHEMA app TO appuser;

GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA app TO appuser;

-- 单表
GRANT SELECT ON app.app_user TO readonly;
GRANT SELECT (id, username) ON app.app_user TO readonly;   -- 列级权限

-- 收回
REVOKE DELETE ON ALL TABLES IN SCHEMA app FROM appuser;
```

### 6.9 默认权限（以后新建的对象自动授权）

```sql
ALTER DEFAULT PRIVILEGES IN SCHEMA app
GRANT SELECT ON TABLES TO readonly;

ALTER DEFAULT PRIVILEGES IN SCHEMA app
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO appuser;

ALTER DEFAULT PRIVILEGES IN SCHEMA app
GRANT USAGE, SELECT ON SEQUENCES TO appuser;

-- 只对某个创建者生效
ALTER DEFAULT PRIVILEGES FOR ROLE appuser IN SCHEMA app
GRANT SELECT ON TABLES TO readonly;
```

### 6.10 函数与类型权限

```sql
GRANT EXECUTE ON FUNCTION app.calc_total(bigint) TO appuser;
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA app TO appuser;
GRANT USAGE ON TYPE app.order_status TO appuser;
```

### 6.11 修改对象所有者

```sql
ALTER TABLE app.app_user OWNER TO appuser;
ALTER SEQUENCE app.app_user_id_seq OWNER TO appuser;
ALTER SCHEMA app OWNER TO appuser;
ALTER DATABASE vectordb OWNER TO appuser;
ALTER FUNCTION app.calc_total(bigint) OWNER TO appuser;
```

### 6.12 查看权限

```sql
\dp
\dp app.app_user
\ddp

SELECT grantee, privilege_type
FROM information_schema.role_table_grants
WHERE table_schema = 'app'
  AND table_name = 'app_user'
ORDER BY grantee, privilege_type;

SELECT has_table_privilege('appuser', 'app.app_user', 'SELECT');
SELECT has_schema_privilege('appuser', 'app', 'USAGE');
SELECT has_database_privilege('appuser', 'vectordb', 'CONNECT');
```

### 6.13 行级安全（RLS）

```sql
ALTER TABLE app.orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE app.orders FORCE ROW LEVEL SECURITY;

CREATE POLICY orders_tenant_isolation
ON app.orders
FOR ALL
TO appuser
USING (tenant_id = nullif(current_setting('app.tenant_id', true), '')::bigint)
WITH CHECK (tenant_id = nullif(current_setting('app.tenant_id', true), '')::bigint);

-- 会话里设置租户
SET app.tenant_id = '1001';

-- 查看策略
\d app.orders
SELECT * FROM pg_policies WHERE tablename = 'orders';

-- 关闭
ALTER TABLE app.orders DISABLE ROW LEVEL SECURITY;
DROP POLICY orders_tenant_isolation ON app.orders;
```

表所有者默认绕过 RLS，`FORCE ROW LEVEL SECURITY` 可以让所有者也被策略约束。

### 6.14 预定义角色

| 角色 | 作用 |
|---|---|
| `pg_read_all_data` | 读取所有表 |
| `pg_write_all_data` | 写入所有表 |
| `pg_read_all_settings` | 读取所有配置 |
| `pg_monitor` | 监控权限集合 |
| `pg_stat_scan_tables` | 统计信息相关 |
| `pg_signal_backend` | 取消/终止其他会话 |
| `pg_checkpoint` | 执行 CHECKPOINT |
| `pg_create_subscription` | 创建订阅（PG16+） |

```sql
GRANT pg_read_all_data TO readonly;
GRANT pg_monitor TO monitor_user;
```

---

## 七、表操作（DDL）

### 7.1 创建表

```sql
CREATE TABLE app.app_user (
    id          bigint GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    username    varchar(64)  NOT NULL UNIQUE,
    email       varchar(128),
    status      smallint     NOT NULL DEFAULT 1,
    profile     jsonb        NOT NULL DEFAULT '{}'::jsonb,
    tags        text[]       NOT NULL DEFAULT '{}',
    created_at  timestamptz  NOT NULL DEFAULT now(),
    updated_at  timestamptz  NOT NULL DEFAULT now()
);
```

带命名约束：

```sql
CREATE TABLE app.orders (
    id          bigint GENERATED BY DEFAULT AS IDENTITY,
    tenant_id   bigint NOT NULL,
    order_no    varchar(64) NOT NULL,
    amount      numeric(12,2) NOT NULL DEFAULT 0,
    status      varchar(20) NOT NULL DEFAULT 'pending',
    created_at  timestamptz NOT NULL DEFAULT now(),
    CONSTRAINT pk_orders PRIMARY KEY (id),
    CONSTRAINT uk_orders_no UNIQUE (order_no),
    CONSTRAINT ck_orders_amount CHECK (amount >= 0)
);
```

### 7.2 临时表与非日志表

```sql
-- 会话级临时表
CREATE TEMP TABLE tmp_users (
    id bigint,
    name text
) ON COMMIT DROP;

-- 非日志表：不写 WAL，崩溃后可能丢数据，适合可重建的中间数据
CREATE UNLOGGED TABLE import_staging (
    id bigint,
    payload jsonb
);
```

### 7.3 复制表结构

```sql
-- 只复制结构
CREATE TABLE app.app_user_copy (LIKE app.app_user);

-- 复制结构、默认值、约束、索引等
CREATE TABLE app.app_user_copy (LIKE app.app_user INCLUDING ALL);

-- 复制结构和数据
CREATE TABLE app.app_user_backup AS
SELECT * FROM app.app_user;
```

`CREATE TABLE AS` 不会复制主键、索引、默认值和约束。

### 7.4 增加列

```sql
ALTER TABLE app.app_user ADD COLUMN nickname varchar(64);
ALTER TABLE app.app_user ADD COLUMN score integer NOT NULL DEFAULT 0;
ALTER TABLE app.app_user ADD COLUMN IF NOT EXISTS remark text;
```

PostgreSQL 11+ 添加带默认值的列通常不需要重写整张表。

### 7.5 删除列

```sql
ALTER TABLE app.app_user DROP COLUMN nickname;
ALTER TABLE app.app_user DROP COLUMN IF EXISTS nickname;
ALTER TABLE app.app_user DROP COLUMN nickname CASCADE;
```

### 7.6 修改列

```sql
-- 改类型
ALTER TABLE app.app_user ALTER COLUMN email TYPE varchar(255);

-- 改类型并指定转换
ALTER TABLE app.app_user
    ALTER COLUMN status TYPE integer
    USING status::integer;

-- 改默认值
ALTER TABLE app.app_user ALTER COLUMN status SET DEFAULT 0;
ALTER TABLE app.app_user ALTER COLUMN status DROP DEFAULT;

-- 非空
ALTER TABLE app.app_user ALTER COLUMN email SET NOT NULL;
ALTER TABLE app.app_user ALTER COLUMN email DROP NOT NULL;

-- 列名
ALTER TABLE app.app_user RENAME COLUMN nickname TO display_name;

-- 列注释
COMMENT ON COLUMN app.app_user.email IS '邮箱';
```

### 7.7 自增列与生成列

```sql
-- 给已有列加 identity
ALTER TABLE app.app_user
    ALTER COLUMN id ADD GENERATED BY DEFAULT AS IDENTITY;

-- 修改 identity 起始值
ALTER TABLE app.app_user ALTER COLUMN id RESTART WITH 1000;

-- 删除 identity
ALTER TABLE app.app_user ALTER COLUMN id DROP IDENTITY IF EXISTS;

-- 生成列（表达式自动计算，不能直接写入）
ALTER TABLE app.app_user
    ADD COLUMN name_lower text
    GENERATED ALWAYS AS (lower(username)) STORED;
```

### 7.8 修改表名和 schema

```sql
ALTER TABLE app.app_user RENAME TO sys_user;
ALTER TABLE app.sys_user SET SCHEMA business;
ALTER TABLE business.sys_user RENAME TO app_user;
```

### 7.9 修改所有者、表空间、存储参数

```sql
ALTER TABLE app.app_user OWNER TO appuser;
ALTER TABLE app.app_user SET TABLESPACE pg_default;

-- fillfactor：预留更新空间，减少页分裂
ALTER TABLE app.app_user SET (fillfactor = 90);

-- autovacuum 参数
ALTER TABLE app.app_user SET (
    autovacuum_vacuum_scale_factor = 0.05,
    autovacuum_analyze_scale_factor = 0.02
);

-- 列统计目标，影响查询计划
ALTER TABLE app.app_user ALTER COLUMN status SET STATISTICS 500;

-- 列压缩（PG14+，仅对支持压缩的存储有效）
ALTER TABLE app.app_user ALTER COLUMN profile SET COMPRESSION lz4;
```

### 7.10 清空与删除表

```sql
TRUNCATE TABLE app.app_user;
TRUNCATE TABLE app.app_user RESTART IDENTITY;
TRUNCATE TABLE app.app_user RESTART IDENTITY CASCADE;

DROP TABLE app.app_user;
DROP TABLE IF EXISTS app.app_user;
DROP TABLE app.app_user CASCADE;
```

区别：

- `DELETE` 逐行删除，可回滚，会触发触发器，适合有条件删除。
- `TRUNCATE` 快速清空，默认加 `ACCESS EXCLUSIVE` 锁，不可条件删除。
- `DROP` 删除表结构和数据。

### 7.11 表注释

```sql
COMMENT ON TABLE app.app_user IS '应用用户表';
COMMENT ON COLUMN app.app_user.username IS '登录名';

SELECT obj_description('app.app_user'::regclass, 'pg_class');
```

### 7.12 查看表信息

```sql
\d app.app_user
\d+ app.app_user

SELECT column_name,
       data_type,
       character_maximum_length,
       is_nullable,
       column_default
FROM information_schema.columns
WHERE table_schema = 'app'
  AND table_name = 'app_user'
ORDER BY ordinal_position;

SELECT table_schema, table_name, table_type
FROM information_schema.tables
WHERE table_schema = 'app'
ORDER BY table_name;
```

### 7.13 分区表

```sql
-- 按范围分区
CREATE TABLE app.orders (
    id          bigint GENERATED BY DEFAULT AS IDENTITY,
    tenant_id   bigint NOT NULL,
    created_at  timestamptz NOT NULL,
    amount      numeric(12,2) NOT NULL,
    PRIMARY KEY (id, created_at)
) PARTITION BY RANGE (created_at);

CREATE TABLE app.orders_2026_01
    PARTITION OF app.orders
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE app.orders_2026_02
    PARTITION OF app.orders
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

-- 默认分区
CREATE TABLE app.orders_default
    PARTITION OF app.orders DEFAULT;

-- 查看分区
\d+ app.orders
SELECT relname, pg_get_expr(relpartbound, oid)
FROM pg_class
WHERE relispartition = true;

-- 脱离分区
ALTER TABLE app.orders DETACH PARTITION app.orders_2026_01;
ALTER TABLE app.orders DETACH PARTITION app.orders_2026_01 CONCURRENTLY;

-- 重新挂载
ALTER TABLE app.orders ATTACH PARTITION app.orders_2026_01
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```

分区键必须包含在主键或唯一约束里，否则无法创建对应约束。

### 7.14 继承（了解即可）

```sql
CREATE TABLE app.base_event (
    id bigserial PRIMARY KEY,
    created_at timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE app.login_event (
    user_id bigint
) INHERITS (app.base_event);
```

新项目优先用声明式分区，继承更多用于历史系统。

### 7.15 表空间

```sql
-- 需要超级用户；目录必须存在、为空，且不能位于 PGDATA 内
CREATE TABLESPACE fastspace
    LOCATION '/var/lib/postgresql/tablespaces/fast';

ALTER TABLE app.orders SET TABLESPACE fastspace;
ALTER INDEX app.idx_orders_status SET TABLESPACE fastspace;

-- 查看表空间
\db
SELECT spcname, pg_tablespace_location(oid)
FROM pg_tablespace;

-- 删除前必须先把对象移走
ALTER TABLE app.orders SET TABLESPACE pg_default;
DROP TABLESPACE fastspace;
```

Docker 里要先在容器内建目录并交给 `postgres` 用户：

```bash
mkdir -p /var/lib/postgresql/tablespaces/fast
chown postgres:postgres /var/lib/postgresql/tablespaces/fast
chmod 700 /var/lib/postgresql/tablespaces/fast
```

---

## 八、约束

### 8.1 约束类型

| 约束 | 作用 |
|---|---|
| `PRIMARY KEY` | 主键，非空且唯一 |
| `UNIQUE` | 唯一 |
| `FOREIGN KEY` | 外键 |
| `CHECK` | 条件检查 |
| `NOT NULL` | 非空 |
| `EXCLUSION` | 排他约束，范围/几何场景常用 |

### 8.2 主键

```sql
-- 建表时
CREATE TABLE app.app_user (
    id bigint GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    username varchar(64) NOT NULL
);

-- 命名主键
CREATE TABLE app.orders (
    id bigint NOT NULL,
    CONSTRAINT pk_orders PRIMARY KEY (id)
);

-- 给已有表加主键
ALTER TABLE app.orders ADD CONSTRAINT pk_orders PRIMARY KEY (id);

-- 删除主键
ALTER TABLE app.orders DROP CONSTRAINT pk_orders;
```

### 8.3 唯一约束

```sql
ALTER TABLE app.app_user
    ADD CONSTRAINT uk_app_user_username UNIQUE (username);

-- 多列唯一
ALTER TABLE app.orders
    ADD CONSTRAINT uk_orders_tenant_no UNIQUE (tenant_id, order_no);

-- PostgreSQL 15+：NULL 也视为相同值
ALTER TABLE app.app_user
    ADD CONSTRAINT uk_app_user_email UNIQUE NULLS NOT DISTINCT (email);

-- 删除
ALTER TABLE app.app_user DROP CONSTRAINT uk_app_user_username;
```

### 8.4 外键

```sql
ALTER TABLE app.orders
    ADD CONSTRAINT fk_orders_user
    FOREIGN KEY (user_id) REFERENCES app.app_user(id);

-- 级联删除、级联更新
ALTER TABLE app.orders
    ADD CONSTRAINT fk_orders_user
    FOREIGN KEY (user_id) REFERENCES app.app_user(id)
    ON DELETE CASCADE
    ON UPDATE CASCADE;

-- 删除父记录时把外键置空
ON DELETE SET NULL

-- 删除父记录时使用默认值
ON DELETE SET DEFAULT

-- 阻止删除/更新（默认）
ON DELETE RESTRICT
ON UPDATE RESTRICT

-- 延迟到事务结束时检查
ALTER TABLE app.orders
    ADD CONSTRAINT fk_orders_user
    FOREIGN KEY (user_id) REFERENCES app.app_user(id)
    DEFERRABLE INITIALLY DEFERRED;
```

外键列建议建索引，否则删除父表记录时可能全表扫描子表。

### 8.5 检查约束

```sql
ALTER TABLE app.orders
    ADD CONSTRAINT ck_orders_amount CHECK (amount >= 0);

ALTER TABLE app.orders
    ADD CONSTRAINT ck_orders_status
    CHECK (status IN ('pending', 'paid', 'shipped', 'closed'));

-- 先加不校验，后续再验证，减少锁表时间
ALTER TABLE app.orders
    ADD CONSTRAINT ck_orders_amount CHECK (amount >= 0) NOT VALID;

ALTER TABLE app.orders VALIDATE CONSTRAINT ck_orders_amount;

-- 删除
ALTER TABLE app.orders DROP CONSTRAINT ck_orders_amount;
```

`CHECK` 返回 `NULL` 时视为通过，所以 `CHECK (amount >= 0)` 不会阻止 `amount IS NULL`。需要非空时另加 `NOT NULL`。

### 8.6 排他约束

```sql
-- 防止同一会议室时间重叠
CREATE EXTENSION IF NOT EXISTS btree_gist;

ALTER TABLE app.booking
    ADD CONSTRAINT no_overlap
    EXCLUDE USING gist (
        room_id WITH =,
        tstzrange(start_at, end_at, '[)') WITH &&
    );
```

### 8.7 查看约束

```sql
\d app.orders

SELECT conname,
       contype,
       pg_get_constraintdef(oid) AS definition
FROM pg_constraint
WHERE conrelid = 'app.orders'::regclass
ORDER BY conname;
```

`contype`：`p` 主键、`u` 唯一、`f` 外键、`c` 检查、`x` 排他。

### 8.8 约束的启用与延迟

```sql
-- 事务内延迟检查（约束必须声明为 DEFERRABLE）
SET CONSTRAINTS ALL DEFERRED;
SET CONSTRAINTS fk_orders_user IMMEDIATE;
```

---

## 九、索引

### 9.1 索引类型

| 类型 | 适用场景 |
|---|---|
| `btree` | 等值、范围、排序，默认类型 |
| `hash` | 仅等值查询 |
| `gin` | 数组、JSONB、全文检索、多值列 |
| `gist` | 几何、范围、全文检索、排他约束 |
| `brin` | 超大表、按物理顺序存储的时间/序列列 |
| `spgist` | 文本搜索、几何等特定场景 |
| `hnsw` | pgvector 向量近似检索，首选 |
| `ivfflat` | pgvector 向量近似检索，另一种选择 |

### 9.2 创建普通索引

```sql
CREATE INDEX idx_app_user_created_at
    ON app.app_user (created_at DESC);

CREATE INDEX IF NOT EXISTS idx_app_user_status
    ON app.app_user (status);

-- 多列索引
CREATE INDEX idx_orders_tenant_status_created
    ON app.orders (tenant_id, status, created_at DESC);
```

多列索引遵循最左前缀原则：`(tenant_id, status, created_at)` 可以支持按 `tenant_id`、`tenant_id + status` 查询，但单独按 `status` 查询通常用不上。

### 9.3 唯一索引

```sql
CREATE UNIQUE INDEX uk_app_user_email_lower
    ON app.app_user (lower(email))
    WHERE email IS NOT NULL;

CREATE UNIQUE INDEX uk_orders_tenant_no
    ON app.orders (tenant_id, order_no);
```

唯一索引和唯一约束的区别：唯一约束会出现在 `pg_constraint` 里，唯一索引只出现在 `pg_index` 里；唯一索引可以带 `WHERE` 条件。

### 9.4 部分索引

```sql
-- 只索引未删除的数据
CREATE INDEX idx_orders_active_created
    ON app.orders (created_at DESC)
    WHERE deleted_at IS NULL;

-- 只索引待处理订单
CREATE INDEX idx_orders_pending
    ON app.orders (created_at)
    WHERE status = 'pending';
```

查询条件必须能匹配 `WHERE` 谓词，规划器才会使用部分索引。

### 9.5 表达式索引

```sql
CREATE INDEX idx_app_user_lower_username
    ON app.app_user (lower(username));

-- 查询要写成同样的表达式
SELECT * FROM app.app_user WHERE lower(username) = 'alice';
```

### 9.6 覆盖索引

```sql
CREATE INDEX idx_orders_tenant_include
    ON app.orders (tenant_id)
    INCLUDE (status, amount);
```

`INCLUDE` 列只用于 index-only scan，不参与排序和唯一性判断。

### 9.7 并发建索引

```sql
CREATE INDEX CONCURRENTLY idx_orders_status
    ON app.orders (status);
```

注意：

- 不能放在事务块里执行。
- 失败后会留下 `INVALID` 索引，需要删除重建。
- 比普通 `CREATE INDEX` 慢，但不会长时间阻塞写入。

查看无效索引：

```sql
SELECT indexrelid::regclass AS index_name,
       indrelid::regclass AS table_name,
       indisvalid
FROM pg_index
WHERE NOT indisvalid;
```

### 9.8 删除与重建索引

```sql
DROP INDEX app.idx_orders_status;
DROP INDEX IF EXISTS app.idx_orders_status;

REINDEX INDEX app.idx_orders_status;
REINDEX TABLE app.orders;
REINDEX SCHEMA app;
REINDEX DATABASE vectordb;

-- 并发重建（PG12+）
REINDEX INDEX CONCURRENTLY app.idx_orders_status;
REINDEX TABLE CONCURRENTLY app.orders;
```

### 9.9 CLUSTER

```sql
-- 按索引顺序重写表，物理顺序与索引一致
CLUSTER app.orders USING idx_orders_created_at;

-- 后续对该表执行 CLUSTER 时沿用上次的索引
CLUSTER app.orders;

-- 查看表的 cluster 标记
SELECT relname, relcluster
FROM pg_class
WHERE relname = 'orders';
```

`CLUSTER` 会锁表，期间阻塞读写。大表建议在维护窗口执行。

### 9.10 索引大小与使用情况

```sql
SELECT indexrelname AS index_name,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size,
       idx_scan,
       idx_tup_read,
       idx_tup_fetch
FROM pg_stat_user_indexes
WHERE relname = 'orders'
ORDER BY pg_relation_size(indexrelid) DESC;

SELECT relname AS table_name,
       pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
       pg_size_pretty(pg_relation_size(relid)) AS table_size,
       pg_size_pretty(pg_indexes_size(relid)) AS index_size
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC;
```

长期 `idx_scan = 0` 的索引可以考虑删除，但要先确认不是只在月初/月末使用的业务索引。

### 9.11 查看索引定义

```sql
\di
\di+ app.idx_orders_status

SELECT indexname, indexdef
FROM pg_indexes
WHERE schemaname = 'app'
  AND tablename = 'orders';

SELECT pg_get_indexdef('app.idx_orders_status'::regclass);
```

### 9.12 索引失效的常见原因

- 查询条件对列做了函数运算，例如 `WHERE upper(name) = 'A'`，但索引建在 `name` 上。
- 类型不匹配，例如 `varchar` 列和 `integer` 比较。
- 使用 `LIKE '%abc'` 前置通配符，普通 B-Tree 用不上。
- 统计信息过期，执行 `ANALYZE`。
- 表太小，规划器认为全表扫描更快。
- 部分索引的 `WHERE` 条件与查询不匹配。
- 多列索引没有遵循最左前缀。

### 9.13 扩展统计信息

多列之间存在相关性时，普通统计信息可能估算不准：

```sql
CREATE STATISTICS app.stats_orders_dependency (dependencies, ndistinct)
ON tenant_id, status
FROM app.orders;

ANALYZE app.orders;

SELECT stxname, stxkeys, stxkind
FROM pg_statistic_ext
WHERE stxname = 'stats_orders_dependency';

DROP STATISTICS app.stats_orders_dependency;
```

`dependencies` 适合“城市依赖省份”这类列，`ndistinct` 适合多列组合基数估算，`mcv` 适合组合值分布。

---

## 十、数据操作（DML）

### 10.1 插入单行

```sql
INSERT INTO app.app_user (username, email)
VALUES ('alice', 'alice@example.com');

INSERT INTO app.app_user (username, email)
VALUES ('bob', 'bob@example.com')
RETURNING id, username, created_at;
```

只插入默认值：

```sql
INSERT INTO app.app_user DEFAULT VALUES;
```

### 10.2 批量插入

```sql
INSERT INTO app.app_user (username, email) VALUES
('alice', 'alice@example.com'),
('bob',   'bob@example.com'),
('carol', 'carol@example.com');
```

从查询结果插入：

```sql
INSERT INTO app.app_user (username, email)
SELECT 'user_' || g, 'user_' || g || '@example.com'
FROM generate_series(1, 1000) AS g;
```

### 10.3 指定 identity 值

```sql
-- GENERATED ALWAYS AS IDENTITY 的列
INSERT INTO app.app_user (id, username)
OVERRIDING SYSTEM VALUE
VALUES (1000, 'alice');
```

### 10.4 Upsert（ON CONFLICT）

```sql
INSERT INTO app.app_user (username, email)
VALUES ('alice', 'new@example.com')
ON CONFLICT (username)
DO UPDATE SET email = EXCLUDED.email,
              updated_at = now()
RETURNING *;

-- 冲突时什么也不做
INSERT INTO app.app_user (username, email)
VALUES ('alice', 'alice@example.com')
ON CONFLICT (username) DO NOTHING;

-- 按约束名
INSERT INTO app.app_user (username, email)
VALUES ('alice', 'alice@example.com')
ON CONFLICT ON CONSTRAINT uk_app_user_username
DO UPDATE SET email = EXCLUDED.email;
```

`EXCLUDED` 表示本次试图插入的那一行。

### 10.5 更新

```sql
UPDATE app.app_user
SET email = 'new@example.com',
    updated_at = now()
WHERE username = 'alice'
RETURNING id, username, email;

-- 用另一张表更新
UPDATE app.orders o
SET status = s.status
FROM app.order_status_stage s
WHERE o.order_no = s.order_no;

-- 只更新匹配的前 N 行（配合子查询）
UPDATE app.orders
SET status = 'processing'
WHERE id IN (
    SELECT id FROM app.orders
    WHERE status = 'pending'
    ORDER BY created_at
    LIMIT 100
)
RETURNING id;
```

### 10.6 删除

```sql
DELETE FROM app.app_user WHERE username = 'alice';
DELETE FROM app.app_user WHERE id = 1 RETURNING *;

-- 用另一张表删除
DELETE FROM app.orders o
USING app.cancelled_orders c
WHERE o.id = c.order_id;

-- 删除全部（可回滚，但大表很慢）
DELETE FROM app.orders;
```

### 10.7 MERGE（PostgreSQL 15+）

```sql
MERGE INTO app.app_user AS t
USING (VALUES ('alice', 'alice@example.com')) AS s(username, email)
ON t.username = s.username
WHEN MATCHED THEN
    UPDATE SET email = s.email, updated_at = now()
WHEN NOT MATCHED THEN
    INSERT (username, email) VALUES (s.username, s.email);
```

### 10.8 COPY 导入导出

服务端 `COPY`（文件在数据库服务器/容器里）：

```sql
COPY app.app_user (username, email)
FROM '/tmp/users.csv'
WITH (FORMAT csv, HEADER true, ENCODING 'UTF8');

COPY app.app_user TO '/tmp/users.csv'
WITH (FORMAT csv, HEADER true);
```

客户端 `\copy`（文件在 psql 所在机器）：

```sql
\copy app.app_user FROM 'C:/data/users.csv' WITH (FORMAT csv, HEADER true)
\copy app.app_user TO 'C:/data/users.csv' WITH (FORMAT csv, HEADER true)
```

容器场景下，`COPY` 读取的是容器内路径，文件要先 `docker cp` 进去；`\copy` 读取的是执行 psql 的那台机器路径。

### 10.9 RETURNING

`INSERT`、`UPDATE`、`DELETE` 都支持 `RETURNING`：

```sql
INSERT INTO app.app_user (username) VALUES ('dave') RETURNING id;
UPDATE app.app_user SET status = 0 WHERE id = 1 RETURNING id, status;
DELETE FROM app.app_user WHERE id = 1 RETURNING *;
```

### 10.10 清空表

```sql
TRUNCATE TABLE app.app_user;
TRUNCATE TABLE app.app_user RESTART IDENTITY;
TRUNCATE TABLE app.app_user RESTART IDENTITY CASCADE;
```

### 10.11 批量更新注意事项

- 大批量 `UPDATE`/`DELETE` 建议分批执行，避免长事务、锁表和 WAL 暴涨。
- 每批处理 1000～10000 行，提交后再继续。
- 大批量导入前可临时删除索引，导入后重建。
- 导入完成后执行 `ANALYZE`，让规划器获得新统计信息。

---

## 十一、查询语法

### 11.1 SELECT 基本结构

```sql
SELECT id, username, email
FROM app.app_user
WHERE status = 1
ORDER BY created_at DESC
LIMIT 20 OFFSET 0;
```

逻辑执行顺序大致是：

```text
FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT/OFFSET
```

### 11.2 去重

```sql
SELECT DISTINCT status FROM app.app_user;

-- 每个 tenant_id 取最新一条
SELECT DISTINCT ON (tenant_id) *
FROM app.orders
ORDER BY tenant_id, created_at DESC;
```

`DISTINCT ON` 的 `ORDER BY` 必须以 `DISTINCT ON` 的列开头。

### 11.3 分页

```sql
SELECT * FROM app.orders ORDER BY id LIMIT 20 OFFSET 40;
SELECT * FROM app.orders ORDER BY id OFFSET 40 ROWS FETCH FIRST 20 ROWS ONLY;
```

深分页推荐 keyset 方式：

```sql
SELECT *
FROM app.orders
WHERE id > 100000
ORDER BY id
LIMIT 20;
```

`OFFSET` 越大越慢，因为数据库要先扫描并丢弃前面的行。

### 11.4 排序与 NULL

```sql
SELECT * FROM app.orders ORDER BY created_at DESC NULLS LAST;
SELECT * FROM app.orders ORDER BY status ASC, id DESC;
```

### 11.5 连接

```sql
-- 内连接
SELECT o.id, o.order_no, u.username
FROM app.orders o
JOIN app.app_user u ON u.id = o.user_id;

-- 左连接
SELECT u.id, u.username, o.id AS order_id
FROM app.app_user u
LEFT JOIN app.orders o ON o.user_id = u.id;

-- 右连接、全连接
RIGHT JOIN app.orders o ON ...
FULL JOIN app.orders o ON ...

-- 交叉连接
SELECT * FROM app.app_user CROSS JOIN app.orders;

-- 自连接
SELECT e.id, e.name, m.name AS manager_name
FROM app.employee e
LEFT JOIN app.employee m ON m.id = e.manager_id;
```

### 11.6 LATERAL

```sql
SELECT u.id, u.username, o.id AS latest_order_id, o.amount
FROM app.app_user u
LEFT JOIN LATERAL (
    SELECT id, amount
    FROM app.orders
    WHERE user_id = u.id
    ORDER BY created_at DESC
    LIMIT 1
) o ON true;
```

`LATERAL` 子查询可以引用左侧表的列，适合“每个用户取最新一条”这类查询。

### 11.7 聚合

```sql
SELECT status, count(*) AS cnt, sum(amount) AS total
FROM app.orders
GROUP BY status
HAVING count(*) > 10
ORDER BY cnt DESC;

SELECT count(*) AS total,
       count(*) FILTER (WHERE status = 'paid') AS paid_count,
       sum(amount) FILTER (WHERE status = 'paid') AS paid_amount
FROM app.orders;

SELECT string_agg(username, ', ' ORDER BY username) FROM app.app_user;
SELECT array_agg(id ORDER BY id) FROM app.app_user;
SELECT jsonb_agg(jsonb_build_object('id', id, 'name', username)) FROM app.app_user;
```

分组集合：

```sql
SELECT tenant_id, status, sum(amount)
FROM app.orders
GROUP BY ROLLUP (tenant_id, status);

GROUP BY GROUPING SETS ((tenant_id), (status), ());
GROUP BY CUBE (tenant_id, status);
```

### 11.8 窗口函数

```sql
SELECT id,
       user_id,
       amount,
       row_number() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn,
       rank() OVER (ORDER BY amount DESC) AS amount_rank,
       sum(amount) OVER (PARTITION BY user_id) AS user_total,
       avg(amount) OVER (
           PARTITION BY user_id
           ORDER BY created_at
           ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
       ) AS moving_avg_7
FROM app.orders;
```

取每组最新一条：

```sql
SELECT *
FROM (
    SELECT o.*,
           row_number() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
    FROM app.orders o
) t
WHERE rn = 1;
```

### 11.9 CTE

```sql
WITH paid AS (
    SELECT * FROM app.orders WHERE status = 'paid'
)
SELECT user_id, sum(amount)
FROM paid
GROUP BY user_id;
```

递归 CTE：

```sql
WITH RECURSIVE org AS (
    SELECT id, name, manager_id, 1 AS level
    FROM app.employee
    WHERE manager_id IS NULL

    UNION ALL

    SELECT e.id, e.name, e.manager_id, o.level + 1
    FROM app.employee e
    JOIN org o ON e.manager_id = o.id
)
SELECT * FROM org ORDER BY level, id;
```

### 11.10 集合运算

```sql
SELECT id FROM app.orders_2026_01
UNION
SELECT id FROM app.orders_2026_02;

UNION ALL   -- 保留重复，性能更好
INTERSECT
EXCEPT
```

### 11.11 CASE 与空值处理

```sql
SELECT id,
       CASE
           WHEN amount >= 1000 THEN 'large'
           WHEN amount >= 100  THEN 'medium'
           ELSE 'small'
       END AS size_level
FROM app.orders;

SELECT COALESCE(email, '未填写') FROM app.app_user;
SELECT NULLIF(status, 0) FROM app.app_user;
SELECT GREATEST(1, 2, 3), LEAST(1, 2, 3);
```

### 11.12 字符串函数

```sql
SELECT 'a' || 'b';
SELECT length('你好');
SELECT lower('ABC'), upper('abc');
SELECT trim('  abc  '), btrim('xxabcxx', 'x');
SELECT substring('abcdef' FROM 2 FOR 3);
SELECT split_part('a,b,c', ',', 2);
SELECT replace('abcabc', 'a', 'x');
SELECT position('bc' IN 'abc');
SELECT left('abcdef', 3), right('abcdef', 3);
SELECT lpad('7', 3, '0'), rpad('7', 3, '0');
SELECT reverse('abc');
SELECT md5('abc');

SELECT 'abc' LIKE 'a%';
SELECT 'abc' ILIKE 'A%';
SELECT 'abc' ~ '^a';
SELECT regexp_replace('a1b2', '\d', '', 'g');
SELECT regexp_match('abc123', '([a-z]+)(\d+)');
```

### 11.13 日期与时间

```sql
SELECT now();
SELECT current_date;
SELECT current_timestamp;
SELECT current_time;
SELECT clock_timestamp();

SELECT date_trunc('day', now());
SELECT date_trunc('month', created_at) AS month, sum(amount)
FROM app.orders
GROUP BY 1
ORDER BY 1;

SELECT extract(year FROM now());
SELECT extract(epoch FROM now());
SELECT age(now(), '2026-01-01'::timestamptz);

SELECT to_char(now(), 'YYYY-MM-DD HH24:MI:SS');
SELECT to_date('2026-10-18', 'YYYY-MM-DD');
SELECT to_timestamp('2026-10-18 12:00:00', 'YYYY-MM-DD HH24:MI:SS');

SELECT now() + interval '1 day';
SELECT now() - interval '7 days';
SELECT created_at AT TIME ZONE 'UTC' FROM app.orders;
```

### 11.14 JSON / JSONB

```sql
-- 取值
SELECT profile -> 'city' FROM app.app_user;
SELECT profile ->> 'city' FROM app.app_user;       -- 返回 text
SELECT profile #> '{address,city}' FROM app.app_user;
SELECT profile #>> '{address,city}' FROM app.app_user;

-- 判断包含
SELECT * FROM app.app_user WHERE profile @> '{"level": 3}';
SELECT * FROM app.app_user WHERE profile ? 'email';
SELECT * FROM app.app_user WHERE profile ?| ARRAY['email', 'phone'];
SELECT * FROM app.app_user WHERE profile ?& ARRAY['email', 'phone'];

-- 修改
UPDATE app.app_user
SET profile = jsonb_set(profile, '{city}', '"Shanghai"', true)
WHERE id = 1;

-- 删除键
UPDATE app.app_user SET profile = profile - 'city' WHERE id = 1;

-- 展开数组/对象
SELECT jsonb_array_elements(profile -> 'tags') FROM app.app_user;
SELECT key, value FROM app.app_user, jsonb_each(profile);

-- 构造
SELECT jsonb_build_object('id', id, 'name', username) FROM app.app_user;
SELECT jsonb_pretty(profile) FROM app.app_user LIMIT 1;

-- 索引
CREATE INDEX idx_app_user_profile_gin ON app.app_user USING GIN (profile);
CREATE INDEX idx_app_user_profile_path_gin
    ON app.app_user USING GIN (profile jsonb_path_ops);
```

### 11.15 数组

```sql
SELECT ARRAY[1,2,3];
SELECT array_agg(id) FROM app.app_user;
SELECT unnest(ARRAY[1,2,3]);

SELECT * FROM app.app_user WHERE tags @> ARRAY['vip'];
SELECT * FROM app.app_user WHERE tags && ARRAY['vip', 'new'];

SELECT array_length(tags, 1) FROM app.app_user;
SELECT array_position(tags, 'vip') FROM app.app_user;
SELECT array_remove(tags, 'vip') FROM app.app_user;
SELECT array_append(tags, 'vip') FROM app.app_user;
SELECT array_to_string(tags, ',') FROM app.app_user;
SELECT string_to_array('a,b,c', ',');

CREATE INDEX idx_app_user_tags_gin ON app.app_user USING GIN (tags);
```

### 11.16 全文检索

```sql
SELECT to_tsvector('simple', 'PostgreSQL 全文检索');
SELECT to_tsquery('simple', 'PostgreSQL & 检索');
SELECT plainto_tsquery('simple', 'PostgreSQL 检索');
SELECT websearch_to_tsquery('simple', 'postgresql or 检索');

SELECT id, content
FROM documents
WHERE to_tsvector('simple', content) @@ plainto_tsquery('simple', 'postgresql');

SELECT id,
       ts_rank(to_tsvector('simple', content),
               plainto_tsquery('simple', 'postgresql')) AS rank
FROM documents
WHERE to_tsvector('simple', content) @@ plainto_tsquery('simple', 'postgresql')
ORDER BY rank DESC
LIMIT 10;

CREATE INDEX idx_documents_fts
    ON documents USING GIN (to_tsvector('simple', content));
```

中文全文检索建议用 `zhparser` 或 `pg_jieba` 扩展，内置 parser 对中文分词效果有限。

### 11.17 生成序列

```sql
SELECT generate_series(1, 10);
SELECT generate_series(1, 10, 2);
SELECT generate_series('2026-01-01'::date, '2026-12-01'::date, '1 month');
SELECT * FROM generate_series(1, 3) AS g(id);
```

### 11.18 执行计划

```sql
EXPLAIN SELECT * FROM app.orders WHERE user_id = 1;

EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT * FROM app.orders WHERE user_id = 1;

EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT * FROM app.orders WHERE user_id = 1;
```

关注：

- `Seq Scan`：全表扫描，小表正常，大表要检查索引。
- `Index Scan` / `Index Only Scan`：走索引。
- `Bitmap Heap Scan`：位图索引扫描。
- `Nested Loop` / `Hash Join` / `Merge Join`：连接方式。
- `rows` 估算与实际差距大：执行 `ANALYZE`。
- `Buffers: shared read` 高：缓存未命中。

### 11.19 行锁查询

```sql
SELECT * FROM app.orders WHERE id = 1 FOR UPDATE;
SELECT * FROM app.orders WHERE id = 1 FOR NO KEY UPDATE;
SELECT * FROM app.orders WHERE id = 1 FOR SHARE;
SELECT * FROM app.orders WHERE id = 1 FOR KEY SHARE;

-- 跳过已被锁定的行，适合任务队列
SELECT * FROM app.jobs
WHERE status = 'pending'
ORDER BY created_at
LIMIT 10
FOR UPDATE SKIP LOCKED;

-- 锁不到立即报错
SELECT * FROM app.orders WHERE id = 1 FOR UPDATE NOWAIT;
```

### 11.20 抽样

```sql
SELECT * FROM app.orders TABLESAMPLE SYSTEM (1);   -- 约 1%
SELECT * FROM app.orders TABLESAMPLE BERNOULLI (1);
```

### 11.21 游标

```sql
BEGIN;

DECLARE c_orders CURSOR FOR
SELECT id, order_no, amount
FROM app.orders
ORDER BY id;

FETCH 10 FROM c_orders;
FETCH NEXT FROM c_orders;
MOVE 10 IN c_orders;
FETCH ALL FROM c_orders;

CLOSE c_orders;
COMMIT;
```

游标默认只在当前事务内有效。需要跨事务使用时加 `WITH HOLD`：

```sql
BEGIN;
DECLARE c_users CURSOR WITH HOLD FOR
SELECT id, username FROM app.app_user ORDER BY id;
COMMIT;

FETCH 20 FROM c_users;
CLOSE c_users;
```

### 11.22 预处理语句

```sql
PREPARE find_user (varchar) AS
SELECT id, username, email
FROM app.app_user
WHERE username = $1;

EXECUTE find_user('alice');

DEALLOCATE find_user;
DEALLOCATE ALL;
```

`PREPARE` 是会话级的，连接断开后自动失效。JDBC 的 `PreparedStatement` 是另一套机制，由驱动管理。

### 11.23 LISTEN / NOTIFY

```sql
-- 会话 A 订阅
LISTEN app_events;

-- 会话 B 发送通知
NOTIFY app_events, '{"type":"order_created","id":1001}';
SELECT pg_notify('app_events', 'hello');

-- 会话 A 读取通知（没有通知时会阻塞等待）
-- psql 会自动打印异步通知

-- 取消订阅
UNLISTEN app_events;
UNLISTEN *;
```

`NOTIFY` 的 payload 建议控制在 8000 字节以内。它适合发信号，不适合传大量业务数据。

---

## 十二、事务、锁与并发

### 12.1 基本事务

```sql
BEGIN;
UPDATE app.accounts SET balance = balance - 100 WHERE id = 1;
UPDATE app.accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

回滚：

```sql
BEGIN;
DELETE FROM app.orders WHERE id = 1;
ROLLBACK;
```

等价写法：

```sql
START TRANSACTION;
...
END;        -- 等同 COMMIT
ABORT;      -- 等同 ROLLBACK
```

### 12.2 保存点

```sql
BEGIN;
INSERT INTO app.app_user (username) VALUES ('alice');
SAVEPOINT sp1;
INSERT INTO app.app_user (username) VALUES ('bob');
ROLLBACK TO SAVEPOINT sp1;
RELEASE SAVEPOINT sp1;
COMMIT;
```

### 12.3 隔离级别

```sql
BEGIN ISOLATION LEVEL READ COMMITTED;
BEGIN ISOLATION LEVEL REPEATABLE READ;
BEGIN ISOLATION LEVEL SERIALIZABLE;

SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

| 级别 | 脏读 | 不可重复读 | 幻读 | 说明 |
|---|---|---|---|---|
| `READ UNCOMMITTED` | 允许（PG 实际按 RC 处理） | 允许 | 允许 | PostgreSQL 不支持真正的脏读 |
| `READ COMMITTED` | 不允许 | 允许 | 允许 | 默认级别 |
| `REPEATABLE READ` | 不允许 | 不允许 | 不允许（PG 实现） | 可能报序列化错误 |
| `SERIALIZABLE` | 不允许 | 不允许 | 不允许 | 最严格，可能报 `40001` |

### 12.4 只读事务

```sql
BEGIN READ ONLY;
SELECT ...;
COMMIT;

BEGIN ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;
```

只读事务里执行 `INSERT/UPDATE/DELETE` 会报错。

### 12.5 SET LOCAL

```sql
BEGIN;
SET LOCAL statement_timeout = '5s';
SET LOCAL search_path = app, public;
...
COMMIT;
```

`SET LOCAL` 只对当前事务有效，事务结束自动恢复。

### 12.6 LOCK TABLE

```sql
LOCK TABLE app.orders IN ACCESS SHARE MODE;
LOCK TABLE app.orders IN ROW SHARE MODE;
LOCK TABLE app.orders IN ROW EXCLUSIVE MODE;
LOCK TABLE app.orders IN SHARE UPDATE EXCLUSIVE MODE;
LOCK TABLE app.orders IN SHARE MODE;
LOCK TABLE app.orders IN SHARE ROW EXCLUSIVE MODE;
LOCK TABLE app.orders IN EXCLUSIVE MODE;
LOCK TABLE app.orders IN ACCESS EXCLUSIVE MODE;

LOCK TABLE app.orders IN ACCESS EXCLUSIVE MODE NOWAIT;
```

锁强度从弱到强：

```text
ACCESS SHARE
ROW SHARE
ROW EXCLUSIVE
SHARE UPDATE EXCLUSIVE
SHARE
SHARE ROW EXCLUSIVE
EXCLUSIVE
ACCESS EXCLUSIVE
```

`ALTER TABLE`、`DROP TABLE`、`TRUNCATE`、`VACUUM FULL` 通常需要 `ACCESS EXCLUSIVE`，会阻塞读写。

### 12.7 咨询锁

```sql
-- 会话级锁
SELECT pg_advisory_lock(12345);
SELECT pg_try_advisory_lock(12345);
SELECT pg_advisory_unlock(12345);
SELECT pg_advisory_unlock_all();

-- 事务级锁，事务结束自动释放
SELECT pg_advisory_xact_lock(12345);
SELECT pg_try_advisory_xact_lock(12345);
```

适合做分布式任务互斥，不占用表锁。

### 12.8 死锁与序列化失败

```sql
-- 查看当前锁等待
SELECT pid,
       pg_blocking_pids(pid) AS blocking_pids,
       state,
       wait_event_type,
       wait_event,
       query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;

-- 取消查询
SELECT pg_cancel_backend(pid);

-- 强制终止连接
SELECT pg_terminate_backend(pid);
```

应用层要处理两类可重试错误：

| SQLSTATE | 含义 | 处理 |
|---|---|---|
| `40001` | 序列化失败 | 回滚后重试整个事务 |
| `40P01` | 死锁检测 | 回滚后重试，并统一加锁顺序 |

### 12.9 两阶段提交

```sql
-- 需要 max_prepared_transactions > 0
BEGIN;
UPDATE app.accounts SET balance = balance - 100 WHERE id = 1;
PREPARE TRANSACTION 'tx_001';

COMMIT PREPARED 'tx_001';
ROLLBACK PREPARED 'tx_001';

-- 查看未完成事务
SELECT * FROM pg_prepared_xacts;
```

单机业务通常用不到；分布式事务场景才考虑。

### 12.10 事务里的限制

以下命令不能在事务块中执行：

- `CREATE DATABASE`
- `DROP DATABASE`
- `CREATE INDEX CONCURRENTLY`
- `REINDEX ... CONCURRENTLY`
- `VACUUM`
- `ALTER SYSTEM`
- `CREATE TABLESPACE`

DDL 在 PostgreSQL 中是事务性的：`BEGIN; CREATE TABLE ...; ROLLBACK;` 会撤销建表。

---

## 十三、视图、物化视图、序列

### 13.1 普通视图

```sql
CREATE VIEW app.v_active_user AS
SELECT id, username, email, created_at
FROM app.app_user
WHERE status = 1;

CREATE OR REPLACE VIEW app.v_active_user AS
SELECT id, username, email, created_at, updated_at
FROM app.app_user
WHERE status = 1;

-- 带检查选项：通过视图写入的数据必须满足视图条件
CREATE VIEW app.v_active_user AS
SELECT * FROM app.app_user WHERE status = 1
WITH CHECK OPTION;

SELECT * FROM app.v_active_user;
\d+ app.v_active_user
\sv app.v_active_user

DROP VIEW app.v_active_user;
DROP VIEW IF EXISTS app.v_active_user CASCADE;
```

### 13.2 物化视图

```sql
CREATE MATERIALIZED VIEW app.mv_order_summary AS
SELECT tenant_id,
       date_trunc('day', created_at) AS day,
       count(*) AS order_count,
       sum(amount) AS total_amount
FROM app.orders
GROUP BY tenant_id, date_trunc('day', created_at);

-- 刷新（锁表）
REFRESH MATERIALIZED VIEW app.mv_order_summary;

-- 并发刷新：必须先有唯一索引
CREATE UNIQUE INDEX uk_mv_order_summary
    ON app.mv_order_summary (tenant_id, day);

REFRESH MATERIALIZED VIEW CONCURRENTLY app.mv_order_summary;

-- 查看
\dm
SELECT * FROM app.mv_order_summary;

DROP MATERIALIZED VIEW app.mv_order_summary;
```

`REFRESH ... CONCURRENTLY` 不会阻塞读取，但第一次刷新和没有唯一索引时不能用。

### 13.3 序列

```sql
CREATE SEQUENCE app.order_seq
    START WITH 1000
    INCREMENT BY 1
    MINVALUE 1
    MAXVALUE 999999999
    CACHE 10
    NO CYCLE;

SELECT nextval('app.order_seq');
SELECT currval('app.order_seq');
SELECT last_value FROM app.order_seq;

-- 设置当前值；第三个参数 true 表示下一次 nextval 返回 1001
SELECT setval('app.order_seq', 1000, true);

ALTER SEQUENCE app.order_seq RESTART WITH 1;
ALTER SEQUENCE app.order_seq INCREMENT BY 2;
ALTER SEQUENCE app.order_seq OWNED BY app.orders.id;

DROP SEQUENCE app.order_seq;
```

查看序列：

```sql
\ds
\d app.order_seq
```

### 13.4 自增列与序列的关系

```sql
-- 查看某列关联的序列
SELECT pg_get_serial_sequence('app.orders', 'id');

-- 修正自增列当前值（导入数据后常见）
SELECT setval(
    pg_get_serial_sequence('app.orders', 'id'),
    (SELECT max(id) FROM app.orders),
    true
);
```

---

## 十四、函数、存储过程、触发器

### 14.1 SQL 函数

```sql
CREATE OR REPLACE FUNCTION app.add_one(i integer)
RETURNS integer
LANGUAGE sql
IMMUTABLE
AS $$
    SELECT i + 1;
$$;

SELECT app.add_one(1);
```

### 14.2 PL/pgSQL 函数

```sql
CREATE OR REPLACE FUNCTION app.upsert_user(p_name text, p_email text)
RETURNS bigint
LANGUAGE plpgsql
AS $$
DECLARE
    v_id bigint;
BEGIN
    INSERT INTO app.app_user (username, email)
    VALUES (p_name, p_email)
    ON CONFLICT (username)
    DO UPDATE SET email = EXCLUDED.email,
                  updated_at = now()
    RETURNING id INTO v_id;

    RETURN v_id;
END;
$$;

SELECT app.upsert_user('alice', 'alice@example.com');
```

### 14.3 返回结果集

```sql
CREATE OR REPLACE FUNCTION app.users_by_status(p_status smallint)
RETURNS TABLE (
    id bigint,
    username varchar,
    email varchar
)
LANGUAGE sql
STABLE
AS $$
    SELECT id, username, email
    FROM app.app_user
    WHERE status = p_status;
$$;

SELECT * FROM app.users_by_status(1);
```

### 14.4 函数易变性

| 标记 | 含义 |
|---|---|
| `IMMUTABLE` | 相同输入永远相同输出，不查表、不改数据 |
| `STABLE` | 同一语句内结果稳定，可以查表 |
| `VOLATILE` | 每次调用都可能不同，默认值 |

标记错误会影响索引和查询计划，写函数时不要随便标 `IMMUTABLE`。

### 14.5 SECURITY DEFINER

```sql
CREATE OR REPLACE FUNCTION app.get_user_count()
RETURNS bigint
LANGUAGE sql
SECURITY DEFINER
SET search_path = app, public
AS $$
    SELECT count(*) FROM app.app_user;
$$;
```

`SECURITY DEFINER` 以函数所有者的权限执行，必须固定 `search_path`，否则有提权风险。

### 14.6 删除和查看函数

```sql
DROP FUNCTION app.add_one(integer);
DROP FUNCTION IF EXISTS app.add_one(integer);

\df app.*
\df+ app.upsert_user
\sf app.upsert_user

SELECT pg_get_functiondef('app.upsert_user(text,text)'::regprocedure);
```

### 14.7 存储过程

```sql
CREATE OR REPLACE PROCEDURE app.close_orders(p_tenant_id bigint)
LANGUAGE plpgsql
AS $$
BEGIN
    UPDATE app.orders
    SET status = 'closed',
        updated_at = now()
    WHERE tenant_id = p_tenant_id
      AND status = 'pending';

    COMMIT;
END;
$$;

CALL app.close_orders(1001);

DROP PROCEDURE app.close_orders(bigint);
```

存储过程内部可以 `COMMIT`/`ROLLBACK`，函数不可以。

### 14.8 触发器函数

```sql
CREATE OR REPLACE FUNCTION app.set_updated_at()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.updated_at := now();
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_app_user_updated_at
BEFORE UPDATE ON app.app_user
FOR EACH ROW
EXECUTE FUNCTION app.set_updated_at();
```

审计触发器示例：

```sql
CREATE TABLE app.audit_log (
    id          bigserial PRIMARY KEY,
    table_name  text NOT NULL,
    operation   text NOT NULL,
    row_id      bigint,
    old_data    jsonb,
    new_data    jsonb,
    changed_by  text DEFAULT current_user,
    changed_at  timestamptz DEFAULT now()
);

CREATE OR REPLACE FUNCTION app.audit_trigger()
RETURNS trigger
LANGUAGE plpgsql
AS $$
DECLARE
    v_row_id bigint;
    v_old    jsonb;
    v_new    jsonb;
BEGIN
    IF TG_OP = 'DELETE' THEN
        v_row_id := OLD.id;
        v_old := to_jsonb(OLD);
        v_new := NULL;
    ELSIF TG_OP = 'UPDATE' THEN
        v_row_id := NEW.id;
        v_old := to_jsonb(OLD);
        v_new := to_jsonb(NEW);
    ELSE
        v_row_id := NEW.id;
        v_old := NULL;
        v_new := to_jsonb(NEW);
    END IF;

    INSERT INTO app.audit_log(table_name, operation, row_id, old_data, new_data)
    VALUES (TG_TABLE_NAME, TG_OP, v_row_id, v_old, v_new);

    IF TG_OP = 'DELETE' THEN
        RETURN OLD;
    END IF;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_app_user_audit
AFTER INSERT OR UPDATE OR DELETE ON app.app_user
FOR EACH ROW
EXECUTE FUNCTION app.audit_trigger();
```

触发器变量：`TG_OP`、`TG_TABLE_NAME`、`TG_WHEN`、`TG_LEVEL`。

### 14.9 管理触发器

```sql
-- 查看
\d app.app_user

SELECT tgname,
       pg_get_triggerdef(oid) AS definition
FROM pg_trigger
WHERE tgrelid = 'app.app_user'::regclass
  AND NOT tgisinternal;

-- 禁用/启用
ALTER TABLE app.app_user DISABLE TRIGGER trg_app_user_audit;
ALTER TABLE app.app_user ENABLE TRIGGER trg_app_user_audit;
ALTER TABLE app.app_user DISABLE TRIGGER ALL;
ALTER TABLE app.app_user ENABLE TRIGGER ALL;

-- 删除
DROP TRIGGER trg_app_user_audit ON app.app_user;
```

### 14.10 外部表与 postgres_fdw

```sql
CREATE EXTENSION IF NOT EXISTS postgres_fdw;

CREATE SERVER remote_pg
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host '127.0.0.1', port '5432', dbname 'otherdb');

CREATE USER MAPPING FOR postgres
    SERVER remote_pg
    OPTIONS (user 'postgres', password 'postgres');

CREATE FOREIGN TABLE app.remote_orders (
    id bigint,
    amount numeric(12,2),
    created_at timestamptz
)
SERVER remote_pg
OPTIONS (schema_name 'public', table_name 'orders');

SELECT * FROM app.remote_orders LIMIT 10;

-- 直接导入外部 schema 的所有表
IMPORT FOREIGN SCHEMA public
    FROM SERVER remote_pg
    INTO app;
```

### 14.11 逻辑复制（了解）

发布端：

```sql
-- postgresql.conf: wal_level = logical
CREATE PUBLICATION pub_orders
FOR TABLE app.orders;

SELECT * FROM pg_publication;
SELECT * FROM pg_publication_tables;
```

订阅端：

```sql
CREATE SUBSCRIPTION sub_orders
CONNECTION 'host=192.168.1.10 port=5432 dbname=vectordb user=postgres password=postgres'
PUBLICATION pub_orders;

SELECT * FROM pg_stat_subscription;

DROP SUBSCRIPTION sub_orders;
DROP PUBLICATION pub_orders;
```

逻辑复制只同步 DML，不同步 DDL；表结构需要两端自己维护一致。

---

## 十五、运维监控

### 15.1 版本与运行信息

```sql
SELECT version();
SELECT current_database(), current_user, now();
SHOW server_version;
SHOW server_version_num;
SHOW data_directory;
SHOW config_file;
SHOW hba_file;
SHOW timezone;
SELECT pg_postmaster_start_time();
SELECT now() - pg_postmaster_start_time() AS uptime;
```

```powershell
docker exec pgvector pg_config --version
docker exec pgvector psql --version
docker exec pgvector pg_isready -U postgres -d vectordb
```

### 15.2 当前会话与活动查询

```sql
SELECT pid,
       datname,
       usename,
       application_name,
       client_addr,
       state,
       wait_event_type,
       wait_event,
       backend_start,
       xact_start,
       query_start,
       state_change,
       now() - query_start AS query_duration,
       left(query, 200) AS query
FROM pg_stat_activity
ORDER BY query_start;
```

只看非空闲连接：

```sql
SELECT pid,
       usename,
       datname,
       state,
       now() - query_start AS duration,
       left(query, 300) AS query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY duration DESC;
```

查看超过 5 分钟的查询：

```sql
SELECT pid,
       now() - query_start AS duration,
       state,
       query
FROM pg_stat_activity
WHERE state = 'active'
  AND now() - query_start > interval '5 minutes'
ORDER BY duration DESC;
```

查看长事务（包括 idle in transaction）：

```sql
SELECT pid,
       state,
       now() - xact_start AS xact_duration,
       now() - state_change AS idle_duration,
       query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_duration DESC;
```

按状态统计：

```sql
SELECT state, count(*)
FROM pg_stat_activity
GROUP BY state
ORDER BY count(*) DESC;
```

按应用统计：

```sql
SELECT application_name, count(*)
FROM pg_stat_activity
GROUP BY application_name
ORDER BY count(*) DESC;
```

### 15.3 连接数

```sql
SHOW max_connections;
SHOW superuser_reserved_connections;

SELECT count(*) AS total,
       count(*) FILTER (WHERE state = 'active') AS active,
       count(*) FILTER (WHERE state = 'idle') AS idle,
       count(*) FILTER (WHERE state = 'idle in transaction') AS idle_in_tx
FROM pg_stat_activity;

SELECT usename, datname, count(*)
FROM pg_stat_activity
GROUP BY usename, datname
ORDER BY count(*) DESC;
```

按数据库查看连接上限：

```sql
SELECT datname, datconnlimit
FROM pg_database
ORDER BY datname;
```

### 15.4 锁与阻塞

查看锁：

```sql
SELECT l.pid,
       l.locktype,
       l.mode,
       l.granted,
       l.relation::regclass AS relation,
       a.usename,
       a.state,
       left(a.query, 200) AS query
FROM pg_locks l
LEFT JOIN pg_stat_activity a ON a.pid = l.pid
ORDER BY l.granted, l.pid;
```

只看未授予的锁：

```sql
SELECT locktype,
       relation::regclass AS relation,
       mode,
       granted,
       pid
FROM pg_locks
WHERE NOT granted;
```

查找阻塞链：

```sql
SELECT blocked.pid        AS blocked_pid,
       blocked.usename    AS blocked_user,
       blocking.pid       AS blocking_pid,
       blocking.usename   AS blocking_user,
       blocked.wait_event_type,
       blocked.wait_event,
       left(blocked.query, 200)  AS blocked_query,
       left(blocking.query, 200) AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.pid <> pg_backend_pid();
```

终止会话：

```sql
-- 取消当前查询，连接保留
SELECT pg_cancel_backend(12345);

-- 强制断开连接
SELECT pg_terminate_backend(12345);

-- 终止某个库的所有其他连接
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'mydb'
  AND pid <> pg_backend_pid();
```

### 15.5 数据库、表、索引大小

```sql
-- 数据库大小
SELECT datname,
       pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;

-- 当前库所有表大小
SELECT schemaname,
       relname AS table_name,
       pg_size_pretty(pg_total_relation_size(relid)) AS total_size,
       pg_size_pretty(pg_relation_size(relid))       AS table_size,
       pg_size_pretty(pg_indexes_size(relid))        AS index_size
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(relid) DESC;

-- 单表
SELECT pg_size_pretty(pg_total_relation_size('app.orders')) AS total,
       pg_size_pretty(pg_relation_size('app.orders'))       AS table_only,
       pg_size_pretty(pg_indexes_size('app.orders'))        AS indexes;
```

按 schema 汇总：

```sql
SELECT schemaname,
       count(*) AS table_count,
       pg_size_pretty(sum(pg_total_relation_size(relid))) AS total_size
FROM pg_stat_user_tables
GROUP BY schemaname
ORDER BY sum(pg_total_relation_size(relid)) DESC;
```

单行/单列大小：

```sql
SELECT pg_column_size(profile) FROM app.app_user LIMIT 5;
SELECT pg_column_size(t.*) FROM app.app_user t LIMIT 5;
```

### 15.6 缓存命中率

```sql
SELECT datname,
       blks_read,
       blks_hit,
       round(blks_hit * 100.0 / nullif(blks_hit + blks_read, 0), 2) AS cache_hit_pct
FROM pg_stat_database
WHERE datname = current_database();

-- 表级
SELECT relname,
       heap_blks_read,
       heap_blks_hit,
       round(heap_blks_hit * 100.0 /
             nullif(heap_blks_hit + heap_blks_read, 0), 2) AS hit_pct
FROM pg_statio_user_tables
ORDER BY heap_blks_read DESC;
```

OLTP 场景缓存命中率通常希望高于 99%。低于 95% 要检查 `shared_buffers`、内存和慢查询。

### 15.7 扫描方式统计

```sql
SELECT relname,
       seq_scan,
       seq_tup_read,
       idx_scan,
       idx_tup_fetch,
       n_live_tup,
       n_dead_tup
FROM pg_stat_user_tables
ORDER BY seq_scan DESC;
```

大表 `seq_scan` 很高且 `idx_scan` 很低，说明可能缺索引或索引失效。

### 15.8 表膨胀与死元组

```sql
SELECT relname,
       n_live_tup,
       n_dead_tup,
       round(n_dead_tup * 100.0 / nullif(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
       last_vacuum,
       last_autovacuum,
       last_analyze,
       last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

`n_dead_tup` 持续很高说明 autovacuum 跟不上，可能需要：

- 调低 `autovacuum_vacuum_scale_factor`
- 提高 `autovacuum_vacuum_cost_limit`
- 对重点表单独设置 autovacuum 参数

### 15.9 VACUUM 与 ANALYZE

```sql
-- 日常维护，锁级别低
VACUUM app.orders;
VACUUM (ANALYZE) app.orders;
VACUUM (VERBOSE, ANALYZE) app.orders;

-- 只更新统计信息
ANALYZE app.orders;
ANALYZE VERBOSE app.orders;

-- 冻结旧事务 ID，防止 wraparound
VACUUM (FREEZE, ANALYZE) app.orders;

-- 回收空间，会锁表，生产慎用
VACUUM FULL app.orders;

-- 整库
VACUUM (ANALYZE);
```

查看事务 ID 年龄：

```sql
SELECT datname,
       age(datfrozenxid) AS xid_age,
       current_setting('autovacuum_freeze_max_age')::bigint AS freeze_max_age
FROM pg_database
ORDER BY age(datfrozenxid) DESC;
```

### 15.10 索引维护

```sql
-- 查看索引大小和使用
SELECT indexrelname,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size,
       idx_scan,
       idx_tup_read
FROM pg_stat_user_indexes
ORDER BY idx_scan;

-- 重建
REINDEX INDEX app.idx_orders_status;
REINDEX TABLE app.orders;
REINDEX SCHEMA app;
REINDEX DATABASE vectordb;

-- 并发重建
REINDEX INDEX CONCURRENTLY app.idx_orders_status;

-- 聚簇
CLUSTER app.orders USING idx_orders_created_at;
```

### 15.11 进度视图

```sql
SELECT * FROM pg_stat_progress_vacuum;
SELECT * FROM pg_stat_progress_create_index;
SELECT * FROM pg_stat_progress_analyze;
SELECT * FROM pg_stat_progress_copy;
SELECT * FROM pg_stat_progress_basebackup;
```

这些视图能看到当前 VACUUM、建索引、ANALYZE、COPY、基础备份进行到哪一步。

### 15.12 pg_stat_statements

安装：

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

启用需要重启（`postgresql.conf`）：

```text
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.max = 10000
pg_stat_statements.track = all
```

```powershell
docker restart pgvector
```

常用查询：

```sql
-- 总耗时最高的 SQL
SELECT queryid,
       calls,
       round(total_exec_time::numeric, 2) AS total_ms,
       round(mean_exec_time::numeric, 2) AS mean_ms,
       rows,
       left(query, 200) AS query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;

-- 平均耗时最高
SELECT queryid,
       calls,
       round(mean_exec_time::numeric, 2) AS mean_ms,
       left(query, 200) AS query
FROM pg_stat_statements
WHERE calls > 10
ORDER BY mean_exec_time DESC
LIMIT 20;

-- 重置统计
SELECT pg_stat_statements_reset();
```

### 15.13 数据库级统计

```sql
SELECT datname,
       numbackends,
       xact_commit,
       xact_rollback,
       blks_read,
       blks_hit,
       tup_returned,
       tup_fetched,
       tup_inserted,
       tup_updated,
       tup_deleted,
       conflicts,
       deadlocks,
       temp_files,
       pg_size_pretty(temp_bytes) AS temp_size
FROM pg_stat_database
ORDER BY datname;
```

### 15.14 WAL 与检查点

```sql
SELECT pg_current_wal_lsn();
SELECT pg_walfile_name(pg_current_wal_lsn());
SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), '0/0');

SELECT pg_switch_wal();
SELECT pg_create_restore_point('before_migration');
CHECKPOINT;

SHOW wal_level;
SHOW max_wal_size;
SHOW min_wal_size;
SHOW checkpoint_timeout;
SHOW checkpoint_completion_target;
SHOW archive_mode;
SHOW archive_command;
```

手动 `CHECKPOINT` 会增加 I/O，生产环境不要频繁执行。

### 15.15 复制状态

```sql
SELECT * FROM pg_stat_replication;
SELECT * FROM pg_stat_wal_receiver;
SELECT * FROM pg_stat_archiver;

-- 主库上查看复制槽
SELECT * FROM pg_replication_slots;
```

### 15.16 日志

常用配置：

```sql
ALTER SYSTEM SET log_min_duration_statement = '1s';
ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM SET log_statement = 'ddl';
ALTER SYSTEM SET log_line_prefix = '%m [%p] %u@%d ';
SELECT pg_reload_conf();

SHOW log_directory;
SHOW log_filename;
SHOW log_min_duration_statement;
```

容器日志：

```powershell
docker logs --tail 200 pgvector
docker logs -f pgvector
```

### 15.17 常用对象定义查询

```sql
SELECT pg_get_viewdef('app.v_active_user'::regclass, true);
SELECT pg_get_indexdef('app.idx_orders_status'::regclass);
SELECT pg_get_constraintdef(oid)
FROM pg_constraint
WHERE conrelid = 'app.orders'::regclass;
SELECT pg_get_functiondef('app.upsert_user(text,text)'::regprocedure);
SELECT pg_get_triggerdef(oid)
FROM pg_trigger
WHERE tgrelid = 'app.orders'::regclass;
```

### 15.18 集群控制与 WAL 检查

容器内执行：

```bash
pg_controldata "$PGDATA"
pg_controldata "$PGDATA" | grep -E 'state|checkpoint|REDO'

# 查看 WAL 文件
ls -lh "$PGDATA/pg_wal"

# 解析单个 WAL 文件（只读诊断）
pg_waldump "$PGDATA/pg_wal/000000010000000000000001" | head -50
```

`pg_controldata` 能看到集群状态、最新检查点、REDO LSN、数据页校验和等信息。集群无法启动时，它常用于判断数据目录是否完整。

使用 `pg_ctl` 需要以 `postgres` 用户运行：

```bash
pg_ctl status -D "$PGDATA"
pg_ctl reload -D "$PGDATA"
```

Docker 部署优先用 `docker restart pgvector`，不要在容器里随意 `pg_ctl stop`，否则入口脚本可能会重新初始化或容器直接退出。

---

## 十六、配置与调优

### 16.1 配置文件位置

```sql
SHOW config_file;
SHOW hba_file;
SHOW ident_file;
SHOW data_directory;
```

容器内通常位于：

```text
/var/lib/postgresql/data/postgresql.conf
/var/lib/postgresql/data/pg_hba.conf
/var/lib/postgresql/data/pg_ident.conf
```

Docker 镜像的 `PGDATA` 默认是 `/var/lib/postgresql/data`。PostgreSQL 18 镜像改成了 `/var/lib/postgresql/18/docker`，挂载点也相应变化。

### 16.2 查看参数

```sql
SHOW all;
SHOW shared_buffers;
SHOW work_mem;
SHOW max_connections;
SHOW timezone;

SELECT current_setting('shared_buffers');
SELECT current_setting('max_connections')::int;

SELECT name, setting, unit, category, short_desc
FROM pg_settings
WHERE name IN (
    'shared_buffers',
    'work_mem',
    'maintenance_work_mem',
    'effective_cache_size',
    'max_connections'
);

SELECT name, setting, unit, boot_val, reset_val, source
FROM pg_settings
WHERE name LIKE '%timeout%'
ORDER BY name;
```

### 16.3 修改参数

会话级：

```sql
SET work_mem = '32MB';
SET statement_timeout = '30s';
SET search_path = app, public;
RESET work_mem;
RESET ALL;
```

数据库级：

```sql
ALTER DATABASE vectordb SET timezone = 'Asia/Shanghai';
ALTER DATABASE vectordb SET statement_timeout = '30s';
ALTER DATABASE vectordb RESET statement_timeout;
```

角色级：

```sql
ALTER ROLE appuser SET work_mem = '16MB';
ALTER ROLE appuser IN DATABASE vectordb SET search_path = app, public;
ALTER ROLE appuser RESET work_mem;
```

全局级：

```sql
ALTER SYSTEM SET work_mem = '16MB';
ALTER SYSTEM RESET work_mem;
SELECT pg_reload_conf();
```

`ALTER SYSTEM` 写入 `postgresql.auto.conf`，优先级高于 `postgresql.conf`。

### 16.4 生效方式

| 方式 | 命令 | 是否重启 |
|---|---|---|
| 会话级 | `SET` | 否 |
| 数据库/角色级 | `ALTER DATABASE/ROLE ... SET` | 新连接生效 |
| 全局动态参数 | `ALTER SYSTEM` + `pg_reload_conf()` | 否 |
| 全局静态参数 | `ALTER SYSTEM` + `docker restart pgvector` | 是 |

```sql
SELECT pg_reload_conf();
```

```powershell
docker restart pgvector
```

### 16.5 关键参数

| 参数 | 作用 | 建议 |
|---|---|---|
| `max_connections` | 最大连接数 | 按应用连接池总量设置，不是越大越好 |
| `shared_buffers` | 共享缓冲区 | 物理内存的 25% 左右 |
| `work_mem` | 每个排序/哈希操作的内存 | 从 `4MB`~`16MB` 起 |
| `maintenance_work_mem` | VACUUM、建索引内存 | `256MB`~`1GB` |
| `effective_cache_size` | 规划器估算的可用缓存 | 物理内存的 50%~75% |
| `random_page_cost` | 随机读成本 | SSD 可降到 `1.1`~`2.0` |
| `effective_io_concurrency` | 并发 I/O 能力 | SSD 可设 `100`~`200` |
| `wal_level` | WAL 级别 | 复制用 `replica`，逻辑复制用 `logical` |
| `max_wal_size` | WAL 上限 | 写入量大时适当增大 |
| `checkpoint_timeout` | 检查点间隔 | 通常 `15min`~`30min` |
| `autovacuum` | 自动清理开关 | 保持 `on` |
| `statement_timeout` | SQL 超时 | 按接口设置，避免慢查询堆积 |
| `lock_timeout` | 等锁超时 | 建议设置，避免无限等待 |
| `idle_in_transaction_session_timeout` | 空闲事务超时 | 建议 `30s`~`5min` |
| `deadlock_timeout` | 死锁检测等待 | 默认 `1s` |
| `log_min_duration_statement` | 慢 SQL 日志阈值 | 生产可设 `500ms`~`2s` |

### 16.6 连接相关参数

```sql
SHOW listen_addresses;
SHOW port;
SHOW max_connections;
SHOW superuser_reserved_connections;
SHOW tcp_keepalives_idle;
SHOW tcp_keepalives_interval;
SHOW tcp_keepalives_count;
```

容器要接受宿主机连接，需要：

1. `listen_addresses = '*'`
2. `pg_hba.conf` 允许来源网段
3. `docker run -p` 做了端口映射
4. 宿主机防火墙放行

### 16.7 pg_hba.conf

查看规则：

```sql
SELECT type, database, user_name, address, auth_method, error
FROM pg_hba_file_rules
ORDER BY line_number;
```

常见条目：

```text
# 本地 Unix socket，所有库所有用户
local   all             all                                     scram-sha-256

# IPv4 本机
host    all             all             127.0.0.1/32            scram-sha-256

# IPv6 本机
host    all             all             ::1/128                 scram-sha-256

# 允许 Docker 网段
host    all             all             172.16.0.0/12           scram-sha-256

# 允许任意来源（仅内网测试）
host    all             all             0.0.0.0/0               scram-sha-256

# 复制连接
host    replication     replicator      10.0.0.0/8              scram-sha-256
```

修改后重载：

```sql
SELECT pg_reload_conf();
```

### 16.8 密码认证

```sql
SHOW password_encryption;
SET password_encryption = 'scram-sha-256';
ALTER USER postgres WITH PASSWORD 'new_password';
```

`pg_hba.conf` 里用 `scram-sha-256`，不要用已废弃的 `md5` 或 `trust`。

### 16.9 SSL

```sql
SHOW ssl;
SHOW ssl_cert_file;
SHOW ssl_key_file;
SHOW ssl_ca_file;
```

客户端强制 SSL：

```text
postgresql://user:pass@host:5432/db?sslmode=require
```

验证证书：

```text
sslmode=verify-full&sslrootcert=C:/certs/ca.crt
```

关闭 SSL：

```text
sslmode=disable
```

### 16.10 超时参数

```sql
SET statement_timeout = '30s';
SET lock_timeout = '5s';
SET idle_in_transaction_session_timeout = '60s';
SET idle_session_timeout = '30min';
SET transaction_timeout = '5min';   -- PostgreSQL 17+

SHOW statement_timeout;
SHOW lock_timeout;
```

建议在应用连接池上设置 `connectionTimeout`，在数据库上设置 `statement_timeout` 和 `idle_in_transaction_session_timeout`，双保险。

### 16.11 日志参数

```sql
ALTER SYSTEM SET log_min_duration_statement = '1s';
ALTER SYSTEM SET log_statement = 'ddl';
ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM SET log_temp_files = '10MB';
ALTER SYSTEM SET log_checkpoints = on;
ALTER SYSTEM SET log_connections = on;
ALTER SYSTEM SET log_disconnections = on;
ALTER SYSTEM SET log_line_prefix = '%m [%p] %u@%d ';
SELECT pg_reload_conf();
```

`log_statement = 'all'` 日志量很大，只适合短期排查。

### 16.12 autovacuum 参数

```sql
SHOW autovacuum;
SHOW autovacuum_naptime;
SHOW autovacuum_vacuum_threshold;
SHOW autovacuum_vacuum_scale_factor;
SHOW autovacuum_analyze_threshold;
SHOW autovacuum_analyze_scale_factor;
SHOW autovacuum_vacuum_cost_limit;
SHOW autovacuum_max_workers;
```

单表覆盖：

```sql
ALTER TABLE app.orders SET (
    autovacuum_vacuum_scale_factor = 0.02,
    autovacuum_analyze_scale_factor = 0.01,
    autovacuum_vacuum_cost_limit = 1000
);

ALTER TABLE app.orders RESET (
    autovacuum_vacuum_scale_factor,
    autovacuum_analyze_scale_factor,
    autovacuum_vacuum_cost_limit
);
```

### 16.13 内存参数估算

`work_mem` 不是全局只分配一次，而是**每个排序/哈希操作**都可能分配。一个复杂查询有多个排序节点时，实际占用可能是 `work_mem × 节点数 × 并发数`。

粗略估算：

```text
shared_buffers        = 物理内存 × 25%
effective_cache_size  = 物理内存 × 50%~75%
maintenance_work_mem  = 256MB ~ 1GB
work_mem              = 剩余内存 / (最大并发 × 每查询排序节点数)
```

### 16.14 查看参数来源

```sql
SELECT name, setting, source, sourcefile, sourceline
FROM pg_settings
WHERE name = 'work_mem';
```

`source` 可能是 `default`、`configuration file`、`environment variable`、`database`、`user`、`session` 等。

---

## 十七、备份、恢复与迁移

### 17.1 逻辑备份与物理备份

| 方式 | 工具 | 特点 |
|---|---|---|
| 逻辑备份 | `pg_dump` / `pg_dumpall` | 跨版本、可选择性恢复、文件较小 |
| 物理备份 | `pg_basebackup` | 整实例复制、恢复快、要求版本/架构一致 |
| 文件级备份 | 卷快照 / `tar` | 简单，必须保证一致性 |

日常业务库优先用 `pg_dump`，大规模实例或需要 PITR 时用物理备份 + WAL 归档。

### 17.2 pg_dump 格式

| 格式 | 参数 | 说明 |
|---|---|---|
| plain | `-Fp` | 纯 SQL，默认，可直接 `psql -f` |
| custom | `-Fc` | 压缩、可选择性恢复，推荐 |
| directory | `-Fd` | 目录格式，支持并行 `-j` |
| tar | `-Ft` | tar 包，较少用 |

### 17.3 备份单个数据库

```powershell
# 纯 SQL，在容器内生成文件
docker exec -t pgvector pg_dump -U postgres -d vectordb -Fp -f /tmp/vectordb.sql
docker cp pgvector:/tmp/vectordb.sql C:\backup\vectordb.sql

# 自定义格式，推荐
docker exec -t pgvector pg_dump -U postgres -d vectordb -Fc -Z 6 -f /tmp/vectordb.dump
docker cp pgvector:/tmp/vectordb.dump C:\backup\vectordb.dump
```

只备份结构：

```powershell
docker exec -t pgvector pg_dump -U postgres -d vectordb -s -f /tmp/vectordb_schema.sql
```

只备份数据：

```powershell
docker exec -t pgvector pg_dump -U postgres -d vectordb -a -f /tmp/vectordb_data.sql
```

只备份指定 schema：

```powershell
docker exec -t pgvector pg_dump -U postgres -d vectordb -n app -Fc -f /tmp/app.dump
```

只备份指定表：

```powershell
docker exec -t pgvector pg_dump -U postgres -d vectordb -t app.app_user -Fc -f /tmp/app_user.dump
```

排除某张表：

```powershell
docker exec -t pgvector pg_dump -U postgres -d vectordb -T app.audit_log -Fc -f /tmp/vectordb.dump
```

并行备份（目录格式）：

```powershell
docker exec -t pgvector pg_dump -U postgres -d vectordb -Fd -j 4 -f /tmp/vectordb_dir
```

### 17.4 常用 pg_dump 选项

| 选项 | 作用 |
|---|---|
| `-s` / `--schema-only` | 只导出结构 |
| `-a` / `--data-only` | 只导出数据 |
| `-c` / `--clean` | 生成 DROP 语句（plain 格式） |
| `--if-exists` | DROP 时加 IF EXISTS |
| `-C` / `--create` | 生成 CREATE DATABASE |
| `--no-owner` | 不导出 owner |
| `--no-privileges` | 不导出权限 |
| `-n <schema>` | 指定 schema |
| `-N <schema>` | 排除 schema |
| `-t <table>` | 指定表 |
| `-T <table>` | 排除表 |
| `--inserts` | 用 INSERT 而不是 COPY |
| `--column-inserts` | 带列名的 INSERT，慢但兼容性好 |
| `--rows-per-insert=N` | 每条 INSERT 多行 |
| `--on-conflict-do-nothing` | 配合 INSERT，冲突忽略 |
| `-Z 0..9` | 压缩级别 |
| `-j N` | 并行，仅 directory 格式 |
| `--encoding=UTF8` | 指定编码 |

### 17.5 备份所有数据库和角色

```powershell
docker exec -t pgvector pg_dumpall -U postgres -f /tmp/all.sql
docker cp pgvector:/tmp/all.sql C:\backup\all.sql
```

只备份全局对象（角色、表空间）：

```powershell
docker exec -t pgvector pg_dumpall -U postgres --globals-only -f /tmp/globals.sql
```

### 17.6 恢复纯 SQL

```powershell
docker cp C:\backup\vectordb.sql pgvector:/tmp/vectordb.sql
docker exec -it pgvector psql -U postgres -d vectordb -v ON_ERROR_STOP=1 -f /tmp/vectordb.sql
```

恢复到新库：

```powershell
docker exec -it pgvector psql -U postgres -c "CREATE DATABASE vectordb_new;"
docker exec -it pgvector psql -U postgres -d vectordb_new -f /tmp/vectordb.sql
```

### 17.7 恢复自定义格式

```powershell
docker cp C:\backup\vectordb.dump pgvector:/tmp/vectordb.dump

# 先看备份内容
docker exec -it pgvector pg_restore -l /tmp/vectordb.dump

# 恢复
docker exec -it pgvector pg_restore -U postgres -d vectordb --clean --if-exists /tmp/vectordb.dump

# 恢复到新库，忽略 owner
docker exec -it pgvector pg_restore -U postgres -d vectordb_new --no-owner --no-privileges /tmp/vectordb.dump

# 只恢复某张表
docker exec -it pgvector pg_restore -U postgres -d vectordb -t app_user /tmp/vectordb.dump
```

并行恢复：

```powershell
docker exec -it pgvector pg_restore -U postgres -d vectordb -j 4 /tmp/vectordb.dump
```

### 17.8 备份文件直接管道恢复

```powershell
docker exec pgvector pg_dump -U postgres -d vectordb -Fc |
    docker exec -i pgvector pg_restore -U postgres -d vectordb_new
```

PowerShell 管道传二进制容易出问题，跨容器/跨机器时建议先落盘再 `docker cp`。

### 17.9 PowerShell 重定向的坑

不要这样备份：

```powershell
docker exec pgvector pg_dump -U postgres -d vectordb > backup.sql
```

Windows PowerShell 5.x 的 `>` 可能写出 UTF-16LE，恢复时出现编码错误。推荐：

```powershell
docker exec -t pgvector pg_dump -U postgres -d vectordb -f /tmp/backup.sql
docker cp pgvector:/tmp/backup.sql C:\backup\backup.sql
```

### 17.10 容器数据卷备份

先停容器，保证一致性：

```powershell
docker stop pgvector

docker run --rm `
  -v pgvector_pgdata:/data `
  -v C:\backup:/backup `
  alpine sh -c "tar czf /backup/pgdata_20261018.tgz -C /data ."

docker start pgvector
```

卷名以 `docker volume ls` 为准，不一定是 `pgvector_pgdata`。生产环境更推荐 `pg_dump` 或 `pg_basebackup`，直接复制数据文件有版本和一致性风险。

### 17.11 物理备份 pg_basebackup

```bash
pg_basebackup \
  -h localhost \
  -p 5432 \
  -U replicator \
  -D /backup/base \
  -Fp \
  -Xs \
  -P \
  -R
```

| 参数 | 作用 |
|---|---|
| `-D` | 输出目录 |
| `-Fp` | plain 格式 |
| `-Ft` | tar 格式 |
| `-Xs` | 流式传输 WAL |
| `-P` | 显示进度 |
| `-R` | 自动写 `standby.signal` 和连接配置 |
| `-c fast` | 快速检查点 |

备份校验：

```bash
pg_verifybackup /backup/base
```

### 17.12 PITR（时间点恢复）要点

主库配置：

```sql
ALTER SYSTEM SET wal_level = 'replica';
ALTER SYSTEM SET archive_mode = on;
ALTER SYSTEM SET archive_command = 'test ! -f /archive/%f && cp %p /archive/%f';
```

重启后确认：

```sql
SHOW archive_mode;
SELECT * FROM pg_stat_archiver;
```

恢复时在数据目录创建 `recovery.signal`，配置：

```text
restore_command = 'cp /archive/%f %p'
recovery_target_time = '2026-10-18 12:00:00+08'
recovery_target_action = 'promote'
```

启动后观察日志，恢复完成会删除 `recovery.signal` 并提升为主库。

### 17.13 pg_upgrade 大版本升级

基本流程：

```bash
pg_upgrade \
  -b /usr/lib/postgresql/17/bin \
  -B /usr/lib/postgresql/18/bin \
  -d /var/lib/postgresql/17/data \
  -D /var/lib/postgresql/18/data \
  --check
```

先跑 `--check`，再正式升级。升级前必须完整备份，升级后执行 `ANALYZE`。容器场景更简单的做法是新起一个高版本容器，用 `pg_dump`/`pg_restore` 迁移。

### 17.14 迁移到新容器

```powershell
# 旧容器导出
docker exec -t pgvector pg_dump -U postgres -d vectordb -Fc -f /tmp/vectordb.dump
docker cp pgvector:/tmp/vectordb.dump C:\backup\vectordb.dump

# 新容器导入
docker cp C:\backup\vectordb.dump pgvector_new:/tmp/vectordb.dump
docker exec -it pgvector_new createdb -U postgres vectordb
docker exec -it pgvector_new pg_restore -U postgres -d vectordb --no-owner /tmp/vectordb.dump
```

如果用了 pgvector 扩展，新镜像也必须是 `pgvector/pgvector:pg17` 这类带扩展的镜像，普通 `postgres:17` 没有 `vector` 扩展。

### 17.15 备份策略建议

| 场景 | 建议 |
|---|---|
| 学习/本地 | 每天 `pg_dump -Fc`，保留 7 天 |
| 小生产 | 每天全备 + WAL 归档，保留 7~30 天 |
| 大生产 | 每周全备 + 每日增量/WAL，异地保存 |
| 升级前 | 全备 + 校验恢复，确认备份可恢复 |

备份没有验证过，就等于没有备份。至少每月做一次恢复演练。

---

## 十八、扩展管理

### 18.1 查看扩展

```sql
\dx
\dx+

SELECT extname, extversion, extrelocatable, nspname AS schema
FROM pg_extension e
JOIN pg_namespace n ON n.oid = e.extnamespace
ORDER BY extname;
```

查看可安装扩展：

```sql
SELECT name, default_version, installed_version, comment
FROM pg_available_extensions
ORDER BY name;

SELECT * FROM pg_available_extension_versions
WHERE name = 'vector';
```

### 18.2 安装与卸载

```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- 指定 schema
CREATE EXTENSION IF NOT EXISTS vector SCHEMA public;

-- 升级
ALTER EXTENSION vector UPDATE;
ALTER EXTENSION vector UPDATE TO '0.8.7';

-- 移动 schema
ALTER EXTENSION vector SET SCHEMA extensions;

-- 卸载
DROP EXTENSION vector;
DROP EXTENSION IF EXISTS vector CASCADE;
```

### 18.3 常用扩展

| 扩展 | 作用 |
|---|---|
| `vector` | pgvector 向量类型与索引 |
| `pg_stat_statements` | SQL 执行统计 |
| `pgcrypto` | 加密、摘要、随机数 |
| `uuid-ossp` | UUID 生成（PG13+ 可用内置 `gen_random_uuid()`） |
| `citext` | 大小写不敏感文本 |
| `unaccent` | 去重音字符 |
| `pg_trgm` | 三元组相似度、模糊搜索 |
| `btree_gin` | 让 GIN 支持标量类型 |
| `btree_gist` | 让 GiST 支持标量类型，排他约束常用 |
| `hstore` | key-value 类型 |
| `postgres_fdw` | 访问其他 PostgreSQL |
| `dblink` | 跨库连接（较老） |
| `pg_prewarm` | 预热缓存 |
| `postgis` | 地理空间数据 |
| `timescaledb` | 时序数据（需单独安装） |
| `pg_repack` | 在线重组表（需单独安装） |
| `zhparser` / `pg_jieba` | 中文全文分词（需单独安装） |

### 18.4 pgcrypto

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

SELECT gen_random_uuid();
SELECT digest('abc', 'sha256');
SELECT encode(digest('abc', 'sha256'), 'hex');
SELECT crypt('password', gen_salt('bf'));
SELECT crypt('password', stored_hash) = stored_hash AS matched
FROM app.app_user
WHERE id = 1;
```

### 18.5 citext

```sql
CREATE EXTENSION IF NOT EXISTS citext;

CREATE TABLE app.account (
    id       bigserial PRIMARY KEY,
    email    citext UNIQUE NOT NULL
);

INSERT INTO app.account(email) VALUES ('Alice@Example.com');
SELECT * FROM app.account WHERE email = 'alice@example.com';   -- 能查到
```

### 18.6 pg_trgm 模糊搜索

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

SELECT similarity('postgresql', 'postgres');
SELECT word_similarity('postgres', 'postgresql database');

SELECT * FROM app.app_user
WHERE username % 'alice'
ORDER BY similarity(username, 'alice') DESC;

CREATE INDEX idx_app_user_username_trgm
    ON app.app_user USING GIN (username gin_trgm_ops);
```

### 18.7 btree_gist 与排他约束

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

ALTER TABLE app.booking
    ADD CONSTRAINT no_overlap
    EXCLUDE USING gist (
        room_id WITH =,
        tstzrange(start_at, end_at, '[)') WITH &&
    );
```

### 18.8 扩展安装失败排查

```sql
-- 看服务器有哪些可用扩展
SHOW shared_preload_libraries;

SELECT * FROM pg_available_extensions WHERE name = 'vector';
```

如果 `pg_available_extensions` 里没有，说明镜像里没装对应扩展文件。需要换镜像（如 `pgvector/pgvector:pg17`）或安装系统包。

`pg_stat_statements` 还需要在 `shared_preload_libraries` 中预加载并重启，单靠 `CREATE EXTENSION` 不够。

---

## 十九、pgvector 向量检索

### 19.1 确认环境

```sql
SELECT version();
SELECT extversion FROM pg_extension WHERE extname = 'vector';

\dx vector
```

当前环境应为 PostgreSQL 17 + pgvector 0.8.x。普通 `postgres:17` 镜像不带 pgvector，必须用 `pgvector/pgvector:pg17` 或自行安装扩展。

### 19.2 安装扩展

```sql
CREATE EXTENSION IF NOT EXISTS vector;

SELECT extname, extversion
FROM pg_extension
WHERE extname = 'vector';
```

扩展是**按数据库**安装的。换一个数据库要重新执行 `CREATE EXTENSION`。

### 19.3 向量类型与限制

| 类型 | 说明 | 每维存储 | 建索引最大维度 |
|---|---|---|---|
| `vector(n)` | 单精度浮点 | 4 字节 | 2000 |
| `halfvec(n)` | 半精度浮点 | 2 字节 | 4000 |
| `bit(n)` | 二值向量 | 1 bit | 64000 |
| `sparsevec(n)` | 稀疏向量 | 只存非零 | 1000 个非零元素 |

`vector` 列本身最多可存 16000 维，但 HNSW/IVFFlat 索引最多 2000 维。超过 2000 维时，要么降维，要么改用 `halfvec`。

### 19.4 建表

```sql
CREATE TABLE IF NOT EXISTS items (
    id         bigserial PRIMARY KEY,
    content    text,
    metadata   jsonb NOT NULL DEFAULT '{}'::jsonb,
    embedding  vector(1536)
);
```

Spring AI 默认表结构：

```sql
CREATE TABLE IF NOT EXISTS vector_store (
    id        uuid DEFAULT gen_random_uuid() PRIMARY KEY,
    content   text,
    metadata  json,
    embedding vector(1024)
);
```

### 19.5 写入向量

字符串字面量：

```sql
INSERT INTO items (content, embedding)
VALUES ('第一条', '[0.1, 0.2, 0.3]');
```

显式转换：

```sql
INSERT INTO items (content, embedding)
VALUES ('第二条', '[1,2,3]'::vector);
```

从数组转换：

```sql
INSERT INTO items (content, embedding)
VALUES ('第三条', ARRAY[1,2,3]::vector);
```

生成随机向量：

```sql
INSERT INTO items (content, embedding)
VALUES (
    '随机向量',
    (SELECT array_agg(random())::vector FROM generate_series(1, 1536))
);
```

批量生成：

```sql
INSERT INTO items (content, embedding)
SELECT '文档 ' || g,
       (SELECT array_agg(random())::vector FROM generate_series(1, 1536))
FROM generate_series(1, 500) AS g;
```

更新向量：

```sql
UPDATE items
SET embedding = '[0.5, 0.6, 0.7]'::vector
WHERE id = 1;
```

### 19.6 距离运算符

| 运算符 | 含义 | operator class |
|---|---|---|
| `<->` | 欧氏距离（L2） | `vector_l2_ops` |
| `<=>` | 余弦距离（1 - 余弦相似度） | `vector_cosine_ops` |
| `<#>` | 负内积 | `vector_ip_ops` |
| `<+>` | L1 / 曼哈顿距离 | `vector_l1_ops` |
| `<~>` | 汉明距离（bit） | `bit_hamming_ops` |
| `<%>` | Jaccard 距离（bit） | `bit_jaccard_ops` |

```sql
SELECT '[1,2,3]'::vector <-> '[4,5,6]'::vector AS l2;
SELECT '[1,2,3]'::vector <=> '[4,5,6]'::vector AS cosine_distance;
SELECT '[1,2,3]'::vector <#> '[4,5,6]'::vector AS negative_inner_product;
SELECT '[1,2,3]'::vector <+> '[4,5,6]'::vector AS l1;
```

余弦相似度要自己换算：

```sql
SELECT 1 - ('[1,2,3]'::vector <=> '[4,5,6]'::vector) AS cosine_similarity;
```

内积相似度：

```sql
SELECT -('[1,2,3]'::vector <#> '[4,5,6]'::vector) AS inner_product;
```

### 19.7 距离函数

```sql
SELECT l2_distance('[1,2,3]'::vector, '[4,5,6]'::vector);
SELECT cosine_distance('[1,2,3]'::vector, '[4,5,6]'::vector);
SELECT inner_product('[1,2,3]'::vector, '[4,5,6]'::vector);
SELECT l1_distance('[1,2,3]'::vector, '[4,5,6]'::vector);

SELECT vector_dims(embedding) FROM items LIMIT 5;
SELECT vector_norm(embedding) FROM items LIMIT 5;
SELECT subvector(embedding, 1, 3) FROM items LIMIT 5;
SELECT binary_quantize(embedding) FROM items LIMIT 5;
SELECT avg(embedding), sum(embedding) FROM items;
```

### 19.8 相似度查询

```sql
SELECT id,
       content,
       1 - (embedding <=> (SELECT embedding FROM items WHERE id = 1)) AS similarity
FROM items
WHERE id <> 1
ORDER BY embedding <=> (SELECT embedding FROM items WHERE id = 1)
LIMIT 5;
```

参数化查询（JDBC）：

```sql
SELECT id, content, embedding <=> ?::vector AS distance
FROM items
ORDER BY embedding <=> ?::vector
LIMIT ?;
```

核心规则：

- `ORDER BY` 里直接写向量运算符。
- 运算符要和索引的 operator class 一致。
- 不要写成 `ORDER BY distance`，也不要在外面再套一层排序。
- 查询向量维度必须和列定义一致。

### 19.9 HNSW 索引

```sql
CREATE INDEX IF NOT EXISTS items_embedding_hnsw
ON items USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

| 参数 | 默认 | 说明 |
|---|---|---|
| `m` | 16 | 每个节点的连接数，越大越准、越占内存 |
| `ef_construction` | 64 | 建索引候选集，越大越慢、图质量越好 |
| `hnsw.ef_search` | 40 | 查询候选集，越大越准、越慢 |

不同距离的索引：

```sql
-- 欧氏距离
CREATE INDEX items_embedding_l2
ON items USING hnsw (embedding vector_l2_ops);

-- 负内积
CREATE INDEX items_embedding_ip
ON items USING hnsw (embedding vector_ip_ops);
```

查询时调整：

```sql
SET hnsw.ef_search = 100;
SHOW hnsw.ef_search;
RESET hnsw.ef_search;

-- 只对当前事务
BEGIN;
SET LOCAL hnsw.ef_search = 200;
SELECT ...;
COMMIT;
```

### 19.10 IVFFlat 索引

```sql
CREATE INDEX items_embedding_ivfflat
ON items USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

| 参数 | 说明 |
|---|---|
| `lists` | 聚类中心数量，一般取 `行数 / 1000`（百万行以内） |
| `ivfflat.probes` | 查询时扫描的聚类数，默认 1，越大越准越慢 |

```sql
SET ivfflat.probes = 10;
SHOW ivfflat.probes;
```

IVFFlat 必须在表里**有数据之后**创建，空表建出来的聚类没有意义。

### 19.11 HNSW 与 IVFFlat 怎么选

| 场景 | 推荐 |
|---|---|
| 通用 RAG、追求召回率 | HNSW + `vector_cosine_ops` |
| 数据量大、建索引时间敏感 | IVFFlat，调 `lists` |
| 数据量少于几千条 | 不建索引，直接精确扫描 |
| 维度超过 2000 | 改用 `halfvec` 再建索引 |
| 内存充足、查询延迟敏感 | HNSW，适当提高 `ef_search` |
| 经常全量重建 | IVFFlat 建索引更快 |

HNSW 查询快、召回高，但索引大、建索引慢、占内存。IVFFlat 索引小、建得快，但召回依赖 `lists` 和 `probes`，数据分布变化后需要重建。

### 19.12 带过滤条件的向量查询

```sql
SELECT id, content
FROM items
WHERE metadata->>'source' = 'manual.pdf'
ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector
LIMIT 5;
```

这是“先向量检索、再过滤”还是“先过滤、再向量检索”，取决于计划。过滤条件很严格时，HNSW 可能扫描不到足够的候选，导致返回不足 5 条。解决办法：

1. 提高 `hnsw.ef_search`；
2. 建部分索引；
3. 使用 pgvector 0.8 的迭代扫描；
4. 把过滤列冗余进向量表，避免复杂 JOIN。

### 19.13 迭代扫描（pgvector 0.8+）

```sql
SET hnsw.iterative_scan = relaxed_order;
-- 可选值：off / relaxed_order / strict_order

SET hnsw.max_scan_tuples = 20000;
SET hnsw.scan_mem_multiplier = 1;
```

| 参数 | 说明 |
|---|---|
| `hnsw.iterative_scan = off` | 默认，关闭迭代扫描 |
| `relaxed_order` | 允许结果顺序略有放松，召回更好 |
| `strict_order` | 保持严格距离顺序，开销更大 |
| `hnsw.max_scan_tuples` | 最多扫描多少条候选 |
| `hnsw.scan_mem_multiplier` | 扫描内存倍数 |

过滤条件多、召回不足时优先试：

```sql
SET hnsw.iterative_scan = relaxed_order;
SET hnsw.ef_search = 100;
```

### 19.14 部分索引

```sql
CREATE INDEX items_manual_hnsw
ON items USING hnsw (embedding vector_cosine_ops)
WHERE metadata->>'source' = 'manual.pdf';
```

查询条件必须能匹配这个 `WHERE`，规划器才会用部分索引：

```sql
SELECT id, content
FROM items
WHERE metadata->>'source' = 'manual.pdf'
ORDER BY embedding <=> $1::vector
LIMIT 5;
```

### 19.15 建索引前的参数

```sql
SET maintenance_work_mem = '2GB';
SET max_parallel_maintenance_workers = 4;

CREATE INDEX items_embedding_hnsw
ON items USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

HNSW 建索引会占用大量内存。数据量大时，`maintenance_work_mem` 太小会导致磁盘临时文件，速度明显变慢。

查看建索引进度：

```sql
SELECT * FROM pg_stat_progress_create_index;
```

### 19.16 查询性能调优

```sql
-- 提高召回
SET hnsw.ef_search = 200;

-- IVFFlat 提高扫描聚类数
SET ivfflat.probes = 20;

-- 查看计划
EXPLAIN (ANALYZE, BUFFERS)
SELECT id
FROM items
ORDER BY embedding <=> (SELECT embedding FROM items LIMIT 1)
LIMIT 5;
```

期望看到：

```text
Index Scan using items_embedding_hnsw on items
```

如果看到 `Seq Scan`，常见原因：

- 运算符和索引 operator class 不匹配；
- 表太小，规划器认为全表扫描更便宜；
- `ORDER BY` 没有直接写向量运算符；
- 统计信息过期，执行 `ANALYZE items;`；
- 查询向量类型不对，发生了隐式转换。

### 19.17 修改向量维度

```sql
ALTER TABLE items
    ALTER COLUMN embedding TYPE vector(3072);
```

要求现有数据维度能转换到新类型，否则会报错。修改后必须重建索引：

```sql
DROP INDEX IF EXISTS items_embedding_hnsw;
CREATE INDEX items_embedding_hnsw
ON items USING hnsw (embedding vector_cosine_ops);
```

更稳妥的迁移方式：

```sql
ALTER TABLE items ADD COLUMN embedding_new vector(3072);
UPDATE items SET embedding_new = ...;
ALTER TABLE items DROP COLUMN embedding;
ALTER TABLE items RENAME COLUMN embedding_new TO embedding;
```

### 19.18 halfvec：省一半空间

```sql
CREATE TABLE items_half (
    id        bigserial PRIMARY KEY,
    content   text,
    embedding halfvec(3072)
);

CREATE INDEX items_half_hnsw
ON items_half USING hnsw (embedding halfvec_cosine_ops);

SELECT id
FROM items_half
ORDER BY embedding <=> $1::halfvec(3072)
LIMIT 5;
```

`halfvec` 精度略低，但存储和索引内存约为 `vector` 的一半，适合高维、内存敏感场景。

### 19.19 bit：二值量化

```sql
-- 先生成二值向量列
ALTER TABLE items ADD COLUMN embedding_bin bit(1536);

UPDATE items
SET embedding_bin = binary_quantize(embedding)::bit(1536);

CREATE INDEX items_embedding_bin_hnsw
ON items USING hnsw (embedding_bin bit_hamming_ops);

SELECT id
FROM items
ORDER BY embedding_bin <~> binary_quantize($1::vector)::bit(1536)
LIMIT 10;
```

二值量化速度快、空间小，但召回会下降，通常用于粗排，再用原始向量精排。

### 19.20 sparsevec：稀疏向量

```sql
CREATE TABLE items_sparse (
    id        bigserial PRIMARY KEY,
    content   text,
    embedding sparsevec(100000)
);

INSERT INTO items_sparse (content, embedding)
VALUES ('稀疏向量', '{1:0.5, 10:0.8, 100:0.2}/100000');

CREATE INDEX items_sparse_hnsw
ON items_sparse USING hnsw (embedding sparsevec_l2_ops);
```

格式 `{下标:值,...}/维度`，下标从 1 开始。索引最多支持 1000 个非零元素。

### 19.21 维护与诊断

```sql
-- 表大小
SELECT pg_size_pretty(pg_total_relation_size('items')) AS total,
       pg_size_pretty(pg_relation_size('items'))       AS table_only,
       pg_size_pretty(pg_indexes_size('items'))        AS indexes;

-- 索引列表
\di
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'items';

-- 索引使用情况
SELECT indexrelname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
WHERE relname = 'items';

-- 死元组
SELECT relname, n_live_tup, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'items';

-- 维护
VACUUM (ANALYZE) items;
REINDEX INDEX items_embedding_hnsw;
```

批量导入后执行 `VACUUM (ANALYZE) items;`，否则统计信息和可见性映射不准确，查询可能变慢。

### 19.22 删除向量数据

```sql
DELETE FROM items WHERE id = 1;
DELETE FROM items WHERE metadata->>'source' = 'old.pdf';
TRUNCATE TABLE items RESTART IDENTITY;
DROP TABLE items;
```

大量删除后执行 `VACUUM (ANALYZE) items;`，否则索引和表会保留死元组空间。

### 19.23 Spring AI 配置

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/vectordb
    username: postgres
    password: postgres
  sql:
    init:
      mode: always
  ai:
    vectorstore:
      pgvector:
        index-type: HNSW
        distance-type: COSINE_DISTANCE
        dimensions: 1024
        max-document-batch-size: 10000
        initialize-schema: false
        schema-validation: true
```

对应 `schema.sql`：

```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS hstore;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE IF NOT EXISTS vector_store (
    id        uuid DEFAULT uuid_generate_v4() PRIMARY KEY,
    content   text,
    metadata  json,
    embedding vector(1024)
);

CREATE INDEX IF NOT EXISTS spring_ai_vector_index
ON vector_store USING hnsw (embedding vector_cosine_ops);
```

要点：

- `dimensions` 必须和嵌入模型输出一致，例如 1024。
- `initialize-schema: false` 表示表由你自己建，Spring AI 只校验。
- `schema-validation: true` 会在启动时检查表结构，不匹配直接失败。
- 换嵌入模型时，向量维度、表结构、索引通常都要一起调整。

### 19.24 pgvector 常见报错

| 报错 | 原因 | 解决 |
|---|---|---|
| `type "vector" does not exist` | 当前库没装扩展 | `CREATE EXTENSION vector;` |
| `expected 1536 dimensions, not 1024` | 向量维度和列定义不一致 | 检查 embedding 模型和表定义 |
| `column cannot have more than 2000 dimensions for hnsw index` | 维度超限 | 改用 `halfvec` 或降维 |
| `operator does not exist: vector <-> text` | 字符串没转类型 | 加 `::vector` |
| `operator does not exist: vector <-> double precision[]` | 数组没转类型 | 加 `::vector` |
| `index row size ... exceeds maximum` | 向量太大 | 用 `halfvec`、降维或二值化 |
| `syntax error at or near "["` | 把 `[...]` 当数组字面量 | 加 `::vector` 或单引号 |
| 查询很慢、`Seq Scan` | 索引没生效 | 检查运算符、operator class、`ANALYZE` |
| 返回结果少于 `LIMIT` | 过滤条件导致候选不足 | 提高 `ef_search`，开迭代扫描，建部分索引 |
| 召回结果不准 | `lists` / `probes` / `ef_search` 太小 | 调大参数 |
| 建索引内存不足 | `maintenance_work_mem` 太小 | 调大后重建 |
| `role "postgres" does not exist` | 用户名不对 | 用实际 `POSTGRES_USER` |

### 19.25 pgvector 最佳实践

1. 向量列维度与嵌入模型严格一致，写入前做一次校验。
2. 先确认数据量：几千条以内可以不建索引，几十万条以上优先 HNSW。
3. `ORDER BY` 直接写距离运算符，不要包别名或子查询。
4. 过滤条件多的场景，提前设计部分索引或冗余过滤列。
5. 批量导入后执行 `VACUUM (ANALYZE)`。
6. 建 HNSW 前临时提高 `maintenance_work_mem`。
7. 定期监控索引大小、`idx_scan`、死元组和查询延迟。
8. 备份时把 `vector_store` 和业务表一起 `pg_dump`。

---

## 二十、常见报错与 SQLSTATE

### 20.1 SQLSTATE 速查

| SQLSTATE | 含义 | 常见处理 |
|---|---|---|
| `28P01` | 密码认证失败 | 检查密码、`pg_hba.conf` |
| `28000` | 认证配置拒绝 | 检查用户、来源地址 |
| `3D000` | 数据库不存在 | 建库或改连接串 |
| `42P01` | 表不存在 | 检查 schema、`search_path` |
| `42703` | 列不存在 | 检查列名和大小写 |
| `42501` | 权限不足 | `GRANT` 相应权限 |
| `23505` | 唯一约束冲突 | 用 `ON CONFLICT` 或先查询 |
| `23503` | 外键约束冲突 | 先插父表或调整顺序 |
| `23502` | 非空约束冲突 | 补值或改表结构 |
| `23514` | CHECK 约束失败 | 检查数据范围 |
| `40001` | 序列化失败 | 回滚后重试事务 |
| `40P01` | 死锁 | 回滚后重试，统一加锁顺序 |
| `57014` | 查询被取消 | 超时或被 `pg_cancel_backend` |
| `55P03` | 锁不可用 | 检查 `NOWAIT`/`lock_timeout` |
| `53300` | 连接数过多 | 调大连接数或查连接泄漏 |
| `53100` | 磁盘满 | 清理空间、归档 WAL |
| `53200` | 内存不足 | 降低 `work_mem`/并发 |
| `22001` | 字符串超长 | 扩大列或截断数据 |
| `22P02` | 输入语法错误 | 检查类型转换 |
| `22003` | 数值越界 | 扩大类型或校验 |
| `42P07` | 对象已存在 | 用 `IF NOT EXISTS` 或先删 |
| `42710` | 对象重复定义 | 检查重复约束/索引 |

### 20.2 连接类报错

**`connection refused`**

```powershell
docker ps
docker logs --tail 100 pgvector
Test-NetConnection localhost -Port 5432
```

检查：

- 容器是否运行；
- `-p 5432:5432` 是否映射；
- `listen_addresses` 是否为 `*`；
- 防火墙是否放行。

**`password authentication failed for user "postgres"`**

```sql
ALTER USER postgres WITH PASSWORD 'new_password';
```

检查 `pg_hba.conf` 的认证方式，确认没有写成错误的用户或数据库。

**`FATAL: no pg_hba.conf entry for host ...`**

```sql
SELECT * FROM pg_hba_file_rules;
```

在 `pg_hba.conf` 增加对应网段，然后：

```sql
SELECT pg_reload_conf();
```

**`database "xxx" does not exist`**

```sql
\l
CREATE DATABASE xxx;
```

**`role "xxx" does not exist`**

```sql
\du
CREATE ROLE xxx LOGIN PASSWORD '...';
```

**`too many connections for role "xxx"` / `too many clients already`**

```sql
SHOW max_connections;
SELECT usename, count(*) FROM pg_stat_activity GROUP BY usename;
```

处理：

- 检查应用连接池是否泄漏；
- 降低 `maximum-pool-size`；
- 适当调大 `max_connections`；
- 用 PgBouncer 做连接池。

**`remaining connection slots are reserved for non-replication superuser connections`**

普通用户连接数已满，只剩超级用户保留槽位。先清理空闲连接：

```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle'
  AND state_change < now() - interval '30 minutes';
```

### 20.3 SQL 类报错

**`relation "xxx" does not exist`**

```sql
\dt
\dt *.*
SHOW search_path;
SELECT current_schema();
```

常见原因是表在 `app` schema，但 `search_path` 没有包含它。写全名 `app.xxx` 或设置：

```sql
SET search_path TO app, public;
```

**`column "xxx" does not exist`**

PostgreSQL 会把未加引号的标识符转成小写。建表时用了 `"UserName"`，查询时必须写 `"UserName"`。

**`permission denied for table xxx`**

```sql
GRANT SELECT, INSERT, UPDATE, DELETE ON app.xxx TO appuser;
GRANT USAGE ON SCHEMA app TO appuser;
```

**`duplicate key value violates unique constraint`**

```sql
INSERT INTO app.app_user (username, email)
VALUES ('alice', 'alice@example.com')
ON CONFLICT (username)
DO UPDATE SET email = EXCLUDED.email;
```

**`insert or update on table ... violates foreign key constraint`**

先插入父表记录，或者调整事务内顺序。批量导入时可临时：

```sql
SET CONSTRAINTS ALL DEFERRED;
```

前提是外键声明了 `DEFERRABLE`。

**`null value in column ... violates not-null constraint`**

补默认值或显式传值：

```sql
ALTER TABLE app.orders ALTER COLUMN status SET DEFAULT 'pending';
```

**`value too long for type character varying(n)`**

扩大列：

```sql
ALTER TABLE app.app_user ALTER COLUMN email TYPE varchar(255);
```

**`invalid input syntax for type ...`**

检查数据格式，例如空字符串转数字：

```sql
SELECT NULLIF(trim(age), '')::integer;
```

**`deadlock detected`**

两个事务以不同顺序锁同一批行。统一加锁顺序，或缩小事务范围。应用层捕获 `40P01` 后重试。

**`could not serialize access due to concurrent update`**

`REPEATABLE READ` / `SERIALIZABLE` 下的正常并发冲突。回滚并重试整个事务。

**`canceling statement due to statement timeout`**

```sql
SHOW statement_timeout;
SET statement_timeout = '60s';
```

优化慢 SQL，或对特定接口放宽超时。

**`canceling statement due to lock timeout`**

```sql
SHOW lock_timeout;
SELECT * FROM pg_locks WHERE NOT granted;
```

找出阻塞会话，评估是否终止。

**`database is being accessed by other users`**

```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE datname = 'mydb'
  AND pid <> pg_backend_pid();

DROP DATABASE mydb;
```

### 20.4 运维类报错

**`could not extend file ... No space left on device`**

磁盘满。检查：

```powershell
docker system df
docker exec pgvector df -h
```

清理旧备份、归档 WAL、临时文件；必要时扩容。

**`out of memory` / `could not resize shared memory segment`**

降低 `work_mem`、`maintenance_work_mem`、并发数，或增加 Docker 内存限制。

**`invalid byte sequence for encoding "UTF8"`**

导入文件编码不是 UTF8。转换文件编码：

```powershell
Get-Content .\data.csv -Encoding Default |
    Set-Content .\data_utf8.csv -Encoding UTF8
```

**`prepared statement "S_1" already exists`**

常见于 PgBouncer 的 transaction 模式 + JDBC prepared statement。解决：

```text
prepareThreshold=0
```

或使用 session 模式连接池。

**`SSL SYSCALL error: EOF detected` / `connection reset`**

网络中断、容器重启、连接池空闲连接被防火墙断开。配置 TCP keepalive 和连接池探活。

### 20.5 pgvector 相关报错

见 [19.24 pgvector 常见报错](#十九pgvector-向量检索) 一节。最常见的三个：

```sql
-- 1. 扩展没装
CREATE EXTENSION IF NOT EXISTS vector;

-- 2. 维度不一致
SELECT vector_dims(embedding) FROM items LIMIT 1;

-- 3. 类型没转换
SELECT '[1,2,3]'::vector;
```

### 20.6 排障顺序

遇到问题时按这个顺序查：

1. 容器是否运行：`docker ps`
2. 端口是否可达：`Test-NetConnection`
3. 能否登录：`psql -U ... -d ...`
4. 数据库/表是否存在：`\l`、`\dt *.*`
5. 权限是否足够：`\dp`
6. SQL 是否报错：看 SQLSTATE
7. 是否锁等待：`pg_stat_activity`、`pg_locks`
8. 是否慢查询：`EXPLAIN ANALYZE`、`pg_stat_statements`
9. 是否磁盘/内存问题：`df -h`、`docker stats`

---

## 二十一、一页速查

### 21.1 Docker

```powershell
docker ps
docker ps -a
docker logs -f pgvector
docker restart pgvector
docker stop pgvector
docker start pgvector
docker exec -it pgvector bash
docker exec -it pgvector psql -U postgres -d vectordb
docker exec pgvector pg_isready -U postgres -d vectordb
docker stats --no-stream pgvector
docker system df
```

### 21.2 psql

```text
\l            数据库列表
\c db         切换数据库
\dt           表
\d table      表结构
\di           索引
\dn           schema
\du           角色
\dx           扩展
\df           函数
\dp           权限
\x            纵向显示
\timing       显示耗时
\i file.sql   执行文件
\copy         导入导出 CSV
\watch 5      每 5 秒刷新
\?            帮助
\q            退出
```

### 21.3 数据库与 schema

```sql
CREATE DATABASE mydb;
DROP DATABASE mydb;
ALTER DATABASE mydb RENAME TO mydb2;

CREATE SCHEMA app;
ALTER SCHEMA app RENAME TO business;
DROP SCHEMA business CASCADE;

SHOW search_path;
SET search_path TO app, public;
```

### 21.4 用户与权限

```sql
CREATE ROLE appuser LOGIN PASSWORD 'pass';
GRANT CONNECT ON DATABASE vectordb TO appuser;
GRANT USAGE, CREATE ON SCHEMA app TO appuser;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app TO appuser;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA app TO appuser;

ALTER DEFAULT PRIVILEGES IN SCHEMA app
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO appuser;

ALTER USER appuser WITH PASSWORD 'newpass';
DROP ROLE appuser;
```

### 21.5 表

```sql
CREATE TABLE app.app_user (
    id bigint GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    username varchar(64) NOT NULL UNIQUE,
    created_at timestamptz NOT NULL DEFAULT now()
);

ALTER TABLE app.app_user ADD COLUMN email varchar(128);
ALTER TABLE app.app_user ALTER COLUMN email TYPE varchar(255);
ALTER TABLE app.app_user RENAME COLUMN email TO mail;
ALTER TABLE app.app_user DROP COLUMN mail;

TRUNCATE TABLE app.app_user RESTART IDENTITY CASCADE;
DROP TABLE app.app_user;
```

### 21.6 查询

```sql
SELECT * FROM app.app_user WHERE status = 1 ORDER BY id DESC LIMIT 20;
SELECT status, count(*) FROM app.orders GROUP BY status HAVING count(*) > 10;

SELECT u.username, o.amount
FROM app.app_user u
JOIN app.orders o ON o.user_id = u.id;

SELECT row_number() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
FROM app.orders;

WITH t AS (SELECT * FROM app.orders WHERE status = 'paid')
SELECT user_id, sum(amount) FROM t GROUP BY user_id;

EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
```

### 21.7 事务与锁

```sql
BEGIN;
...
COMMIT;

ROLLBACK;
SAVEPOINT sp1;
ROLLBACK TO SAVEPOINT sp1;

SELECT * FROM app.jobs
WHERE status = 'pending'
ORDER BY created_at
LIMIT 10
FOR UPDATE SKIP LOCKED;

SELECT pid, pg_blocking_pids(pid), query FROM pg_stat_activity;
SELECT pg_cancel_backend(pid);
SELECT pg_terminate_backend(pid);
```

### 21.8 运维

```sql
SELECT version();
SELECT * FROM pg_stat_activity WHERE state <> 'idle';
SELECT * FROM pg_locks WHERE NOT granted;

SELECT pg_size_pretty(pg_total_relation_size('app.orders'));
SELECT * FROM pg_stat_user_tables ORDER BY n_dead_tup DESC;

VACUUM (ANALYZE) app.orders;
ANALYZE app.orders;
REINDEX INDEX app.idx_orders_status;
CLUSTER app.orders USING idx_orders_created_at;

SHOW max_connections;
SHOW shared_buffers;
SHOW work_mem;
ALTER SYSTEM SET work_mem = '16MB';
SELECT pg_reload_conf();
```

### 21.9 备份恢复

```powershell
docker exec -t pgvector pg_dump -U postgres -d vectordb -Fc -f /tmp/vectordb.dump
docker cp pgvector:/tmp/vectordb.dump C:\backup\vectordb.dump

docker cp C:\backup\vectordb.dump pgvector:/tmp/vectordb.dump
docker exec -t pgvector pg_restore -U postgres -d vectordb --clean --if-exists /tmp/vectordb.dump
```

### 21.10 pgvector

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE items (
    id bigserial PRIMARY KEY,
    content text,
    metadata jsonb DEFAULT '{}'::jsonb,
    embedding vector(1536)
);

INSERT INTO items (content, embedding)
VALUES ('文本', (SELECT array_agg(random())::vector FROM generate_series(1,1536)));

SELECT id, content, 1 - (embedding <=> $1::vector) AS similarity
FROM items
ORDER BY embedding <=> $1::vector
LIMIT 5;

CREATE INDEX ON items USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

SET hnsw.ef_search = 100;
SET hnsw.iterative_scan = relaxed_order;

EXPLAIN ANALYZE
SELECT id FROM items ORDER BY embedding <=> $1::vector LIMIT 5;
```

### 21.11 pgvector 运算符与索引

```text
运算符   含义                 operator class
<->      欧氏距离             vector_l2_ops
<=>      余弦距离             vector_cosine_ops
<#>      负内积               vector_ip_ops
<+>      L1 距离              vector_l1_ops
<~>      汉明距离(bit)        bit_hamming_ops
<%>      Jaccard 距离(bit)    bit_jaccard_ops

索引维度上限
vector 2000 / halfvec 4000 / bit 64000 / sparsevec 1000 非零

默认参数
hnsw.m = 16
hnsw.ef_construction = 64
hnsw.ef_search = 40
ivfflat.probes = 1
```

### 21.12 常用日期与 JSON

```sql
SELECT now(), current_date, current_timestamp;
SELECT date_trunc('day', now());
SELECT extract(year FROM now());
SELECT to_char(now(), 'YYYY-MM-DD HH24:MI:SS');
SELECT now() - interval '7 days';

SELECT profile ->> 'city' FROM app.app_user;
SELECT * FROM app.app_user WHERE profile @> '{"level":3}';
SELECT jsonb_set(profile, '{city}', '"Shanghai"', true) FROM app.app_user;
```

### 21.13 最后检查清单

```text
[ ] 容器在运行，端口映射正确
[ ] 能进 psql，能 SELECT version()
[ ] 数据库、schema、search_path 正确
[ ] 用户有 CONNECT / USAGE / 表权限
[ ] 表结构、主键、索引已创建
[ ] 慢 SQL 有 EXPLAIN 和索引
[ ] 连接池大小合理，无连接泄漏
[ ] 有可用备份，且做过恢复验证
[ ] pgvector 扩展版本、维度、索引一致
```

