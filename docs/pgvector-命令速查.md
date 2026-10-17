# pgvector 命令速查手册

> 配套文档：`Windows-Docker-pgvector-安装指南.md`
> 适用：pgvector 0.7 / 0.8，PostgreSQL 16–18，容器方式部署

---

## 目录

1. [进入容器](#一进入容器)
2. [psql 元命令](#二psql-元命令)
3. [数据类型](#三数据类型)
4. [扩展管理](#四扩展管理)
5. [建表与写入](#五建表与写入)
6. [距离运算符与函数](#六距离运算符与函数)
7. [索引](#七索引)
8. [查询写法](#八查询写法)
9. [运行时调参](#九运行时调参)
10. [诊断与维护](#十诊断与维护)
11. [备份恢复与容器运维](#十一备份恢复与容器运维)
12. [常见报错对照](#十二常见报错对照)
13. [一页速查](#十三一页速查)

---

## 一、进入容器

先确认容器在跑：

```powershell
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

### 1. 直接进 psql（最常用）

```powershell
docker exec -it pgvector psql -U postgres -d vectordb
```

| 参数 | 含义 |
|---|---|
| `-it` | 交互 + 伪终端，少了就进不去交互界面 |
| `-U` | 数据库用户，由 `POSTGRES_USER` 决定 |
| `-d` | 数据库名，由 `POSTGRES_DB` 决定 |

退出：`\q`

### 2. 进 shell

```powershell
docker exec -it pgvector bash
```

### 3. 不进容器，单条执行

```powershell
docker exec pgvector psql -U postgres -d vectordb -c "SELECT version();"
```

### 4. 执行 SQL 文件

```powershell
Get-Content .\test.sql -Raw | docker exec -i pgvector psql -U postgres -d vectordb
```

### 5. 用图形客户端

| 项 | 值 |
|---|---|
| Host | `localhost` |
| Port | 你映射的主机端口（如 `15432`） |
| Database | `vectordb` |
| User / Password | 你的 `POSTGRES_USER` / `POSTGRES_PASSWORD` |

---

## 二、psql 元命令

### 查看类

| 命令 | 作用 |
|---|---|
| `\l` | 列出所有数据库 |
| `\c 库名` | 切换数据库 |
| `\dt` | 列出当前库的表 |
| `\dt+` | 表格外带大小和描述 |
| `\d 表名` | 表结构：列、类型、索引、约束 |
| `\d+ 表名` | 更详细 |
| `\di` | 列出索引 |
| `\di+ 索引名` | 索引大小和定义 |
| `\dv` | 视图 |
| `\dn` | schema |
| `\du` | 角色 / 用户 |
| `\df` | 函数 |
| `\dx` | 已安装扩展（查 vector 是否装上） |
| `\dx+ vector` | vector 扩展包含的对象 |
| `\dp` | 权限 |

### 操作类

| 命令 | 作用 |
|---|---|
| `\x` | 切换纵向显示（宽表友好） |
| `\timing` | 开关耗时统计 |
| `\e` | 用编辑器写多行 SQL |
| `\i 文件.sql` | 执行 SQL 文件 |
| `\o 文件.txt` | 把后续结果输出到文件 |
| `\q` | 退出 |

### 帮助

| 命令 | 作用 |
|---|---|
| `\?` | 元命令帮助 |
| `\h` | SQL 语法帮助列表 |
| `\h CREATE INDEX` | 具体语法帮助 |

---

## 三、数据类型

| 类型 | 说明 | 存储 | 索引维度上限 |
|---|---|---|---|
| `vector(n)` | 常规浮点向量（float4） | 4 字节/维 | 2000 |
| `halfvec(n)` | 半精度（float2），省一半空间 | 2 字节/维 | 4000 |
| `bit(n)` | 二值向量，做布尔特征 | 1 bit/维 | 64000 |
| `sparsevec(n)` | 稀疏向量（0.7+） | 只存非零 | 1000 个非零元素 |

`vector` 本身最多可存 16000 维，但建索引最多 2000 维。超过就用 `halfvec`。

---

## 四、扩展管理

```sql
CREATE EXTENSION IF NOT EXISTS vector;

SELECT extversion FROM pg_extension WHERE extname = 'vector';

SELECT * FROM pg_available_extensions WHERE name = 'vector';

ALTER EXTENSION vector UPDATE;

DROP EXTENSION vector;
```

---

## 五、建表与写入

### 建表

```sql
CREATE TABLE IF NOT EXISTS items (
    id         bigserial PRIMARY KEY,
    content    text,
    metadata   jsonb DEFAULT '{}'::jsonb,
    embedding  vector(1536)
);
```

### 插入

```sql
INSERT INTO items (content, embedding) VALUES ('第一条', '[0.1, 0.2, 0.3]');

INSERT INTO items (content, embedding) VALUES ('第二条', '[1,2,3]'::vector);

INSERT INTO items (content, embedding)
VALUES ('测试文档', (SELECT array_agg(random())::vector FROM generate_series(1, 1536)));

INSERT INTO items (content, embedding)
SELECT '文档 ' || g, (SELECT array_agg(random())::vector FROM generate_series(1, 1536))
FROM generate_series(1, 500) g;
```

### 改类型 / 改维度

```sql
ALTER TABLE items ALTER COLUMN embedding TYPE vector(3072);
ALTER TABLE items ALTER COLUMN embedding TYPE halfvec(3072);
```

改完必须重建索引。

---

## 六、距离运算符与函数

### 运算符

| 运算符 | 含义 | 配套 operator class |
|---|---|---|
| `<->` | 欧氏距离（L2） | `vector_l2_ops` |
| `<=>` | 余弦距离（1 - 余弦相似度） | `vector_cosine_ops` |
| `<#>` | 负内积（值为负，取反才是相似度） | `vector_ip_ops` |
| `<+>` | L1 / 曼哈顿距离（0.7+） | `vector_l1_ops` |
| `<~>` | 汉明距离（bit 类型） | `bit_hamming_ops` |
| `<%>` | Jaccard 距离（bit 类型） | `bit_jaccard_ops` |

```sql
SELECT '[1,2,3]'::vector <-> '[4,5,6]'::vector;
SELECT '[1,2,3]'::vector <=> '[4,5,6]'::vector;
SELECT '[1,2,3]'::vector <#> '[4,5,6]'::vector;
```

### 函数

```sql
SELECT l2_distance('[1,2,3]'::vector, '[4,5,6]'::vector);
SELECT inner_product('[1,2,3]'::vector, '[4,5,6]'::vector);
SELECT cosine_distance('[1,2,3]'::vector, '[4,5,6]'::vector);
SELECT l1_distance('[1,2,3]'::vector, '[4,5,6]'::vector);

SELECT vector_dims(embedding) FROM items LIMIT 5;
SELECT vector_norm(embedding) FROM items LIMIT 5;
SELECT subvector(embedding, 1, 3) FROM items LIMIT 5;
SELECT binary_quantize(embedding) FROM items LIMIT 5;

SELECT avg(embedding), sum(embedding) FROM items;
```

---

## 七、索引

### HNSW（首选）

```sql
CREATE INDEX IF NOT EXISTS items_emb_hnsw
ON items USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);
```

| 参数 | 默认 | 说明 |
|---|---|---|
| `m` | 16 | 每层连接数，越大越准越占内存 |
| `ef_construction` | 64 | 建索引时的候选数，越大越慢、图质量越好 |
| `hnsw.ef_search` | 40 | 查询时用，越大越准越慢 |

### IVFFlat

```sql
CREATE INDEX IF NOT EXISTS items_emb_ivfflat
ON items USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

| 参数 | 说明 |
|---|---|
| `lists` | 一般取「行数 / 1000」（百万行以内） |
| `ivfflat.probes` | 查询时用，默认 1，越大越准越慢 |

IVFFlat 必须在表里有数据之后建，空表建出来的聚类是错的。

### 其它

```sql
DROP INDEX IF EXISTS items_emb_hnsw;
REINDEX INDEX items_emb_hnsw;
CREATE INDEX CONCURRENTLY IF NOT EXISTS ... ;
```

### 选型

| 场景 | 选择 |
|---|---|
| 通用、追求召回率 | HNSW + `vector_cosine_ops` |
| 建索引时间敏感、数据量大 | IVFFlat，调 `lists` |
| 数据量少于几千条 | 不建索引 |
| 维度超过 2000 | 改用 `halfvec` 再建索引 |

---

## 八、查询写法

### 基本 Top-K

```sql
SELECT id, content,
       1 - (embedding <=> (SELECT embedding FROM items WHERE id = 1)) AS similarity
FROM items
ORDER BY embedding <=> (SELECT embedding FROM items WHERE id = 1)
LIMIT 5;
```

核心规则：`ORDER BY` 里必须直接写向量运算符，且要和索引的 operator class 一致。写成 `ORDER BY similarity` 或套一层子查询，索引就用不上。

### 带元数据过滤

```sql
SELECT id, content
FROM items
WHERE metadata->>'source' = 'manual.pdf'
ORDER BY embedding <=> '[0.1, 0.2, ...]'
LIMIT 5;
```

过滤条件很严时，HNSW 可能召回不足，开迭代扫描：

```sql
SET hnsw.iterative_scan = relaxed_order;
```

### 只看距离

```sql
SELECT id, embedding <-> '[1,2,3]' AS dist
FROM items
ORDER BY embedding <-> '[1,2,3]'
LIMIT 10;
```

---

## 九、运行时调参

```sql
SET hnsw.ef_search = 100;
SET ivfflat.probes = 10;
SET hnsw.iterative_scan = relaxed_order;
SHOW hnsw.ef_search;
RESET hnsw.ef_search;
```

`SET` 只对当前会话有效，要持久化：

```sql
ALTER DATABASE vectordb SET hnsw.ef_search = 100;
ALTER ROLE postgres SET hnsw.ef_search = 100;
```

---

## 十、诊断与维护

```sql
EXPLAIN ANALYZE
SELECT id FROM items
ORDER BY embedding <=> (SELECT embedding FROM items LIMIT 1)
LIMIT 5;

SELECT pg_size_pretty(pg_total_relation_size('items')) AS total,
       pg_size_pretty(pg_relation_size('items'))       AS table_only,
       pg_size_pretty(pg_indexes_size('items'))        AS indexes;

SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'items';

SELECT indexrelname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes WHERE relname = 'items';

SELECT relname, n_live_tup, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables WHERE relname = 'items';

SHOW hnsw.ef_search;
SELECT * FROM pg_settings WHERE name LIKE 'hnsw.%';

VACUUM ANALYZE items;
REINDEX INDEX items_emb_hnsw;
```

执行计划里应该出现 `Index Scan using items_emb_hnsw`。看到 `Seq Scan` 说明索引没生效，通常是运算符和 operator class 不匹配，或者统计信息过期。

---

## 十一、备份恢复与容器运维

```powershell
docker exec pgvector pg_dump -U postgres -d vectordb > backup.sql

docker exec pgvector pg_dump -U postgres -d vectordb --schema-only > schema.sql

docker exec pgvector pg_dump -U postgres -d vectordb --data-only > data.sql

Get-Content .\backup.sql -Raw | docker exec -i pgvector psql -U postgres -d vectordb

docker exec pgvector pg_isready -U postgres -d vectordb

docker stats --no-stream pgvector

docker compose restart pgvector
docker compose down
docker compose down -v
```

---

## 十二、常见报错对照

| 报错 | 原因 | 解决 |
|---|---|---|
| `type "vector" does not exist` | 当前库没装扩展 | `CREATE EXTENSION vector;` |
| `expected 1536 dimensions, not 1024` | 向量维度和列定义不一致 | 检查 embedding 模型输出维度 |
| `column cannot have more than 2000 dimensions for hnsw index` | 维度超限 | 改用 `halfvec`，或降维 |
| `operator does not exist: vector <-> text` | 字符串没转类型 | 加 `::vector` |
| `operator does not exist: vector <-> double precision[]` | 传了数组没转 | 加 `::vector` |
| `index row size ... exceeds maximum` | 维度太大 | 换 `halfvec` 或降维 |
| `syntax error at or near "["` | 直接把 `[...]` 当数组字面量 | 加 `::vector` |
| 查询很慢、`Seq Scan` | 索引没生效 | 检查 `ORDER BY` 是否用了匹配的运算符 |
| 召回结果不对 | `lists` / `ef_search` 太小 | 调大参数，或开 `iterative_scan` |
| `role "postgres" does not exist` | `POSTGRES_USER` 不是 postgres | 用实际用户名 |

---

## 十三、一页速查

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE items (
    id bigserial PRIMARY KEY,
    content text,
    metadata jsonb DEFAULT '{}',
    embedding vector(1536)
);

INSERT INTO items (content, embedding)
VALUES ('文本', (SELECT array_agg(random())::vector FROM generate_series(1,1536)));

SELECT id, content, 1 - (embedding <=> $1) AS similarity
FROM items ORDER BY embedding <=> $1 LIMIT 5;

CREATE INDEX ON items USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

SET hnsw.ef_search = 100;

EXPLAIN ANALYZE SELECT id FROM items ORDER BY embedding <=> $1 LIMIT 5;
```

```text
运算符   含义               operator class
<->      欧氏距离           vector_l2_ops
<=>      余弦距离           vector_cosine_ops
<#>      负内积             vector_ip_ops
<+>      L1 距离            vector_l1_ops
<~>      汉明距离(bit)      bit_hamming_ops

索引维度上限
vector 2000 / halfvec 4000 / bit 64000 / sparsevec 1000 非零

默认参数
hnsw.m=16  hnsw.ef_construction=64  hnsw.ef_search=40
ivfflat.probes=1
```

```text
psql 常用
\l 库列表   \c 库 切换   \dt 表     \d 表 结构
\di 索引    \dx 扩展     \df 函数   \x 纵向显示
\timing     \i 文件      \? 帮助    \q 退出
```
