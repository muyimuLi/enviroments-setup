# Windows Docker 部署 PostgreSQL / pgvector 与 Spring Boot 对接完整指南

> 适用环境：Windows 10 Pro 22H2（19045+）/ Windows 11 + WSL2 + Docker Desktop  
> 技术栈：PostgreSQL 17、pgvector 0.8.x、Spring Boot 3.x/4.x、Spring AI  
> 合并来源：`Windows-Docker-pgvector-安装指南.md` + `PostgreSQL-Docker-安装与SpringBoot对接指南.md`  
> 配套速查：`PostgreSQL&pgvector命令速查手册.md`（全量 SQL、运维、备份命令）  
> 整理日期：2026-10-18

本文把 Windows 环境准备、Docker Desktop、WSL2、PostgreSQL/pgvector 部署、Spring Boot 数据源配置、JPA 实体、Spring AI 向量库接入和常见故障排查整合成一条完整链路。按顺序阅读可以从前置检查一直做到应用启动成功。

---

## 目录

1. 先看结论：到底要不要装第二个 PostgreSQL
2. Windows 环境准备
3. WSL2 安装与配置
4. Docker Desktop 安装与网络配置
5. 部署 PostgreSQL / pgvector
6. pgvector 初始化与验证
7. Spring Boot 对接 PostgreSQL
8. Spring AI 对接 pgvector
9. 运维、备份与迁移
10. 故障排查
11. 最小验收清单
12. 附录：端口、目录与一条龙命令

---

## 一、先看结论：到底要不要装第二个 PostgreSQL

### 1.1 pgvector 镜像就是 PostgreSQL

`pgvector/pgvector:pg17` 不是另一种数据库，它是**预装了 vector 扩展的 PostgreSQL 17**。所以：

| 你的目标 | 推荐做法 |
|---|---|
| 只要一个普通 PostgreSQL | 直接用现有 `pgvector` 容器，不必再装 |
| 要存业务表，也要做向量检索 | 同一个实例，业务表放 `app` schema，向量表放 `public` |
| 想完全隔离业务库和向量库 | 再起一个容器，端口用 `5433` 或 `15432` |
| 业务库必须是 MySQL | 多数据源，pgvector 继续留在主数据源最省事 |
| 只是想学习 PostgreSQL | 用现有容器建库建表即可 |

只有以下情况才值得再启动一个 PostgreSQL 容器：

- 需要测试 PostgreSQL 大版本升级；
- 业务库和向量库要有独立的备份、恢复、迁移周期；
- 两个库需要不同的内存参数、扩展或资源限制；
- 想模拟两个独立数据库服务器的部署结构；
- 担心误操作影响向量数据。

### 1.2 本机端口规划

| 用途 | 端口 | 说明 |
|---|---|---|
| 现有 pgvector 容器 | `5432` | 本机当前实际映射端口 |
| 新独立 PostgreSQL | `5433` | 与现有容器并存时使用 |
| 5432 被 Windows 保留时 | `15432` | 常见规避端口 |
| MySQL | `3306` | 与 PostgreSQL 不冲突 |

先确认真实端口，不要凭文档猜：

```powershell
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Ports}}\t{{.Status}}"
docker port pgvector
```

Spring Boot 的 JDBC URL 必须和这里看到的**宿主机端口**一致。

### 1.3 版本基线

本文以以下版本为例：

```text
PostgreSQL 17.x
pgvector 0.8.x
Docker Desktop 4.x
Spring Boot 3.x / 4.x
Spring AI 1.x / 2.x
```

PostgreSQL 18 的官方镜像调整了数据目录，详见 5.6 节。pgvector 的版本用 `SELECT extversion FROM pg_extension WHERE extname='vector';` 查看。

---

## 二、Windows 环境准备

### 2.1 系统要求

Docker Desktop 要求：

- Windows 10 22H2，内部版本 **19045** 及以上，或 Windows 11；
- BIOS/UEFI 已开启虚拟化；
- 建议内存 8GB 以上，磁盘预留 20GB 以上；
- 使用 WSL2 后端，而不是旧版 Hyper-V 后端。

查看系统版本：

```powershell
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion' |
  Select-Object ProductName, DisplayVersion, CurrentBuild
```

### 2.2 检查虚拟化

开机后在**任务管理器 → 性能 → CPU**，右下角应显示「虚拟化：已启用」。

如果没有启用，重启进 BIOS/UEFI，把：

- Intel：`Intel Virtualization Technology` / `VT-x`
- AMD：`AMD-V` / `SVM Mode`

设为 Enabled。

### 2.3 检查端口占用

```powershell
Get-NetTCPConnection -State Listen -LocalPort 5432 -ErrorAction SilentlyContinue
Get-NetTCPConnection -State Listen -LocalPort 5433 -ErrorAction SilentlyContinue
```

没有输出说明端口空闲。还要检查 Windows 保留端口段：

```powershell
netsh int ipv4 show excludedportrange protocol=tcp
netsh int ipv4 show dynamicport tcp
```

如果 5432 落在保留段里，Docker 会报 `access permissions`，不是普通的端口占用。处理方式见 10.4 节。

### 2.4 项目代码放在哪里

**放在 WSL 的 Linux 文件系统里，不要放在 `/mnt/c/...`。**

```bash
mkdir -p ~/projects
cd ~/projects
```

跨文件系统读写大量小文件时性能差很多，Maven、npm、Git 操作会明显变慢。想在资源管理器里访问：

```text
\\wsl$\Ubuntu\home\<用户名>\projects
```

---

## 三、WSL2 安装与配置

### 3.1 安装 WSL2

以**管理员身份**打开 PowerShell：

```powershell
wsl --install
```

这条命令会启用「虚拟机平台」、启用「适用于 Linux 的 Windows 子系统」并安装 Ubuntu。**执行完必须重启电脑。**

重启后验证：

```powershell
wsl --version
wsl -l -v
wsl -d Ubuntu -- uname -a
```

`VERSION` 列必须是 `2`。

### 3.2 卡在「正在安装 Ubuntu」或提示连接重置

前两步本地功能已启用，但发行版下载失败，这是国内网络常见情况。按顺序尝试：

**第一步：重启后再执行一次**

```powershell
wsl --install -d Ubuntu
```

进度条长期停在 `0.0%` 是正常的，它不刷新。判断是否真的解包成功，看 `ext4.vhdx` 大小：

```powershell
Get-ChildItem "$env:LOCALAPPDATA\wsl" -Recurse -File |
  Select-Object FullName, @{n='MB';e={[math]::Round($_.Length/1MB,1)}}
```

超过 1GB 通常说明 rootfs 已经解包完成。

**第二步：换下载通道**

```powershell
wsl --install -d Ubuntu --web-download
```

或者打开 Microsoft Store 搜索 `Ubuntu 22.04 LTS` 手动安装，它走另一套 CDN。

**第三步：离线导入**

从清华 TUNA、中科大或阿里云镜像站下载 WSL rootfs 压缩包，然后：

```powershell
New-Item -ItemType Directory D:\wsl\ubuntu -Force
wsl --import Ubuntu D:\wsl\ubuntu "D:\Downloads\ubuntu-jammy-wsl-amd64-wsl.rootfs.tar.gz" --version 2
wsl --set-default Ubuntu
```

`--import` 方式默认直接进 root，不会引导创建用户；需要的话自己执行 `adduser`。

### 3.3 配置 .wslconfig

新建**文件** `C:\Users\<用户名>\.wslconfig`：

```ini
[wsl2]
memory=8GB
processors=4
swap=2GB
systemd=true
```

说明：

- `memory` 按物理内存的一半左右设置；
- `systemd=true` 方便在 WSL 里运行 Linux 服务；
- 修改后执行 `wsl --shutdown` 重新加载。

> **最容易踩的坑**：记事本保存会自动补 `.txt`，文件名变成 `.wslconfig.txt`，WSL 不认。保存时「保存类型」选**所有文件**，或者文件名两边加英文引号。

验证它确实是文件而不是文件夹：

```powershell
Get-Item "$env:USERPROFILE\.wslconfig" | Select-Object FullName, Attributes
```

`Attributes` 里不能有 `Directory`。

### 3.4 WSL 常用命令

```powershell
wsl -l -v
wsl -d Ubuntu
wsl --shutdown
wsl --update
wsl --set-default-version 2
```

`wsl --update` 报 403 `Wsl/UpdatePackage/HTTP_E_STATUS_FORBIDDEN` 通常可以忽略，它只是检查更新，不影响已经装好的 WSL。根因是 `wsl.exe` 走 WinHTTP 通道，而代理常只配在 WinINET 层。

---

## 四、Docker Desktop 安装与网络配置

### 4.1 安装

```powershell
winget install Docker.DockerDesktop
```

装完**新开一个终端**，让 PATH 刷新：

```powershell
docker version
docker compose version
docker run hello-world
```

客户端和服务端版本都能打印出来才算通。

如果 `docker` 命令找不到，CLI 完整路径是：

```text
C:\Program Files\Docker\Docker\resources\bin\docker.exe
```

### 4.2 Docker Desktop 关键设置

打开 Docker Desktop → Settings：

| 位置 | 建议 |
|---|---|
| `General` | 按需开启开机自启 |
| `Resources → WSL Integration` | 打开 Ubuntu 发行版开关 |
| `Resources → Proxies` | 需要代理时手动配置 |
| `Docker Engine` | 可选配镜像加速器 |
| `Resources → Advanced` | 按机器内存调整 CPU/内存 |

WSL Integration 打开后，在 Ubuntu 终端里也能直接使用 `docker`，不需要 `sudo`。

### 4.3 镜像拉取失败

典型报错：

```text
docker: Error response from daemon: pull access denied for hello-world,
repository does not exist or may require 'docker login'
```

`hello-world` 是公开镜像，这个报错通常不是“镜像不存在”，而是请求被拒或匿名拉取被限流。

先判断网络层：

```powershell
curl.exe -v https://registry-1.docker.io/v2/
```

- 返回 `401`，并带有 `www-authenticate: Bearer realm="https://auth.docker.io/token"`：网络是通的，执行 `docker login` 后重试；
- 返回 `403`、超时或连接重置：代理或网络没通，继续下面的代理配置。

```powershell
docker login
docker pull hello-world
```

免费账号即可，登录后的拉取配额宽松得多。

### 4.4 代理配置

先看系统代理：

```powershell
netsh winhttp show proxy
Get-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings' |
  Select-Object ProxyEnable, ProxyServer
```

常见问题是代理只配在 WinINET（浏览器）层，而 `wsl.exe` 和系统下载组件走 WinHTTP：

```powershell
netsh winhttp set proxy proxy-server="127.0.0.1:7897" bypass-list="<local>"
```

还原：

```powershell
netsh winhttp reset proxy
```

给 Docker 守护进程单独配代理：`Settings → Resources → Proxies → Manual proxy configuration`，HTTP/HTTPS 都填。

> **WSL2 关键提醒**：Docker 守护进程跑在 WSL2 的 `docker-desktop` 发行版里，它里面的 `127.0.0.1` 不是 Windows 的 `127.0.0.1`。需要在代理软件里打开 **Allow LAN**，然后在 `ipconfig` 里找到 Windows 的局域网 IPv4（例如 `192.168.1.100`），填 `http://192.168.1.100:7897`，不要填 `127.0.0.1`。

### 4.5 镜像加速器

`Settings → Docker Engine` 中可以加：

```json
"registry-mirrors": ["https://你的账号专属加速地址"]
```

不要照抄网上教程里的公共地址，很多已经关停。优先使用阿里云、腾讯云、华为云容器镜像服务控制台里的账号专属地址。

### 4.6 验证整条链路

```powershell
docker run hello-world
docker run -d --name web -p 8080:80 nginx
```

浏览器打开 `http://localhost:8080`，能看到 nginx 欢迎页，说明 WSL2 → Docker 守护进程 → Windows 端口转发全通了。测试完可以删除：

```powershell
docker rm -f web
```

---

## 五、部署 PostgreSQL / pgvector

### 5.1 镜像怎么选

| 镜像 | 是什么 | 是否带 pgvector |
|---|---|---|
| `postgres:17` | 官方纯净 PostgreSQL | 否，需要自己安装扩展 |
| `pgvector/pgvector:pg17` | PostgreSQL 17 + 预装 vector 扩展 | 是，开箱即用 |
| `pgvector/pgvector:pg18` | PostgreSQL 18 + pgvector | 是 |

验证镜像确实是完整 PostgreSQL：

```powershell
docker run --rm pgvector/pgvector:pg17 postgres --version
```

### 5.2 复用现有容器（首选）

如果 `docker ps` 已经能看到 `pgvector`：

```powershell
docker ps --filter "name=pgvector"
docker exec -it pgvector psql -U postgres -d vectordb
```

直接使用即可，不需要再启动第二个 PostgreSQL。业务表建议放在独立 schema：

```sql
CREATE SCHEMA IF NOT EXISTS app;
ALTER ROLE postgres SET search_path TO app, public;
```

`vector_store` 留在 `public`，业务表放 `app`，两者在同一个数据库里，可以直接 JOIN。

### 5.3 新建 pgvector 容器（docker compose）

新建目录，例如 `E:\docker\pgvector`：

```text
E:\docker\pgvector\
├─ docker-compose.yml
└─ init\
   └─ 01-vector.sql
```

`docker-compose.yml`：

```yaml
services:
  pgvector:
    image: pgvector/pgvector:pg17
    container_name: pgvector
    restart: unless-stopped
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: vectordb
      TZ: Asia/Shanghai
      PGTZ: Asia/Shanghai
    ports:
      - "15432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d vectordb"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

`init/01-vector.sql`：

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

端口说明：

- 示例用 `15432`，是为了避开 Windows 保留端口段；
- 如果 5432 空闲，可以改成 `"5432:5432"`；
- 如果已有 pgvector 占用 5432，新容器用 `5433` 或 `15432`；
- Spring Boot 的 URL 端口要跟着改。

### 5.4 启动与验证

```powershell
cd E:\docker\pgvector
docker compose up -d
docker compose ps
docker compose logs -f pgvector
```

验证版本和扩展：

```powershell
docker exec -it pgvector psql -U postgres -d vectordb -c "SELECT version();"
docker exec -it pgvector psql -U postgres -d vectordb -c "SELECT extname, extversion FROM pg_extension;"
docker exec -it pgvector psql -U postgres -d vectordb -c "SELECT '[1,2,3]'::vector <-> '[4,5,6]'::vector AS distance;"
```

能看到 `vector | 0.8.x`，并且距离查询返回数字，就说明 pgvector 正常。

### 5.5 新建普通 PostgreSQL 容器（可选）

如果只是要一个独立的普通 PostgreSQL，不涉及向量：

```powershell
docker run -d --name postgres17 -e POSTGRES_USER=postgres -e POSTGRES_PASSWORD=postgres -e POSTGRES_DB=mydb -e TZ=Asia/Shanghai -p 5433:5432 -v postgres17_data:/var/lib/postgresql/data --restart unless-stopped postgres:17
```

或 compose：

```yaml
services:
  postgres:
    image: postgres:17
    container_name: postgres17
    restart: unless-stopped
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: mydb
      TZ: Asia/Shanghai
      PGTZ: Asia/Shanghai
    ports:
      - "5433:5432"
    volumes:
      - postgres17_data:/var/lib/postgresql/data
      - ./init:/docker-entrypoint-initdb.d:ro

volumes:
  postgres17_data:
```

再次强调：既然已有 `pgvector` 容器，普通业务表完全可以直接放进去，不一定需要这个独立实例。

### 5.6 PostgreSQL 18 的数据目录变化

PostgreSQL 18 官方镜像把数据目录改成了：

```text
/var/lib/postgresql/18/docker
```

推荐挂载点：

```yaml
volumes:
  - pgdata18:/var/lib/postgresql
```

如果继续挂 `/var/lib/postgresql/data`，可能看起来启动了，但数据没有按预期持久化。使用 `postgres:17` 或 `pgvector/pgvector:pg17` 时仍按 `/var/lib/postgresql/data`。

### 5.7 初始化脚本

`/docker-entrypoint-initdb.d` 里的脚本**只在数据目录为空时执行一次**。首次启动后，再改脚本不会自动生效。

目录示例：

```text
init\
├─ 01-vector.sql
├─ 02-schema.sql
└─ 03-data.sql
```

如果需要后续版本化迁移，使用 Flyway 或 Liquibase，不要依赖 init 脚本反复执行。

### 5.8 数据持久化

| 方式 | 写法 | 特点 |
|---|---|---|
| 命名卷 | `pgdata:/var/lib/postgresql/data` | 推荐，Docker 管理 |
| 绑定目录 | `C:\docker\pgdata:/var/lib/postgresql/data` | 文件直观，但 Windows 权限和性能问题多 |

```powershell
docker compose down      # 删除容器，保留命名卷
docker compose down -v   # 连数据卷一起删除，数据永久丢失
```

环境变量 `POSTGRES_PASSWORD` 也只在首次初始化时生效。之后改密码要进库执行：

```sql
ALTER USER postgres WITH PASSWORD 'new_password';
```

---

## 六、pgvector 初始化与验证

### 6.1 进入 psql

```powershell
docker exec -it pgvector psql -U postgres -d vectordb
```

先确认连接：

```sql
SELECT version();
SELECT current_database(), current_user, now();
```

### 6.2 启用扩展

```sql
CREATE EXTENSION IF NOT EXISTS vector;

SELECT extname, extversion
FROM pg_extension
WHERE extname = 'vector';
```

扩展是**按数据库**安装的。换一个数据库要重新执行 `CREATE EXTENSION`。

### 6.3 建向量表

```sql
CREATE TABLE IF NOT EXISTS items (
    id         bigserial PRIMARY KEY,
    content    text NOT NULL,
    metadata   jsonb NOT NULL DEFAULT '{}'::jsonb,
    embedding  vector(1536)
);
```

`vector(n)` 的维度必须和 embedding 模型一致。常见取值：

| 模型 | 维度 |
|---|---|
| OpenAI `text-embedding-3-small` | 1536 |
| OpenAI `text-embedding-3-large` | 3072 |
| 千问 `text-embedding-v4` | 1024（常用配置） |
| BGE-M3 | 1024 |

写错维度会报：

```text
expected 1536 dimensions, not 1024
```

### 6.4 插入测试数据

生成 500 条随机向量：

```sql
INSERT INTO items (content, embedding)
SELECT '文档 ' || g,
       (SELECT array_agg(random())::vector FROM generate_series(1, 1536))
FROM generate_series(1, 500) AS g;
```

插入单条：

```sql
INSERT INTO items (content, embedding)
VALUES ('测试文档', '[0.1, 0.2, 0.3]'::vector);
```

### 6.5 相似度查询

```sql
SELECT id,
       content,
       1 - (embedding <=> (SELECT embedding FROM items WHERE id = 1)) AS similarity
FROM items
WHERE id <> 1
ORDER BY embedding <=> (SELECT embedding FROM items WHERE id = 1)
LIMIT 5;
```

距离运算符与索引 operator class 必须匹配：

| 运算符 | 含义 | operator class |
|---|---|---|
| `<->` | 欧氏距离 | `vector_l2_ops` |
| `<=>` | 余弦距离 | `vector_cosine_ops` |
| `<#>` | 负内积 | `vector_ip_ops` |
| `<+>` | L1 距离 | `vector_l1_ops` |

`ORDER BY` 里必须直接写向量运算符。写成 `ORDER BY similarity` 通常用不上索引。

### 6.6 建索引

数据量小（几千条以内）可以先不建索引，全表扫描反而更快更准。数据量上来后再建：

```sql
-- HNSW：查询快、召回高，建索引慢且吃内存，当前首选
CREATE INDEX IF NOT EXISTS items_emb_hnsw
ON items USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- IVFFlat：建得快，召回依赖 lists/probes，必须在有数据后建
CREATE INDEX IF NOT EXISTS items_emb_ivfflat
ON items USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

HNSW 和 IVFFlat 二选一即可，不要同时建两个相同距离的索引，否则写入和磁盘开销都会增加。

### 6.7 参数调优

| 索引 | 参数 | 默认 | 说明 |
|---|---|---|---|
| HNSW | `m` | 16 | 每个节点连接数，越大越准越占内存 |
| HNSW | `ef_construction` | 64 | 建索引候选数，越大越慢、图质量越好 |
| HNSW | `hnsw.ef_search` | 40 | 查询候选数，越大越准越慢 |
| IVFFlat | `lists` | 无默认 | 一般取行数 / 1000（百万行以内） |
| IVFFlat | `ivfflat.probes` | 1 | 查询扫描的聚类数，越大越准越慢 |

```sql
SET hnsw.ef_search = 100;
SET ivfflat.probes = 10;

SHOW hnsw.ef_search;
SHOW ivfflat.probes;
```

只对当前会话有效。要持久化：

```sql
ALTER DATABASE vectordb SET hnsw.ef_search = 100;
ALTER ROLE postgres SET hnsw.ef_search = 100;
```

### 6.8 确认索引生效

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id
FROM items
ORDER BY embedding <=> (SELECT embedding FROM items WHERE id = 1)
LIMIT 5;
```

输出里应出现：

```text
Index Scan using items_emb_hnsw on items
```

如果看到 `Seq Scan`，检查：

- `ORDER BY` 是否直接写了向量运算符；
- 运算符和索引 operator class 是否匹配；
- 表是否太小，规划器主动选择全表扫描；
- 是否执行过 `ANALYZE items;`；
- 查询向量是否发生了隐式类型转换。

### 6.9 超过 2000 维怎么办

`vector` 类型的索引上限是 **2000 维**。使用 3072 维模型时：

**方案一：改用 halfvec**

```sql
CREATE TABLE items_half (
    id        bigserial PRIMARY KEY,
    content   text,
    embedding halfvec(3072)
);

CREATE INDEX items_half_hnsw
ON items_half USING hnsw (embedding halfvec_cosine_ops);
```

`halfvec` 索引上限 4000 维，存储约为 `vector` 的一半，精度损失很小。

**方案二：降维**

把 3072 维截断或投影到 1536 维。需要模型侧支持，不能随意截断，否则语义会变。

### 6.10 Spring AI 的 vector_store 表

Spring AI 自动建表时使用如下结构（`metadata` 是 `json`，不是 `jsonb`）：

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

- `embedding` 维度必须和 `spring.ai.vectorstore.pgvector.dimensions` 一致；
- `metadata` 用 `json`，手动建表时不要擅自改成 `jsonb`；
- 索引名可以不同，但 operator class 要和 `distance-type` 匹配；
- 扩展要建在当前数据库里。

### 6.11 业务表和向量表怎么放

推荐在同一个数据库里用 schema 隔离：

```sql
CREATE SCHEMA IF NOT EXISTS app;
```

业务表：

```sql
CREATE TABLE app.app_user (
    id          bigint GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    username    varchar(64) NOT NULL UNIQUE,
    created_at  timestamptz NOT NULL DEFAULT now()
);
```

向量表仍在 `public`：

```sql
CREATE TABLE public.vector_store (
    ...
);
```

这样可以直接做“先按业务字段过滤，再按向量排序”的查询，也能在同一个事务里同时写业务数据和向量数据。

设置默认 schema：

```sql
ALTER ROLE postgres SET search_path TO app, public;
```

Spring JPA 侧对应：

```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_schema: app
```

实体上也可以显式写：

```java
@Table(name = "app_user", schema = "app")
```

---

## 七、Spring Boot 对接 PostgreSQL

### 7.1 先判断你属于哪种情况

| 情况 | 需要的依赖 | 数据源 |
|---|---|---|
| 只用 Spring AI 的 pgvector | `spring-ai-starter-vector-store-pgvector` | `spring.datasource` |
| 业务表用 JPA | JPA starter + PostgreSQL 驱动 | 同一个 `spring.datasource` |
| 业务表用 JdbcTemplate | JDBC starter + PostgreSQL 驱动 | 同一个 `spring.datasource` |
| 同时接 MySQL | 多数据源 | 一个主库 + 一个从库 |

如果项目里已经有 pgvector 数据源（`spring.datasource` 指向 `vectordb`），**不需要再创建一个数据源**。直接加 JPA/JDBC 依赖，让业务表复用同一个连接池即可。

### 7.2 Maven 依赖

JPA 方案：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

只用 JdbcTemplate：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>

<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

Gradle：

```groovy
implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
runtimeOnly 'org.postgresql:postgresql'
```

`postgresql` 驱动版本由 Spring Boot BOM 管理，通常不用手写。

### 7.3 application.yaml

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:${DB_PORT:5432}/vectordb
    username: ${DB_USERNAME:postgres}
    password: ${DB_PASSWORD:postgres}
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000

  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate
    properties:
      hibernate:
        default_schema: app
        jdbc:
          time_zone: Asia/Shanghai
    show-sql: true
```

说明：

- `${DB_PORT:5432}` 表示默认 5432，可用环境变量覆盖；如果你的容器映射是 15432，就设置 `DB_PORT=15432`。
- `ddl-auto`：开发期可用 `update`，生产用 `validate`，表结构交给 Flyway/Liquibase。
- `default_schema: app` 要和实际业务 schema 一致。
- `open-in-view: false` 避免视图层隐式查库。
- PostgreSQL 不需要手动指定 Hibernate dialect。

如果暂时没有 `app` schema，把 `default_schema` 去掉，或者先执行：

```sql
CREATE SCHEMA IF NOT EXISTS app;
```

### 7.4 建表 SQL

```sql
CREATE SCHEMA IF NOT EXISTS app;

CREATE TABLE IF NOT EXISTS app.app_user (
    id          BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    username    VARCHAR(64)  NOT NULL UNIQUE,
    email       VARCHAR(128),
    status      SMALLINT     NOT NULL DEFAULT 1,
    profile     JSONB,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ  NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS idx_app_user_created_at
    ON app.app_user (created_at DESC);
```

### 7.5 JPA 实体

```java
package com.example.demo.user;

import jakarta.persistence.*;
import org.hibernate.annotations.JdbcTypeCode;
import org.hibernate.type.SqlTypes;

import java.time.OffsetDateTime;

@Entity
@Table(name = "app_user", schema = "app")
public class AppUser {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 64)
    private String username;

    @Column(length = 128)
    private String email;

    @Column(nullable = false)
    private Short status = 1;

    @JdbcTypeCode(SqlTypes.JSON)
    @Column(columnDefinition = "jsonb")
    private String profile;

    @Column(name = "created_at", nullable = false, insertable = false, updatable = false)
    private OffsetDateTime createdAt;

    @Column(name = "updated_at", nullable = false, insertable = false, updatable = false)
    private OffsetDateTime updatedAt;

    // getter / setter 省略
}
```

映射要点：

- `BIGINT IDENTITY` 用 `GenerationType.IDENTITY`；
- `TIMESTAMPTZ` 用 `OffsetDateTime` 或 `Instant`；
- 不要用 `LocalDateTime` 映射 `TIMESTAMPTZ`，否则容易差 8 小时；
- `JSONB` 在 Hibernate 6 中用 `@JdbcTypeCode(SqlTypes.JSON)`；
- 原生 SQL 写 JSONB 时可能需要 `?::jsonb` 强转。

### 7.6 Repository 与 Service

```java
public interface AppUserRepository extends JpaRepository<AppUser, Long> {

    Optional<AppUser> findByUsername(String username);

    boolean existsByEmail(String email);
}
```

```java
@Service
public class AppUserService {

    private final AppUserRepository repository;

    public AppUserService(AppUserRepository repository) {
        this.repository = repository;
    }

    @Transactional
    public AppUser create(String username, String email) {
        AppUser user = new AppUser();
        user.setUsername(username);
        user.setEmail(email);
        return repository.save(user);
    }

    @Transactional(readOnly = true)
    public AppUser getByUsername(String username) {
        return repository.findByUsername(username)
                .orElseThrow(() -> new IllegalArgumentException("用户不存在: " + username));
    }
}
```

### 7.7 JdbcTemplate 写法

```java
@Repository
public class AppUserJdbcRepository {

    private final JdbcTemplate jdbcTemplate;

    public AppUserJdbcRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    public List<Map<String, Object>> findAll() {
        return jdbcTemplate.queryForList(
                "SELECT id, username, email, created_at FROM app.app_user ORDER BY id DESC");
    }

    public Long insert(String username, String email) {
        return jdbcTemplate.queryForObject(
                "INSERT INTO app.app_user(username, email) VALUES (?, ?) RETURNING id",
                Long.class, username, email);
    }
}
```

### 7.8 验证连接

```java
@SpringBootTest
class PostgresConnectionTest {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Test
    void connectionWorks() {
        String version = jdbcTemplate.queryForObject("SELECT version()", String.class);
        assertThat(version).contains("PostgreSQL");
    }
}
```

启动日志里出现：

```text
HikariPool-1 - Start completed
```

并且没有 `Connection refused`、`password authentication failed`，说明数据源通了。

### 7.9 Spring Boot 常见报错

| 报错 | 原因 | 解决 |
|---|---|---|
| `Connection refused` | 端口不对 / 容器没启动 | 核对 `docker port pgvector` |
| `password authentication failed` | 密码错误 | `ALTER USER postgres WITH PASSWORD ...` |
| `database "xxx" does not exist` | 库名写错 | `CREATE DATABASE xxx;` |
| `relation "xxx" does not exist` | schema/search_path 不对 | 写全名 `app.xxx` 或设置 `default_schema` |
| `Schema-validation: missing table` | `ddl-auto: validate` 且表没建 | 执行建表 SQL |
| `jsonb but expression is character varying` | JSONB 类型不匹配 | `?::jsonb` 或 `@JdbcTypeCode` |
| `FATAL: no pg_hba.conf entry` | 来源不允许 | 检查 `pg_hba.conf` |
| 时间差 8 小时 | `timestamp` + `LocalDateTime` | 改 `timestamptz` + `OffsetDateTime` |

### 7.10 同时接 MySQL：多数据源

一个 Spring Boot 应用默认只有一个 `spring.datasource`。如果 pgvector 已经占用它，增加 MySQL 最稳妥的方式是：

- pgvector 继续用 `spring.datasource`；
- MySQL 用自定义前缀 `mysql.datasource`；
- MySQL 的 `JdbcTemplate` 用 `@Qualifier` 区分。

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/vectordb
    username: postgres
    password: postgres

mysql:
  datasource:
    url: jdbc:mysql://localhost:3306/appdb?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai
    username: root
    password: root
```

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource")
    public DataSourceProperties pgDataSourceProperties() {
        return new DataSourceProperties();
    }

    @Bean
    @Primary
    @ConfigurationProperties("spring.datasource.hikari")
    public HikariDataSource pgDataSource(
            @Qualifier("pgDataSourceProperties") DataSourceProperties properties) {
        return properties.initializeDataSourceBuilder()
                .type(HikariDataSource.class)
                .build();
    }

    @Bean
    @ConfigurationProperties("mysql.datasource")
    public DataSourceProperties mysqlDataSourceProperties() {
        return new DataSourceProperties();
    }

    @Bean
    @ConfigurationProperties("mysql.datasource.hikari")
    public HikariDataSource mysqlDataSource(
            @Qualifier("mysqlDataSourceProperties") DataSourceProperties properties) {
        return properties.initializeDataSourceBuilder()
                .type(HikariDataSource.class)
                .build();
    }

    @Bean
    @Primary
    public JdbcTemplate pgJdbcTemplate(
            @Qualifier("pgDataSource") DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }

    @Bean
    public JdbcTemplate mysqlJdbcTemplate(
            @Qualifier("mysqlDataSource") DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```

使用：

```java
@Autowired
@Qualifier("mysqlJdbcTemplate")
private JdbcTemplate mysqlJdbcTemplate;
```

注意事项：

1. **一旦自己声明了 `DataSource` Bean，Spring Boot 的数据源自动配置就会退让**，所以主库也要显式声明，不能只声明 MySQL 那个。
2. 两个 `DataSource` 时必须有一个 `@Primary`；两个 `JdbcTemplate` 也一样，否则注入会报“期望一个但找到两个”。
3. `@Primary` 的 `JdbcTemplate` 会被 Spring AI 的 pgvector 自动配置使用，所以它必须指向 PostgreSQL。
4. MySQL 若用 JPA，需要单独的 `EntityManagerFactory` 和 `TransactionManager`；用 MyBatis 则配置独立的 `SqlSessionFactory`。
5. `spring.sql.init.mode: always` 只对主数据源生效。不要让 PostgreSQL 的 `schema.sql` 跑到 MySQL 上。
6. 如果非要把 MySQL 设为主库，必须排除 `PgVectorStoreAutoConfiguration`，手动创建 `pgDataSource`、`pgJdbcTemplate` 和 `PgVectorStore`，否则 pgvector 会去 MySQL 建向量表。

对当前项目来说，pgvector 留在 `spring.datasource` 最省事。

---

## 八、Spring AI 对接 pgvector

### 8.1 依赖

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-vector-store-pgvector</artifactId>
</dependency>
```

版本交给 `spring-ai-bom` 管理。这个 starter 会把 PostgreSQL JDBC 驱动和 `spring-boot-starter-jdbc` 一起带进来，所以只用向量库时不必再单独加 JDBC starter。

### 8.2 application.yaml

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:${DB_PORT:5432}/vectordb
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

如果项目已经有 `spring.datasource` 指向 pgvector，只补 `spring.ai.vectorstore.pgvector` 即可。

### 8.3 关键配置说明

| 配置 | 作用 |
|---|---|
| `index-type` | `HNSW` 或 `IVFFLAT` |
| `distance-type` | `COSINE_DISTANCE`、`EUCLIDEAN_DISTANCE`、`NEGATIVE_INNER_PRODUCT` |
| `dimensions` | 必须和 embedding 模型输出维度一致 |
| `initialize-schema` | 是否让 Spring AI 自动建表 |
| `schema-validation` | 启动时校验表结构 |
| `max-document-batch-size` | 批量写入文档的上限 |

### 8.4 initialize-schema 怎么选

| 值 | 适用场景 |
|---|---|
| `true` | 第一次跑通，让框架自动建 `vector_store` |
| `false` | 表结构由 `schema.sql` 或 Flyway 管理 |

推荐流程：

1. 第一次设 `initialize-schema: true`，启动应用，让 Spring AI 建表；
2. 导出 DDL：

```powershell
docker exec pgvector pg_dump -U postgres -d vectordb --schema-only --table=vector_store -f /tmp/vector_store.sql
docker cp pgvector:/tmp/vector_store.sql .\vector_store.sql
```

3. 把导出的 SQL 纳入版本管理；
4. 改回 `initialize-schema: false`，加上 `schema-validation: true`。

这样表结构一定和框架期望的一致，又不会在每次启动时隐式改表。

### 8.5 schema.sql 示例

如果你自己管理表结构，可以用：

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

对应配置：

```yaml
spring:
  sql:
    init:
      mode: always
  ai:
    vectorstore:
      pgvector:
        dimensions: 1024
        initialize-schema: false
        schema-validation: true
```

`schema.sql` 在每次启动时执行，所以里面的语句必须幂等，使用 `IF NOT EXISTS`。

### 8.6 维度必须对齐

`dimensions`、`vector_store.embedding`、嵌入模型输出三者必须一致：

```text
embedding 模型输出 1024 维
        ↓
vector_store.embedding vector(1024)
        ↓
spring.ai.vectorstore.pgvector.dimensions: 1024
```

任何一处不一致都会报类似：

```text
expected 1024 dimensions, not 1536
```

换嵌入模型时，表结构、索引、配置要一起调整。已有数据通常需要重新生成向量。

### 8.7 metadata 用 json 还是 jsonb

Spring AI 自动建的 `vector_store.metadata` 是 `json`。手动建表时建议保持一致：

```sql
metadata json
```

如果你改成 `jsonb`，大多数读写也能工作，但过滤查询、类型转换和框架版本升级时可能出现兼容问题。除非你明确需要 JSONB 索引，否则不要改。

### 8.8 Chat 模型和 Embedding 模型是两件事

Spring AI 中：

- `ChatModel` 负责对话生成；
- `EmbeddingModel` 负责把文本转成向量；
- `VectorStore` 负责存向量和相似度检索。

如果聊天模型服务不提供 embedding 接口（例如 DeepSeek），需要单独配置一个 embedding 服务，不要指望同一个 `base-url` 什么都有。

示例：

```yaml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o-mini
      embedding:
        options:
          model: text-embedding-3-small
```

具体属性名随 Spring AI 版本变化，以你项目里的 `spring-configuration-metadata.json` 或 IDE 提示为准。

### 8.9 用代码验证 VectorStore

```java
@SpringBootTest
class VectorStoreTest {

    @Autowired
    private VectorStore vectorStore;

    @Test
    void addAndSearch() {
        vectorStore.add(List.of(
                new Document("PostgreSQL 是一个关系型数据库"),
                new Document("pgvector 是 PostgreSQL 的向量扩展")
        ));

        List<Document> results = vectorStore.similaritySearch("向量数据库");
        assertThat(results).isNotEmpty();
    }
}
```

`Document` 来自 `org.springframework.ai.document.Document`。

### 8.10 带过滤条件的检索

```java
SearchRequest request = SearchRequest.builder()
        .query("年假政策")
        .topK(5)
        .similarityThreshold(0.7)
        .filterExpression("source == 'employee-handbook.pdf'")
        .build();

List<Document> documents = vectorStore.similaritySearch(request);
```

过滤条件很严格时，HNSW 可能召回不足。可以：

- 提高 `hnsw.ef_search`；
- 使用部分索引；
- 开启 `hnsw.iterative_scan = relaxed_order`；
- 把常用过滤字段冗余到 metadata 中。

### 8.11 schema-validation 报错

启动时报：

```text
Schema validation failed
```

检查：

```sql
\d vector_store

SELECT column_name, data_type
FROM information_schema.columns
WHERE table_name = 'vector_store';
```

重点核对：

- 表是否存在；
- `embedding` 维度；
- `metadata` 类型；
- 索引是否存在且 operator class 匹配；
- 当前数据库是否装了 `vector` 扩展。

临时排查可以把 `schema-validation` 设为 `false`，确认连接和读写是否正常，但不要长期关闭。

### 8.12 Spring AI 常见报错

| 报错 | 原因 | 解决 |
|---|---|---|
| `type "vector" does not exist` | 当前库没装扩展 | `CREATE EXTENSION vector;` |
| `expected 1024 dimensions, not 1536` | 维度不一致 | 改配置或改表 |
| `Schema validation failed` | 表结构不匹配 | 按 8.5 重建表 |
| `relation "vector_store" does not exist` | 没建表 | `initialize-schema: true` 或执行 schema.sql |
| `column "metadata" is of type json but expression is of type jsonb` | 类型不一致 | 保持 `json` |
| 检索结果为空 | 没写入文档 / 阈值太高 | 先 `vectorStore.add`，降低阈值 |

---

## 九、运维、备份与迁移

### 9.1 日常容器命令

```powershell
docker compose ps
docker compose logs -f pgvector
docker compose restart pgvector
docker compose stop
docker compose start
docker stats --no-stream pgvector
```

直接查看容器：

```powershell
docker ps --filter "name=pgvector"
docker logs --tail 100 pgvector
docker exec -it pgvector bash
```

### 9.2 常用 psql 命令

| 命令 | 作用 |
|---|---|
| `\l` | 数据库列表 |
| `\c vectordb` | 切换数据库 |
| `\dt` | 表列表 |
| `\d vector_store` | 表结构 |
| `\di` | 索引 |
| `\dn` | schema |
| `\du` | 用户 |
| `\dx` | 扩展 |
| `\x` | 纵向显示 |
| `\timing` | 显示耗时 |
| `\q` | 退出 |

完整命令见 `PostgreSQL&pgvector命令速查手册.md`。

### 9.3 备份

推荐自定义格式：

```powershell
docker exec -t pgvector pg_dump -U postgres -d vectordb -Fc -Z 6 -f /tmp/vectordb.dump
docker cp pgvector:/tmp/vectordb.dump C:\backup\vectordb.dump
```

只备份结构：

```powershell
docker exec -t pgvector pg_dump -U postgres -d vectordb -s -f /tmp/vectordb_schema.sql
docker cp pgvector:/tmp/vectordb_schema.sql C:\backup\vectordb_schema.sql
```

只备份某张表：

```powershell
docker exec -t pgvector pg_dump -U postgres -d vectordb -t vector_store -Fc -f /tmp/vector_store.dump
```

### 9.4 恢复

```powershell
docker cp C:\backup\vectordb.dump pgvector:/tmp/vectordb.dump
docker exec -t pgvector pg_restore -U postgres -d vectordb --clean --if-exists /tmp/vectordb.dump
```

恢复到新库：

```powershell
docker exec -it pgvector psql -U postgres -c "CREATE DATABASE vectordb_new;"
docker exec -t pgvector pg_restore -U postgres -d vectordb_new --no-owner /tmp/vectordb.dump
```

### 9.5 不要用 PowerShell 重定向备份

不推荐：

```powershell
docker exec pgvector pg_dump -U postgres -d vectordb > backup.sql
```

Windows PowerShell 5.x 的 `>` 可能写出 UTF-16LE，恢复时报编码错误。用 `-f` 在容器内写文件，再 `docker cp` 出来最稳。

### 9.6 迁移到新容器

```powershell
# 旧容器导出
docker exec -t pgvector pg_dump -U postgres -d vectordb -Fc -f /tmp/vectordb.dump
docker cp pgvector:/tmp/vectordb.dump C:\backup\vectordb.dump

# 新容器建库
docker exec -it pgvector_new psql -U postgres -c "CREATE DATABASE vectordb;"

# 导入
docker cp C:\backup\vectordb.dump pgvector_new:/tmp/vectordb.dump
docker exec -t pgvector_new pg_restore -U postgres -d vectordb --no-owner /tmp/vectordb.dump
```

如果原库用了 pgvector，新镜像也必须是 `pgvector/pgvector:pg17`。普通 `postgres:17` 没有 `vector` 扩展。

### 9.7 磁盘清理

```powershell
docker system df
docker system prune -a
```

`docker system prune -a` 会删除未使用的镜像、容器和网络，执行前确认没有需要保留的临时容器。

`ext4.vhdx` 不会因为删除镜像自动缩小，必要时在 Docker Desktop 设置里做磁盘清理。

### 9.8 查看资源占用

```powershell
docker stats --no-stream pgvector
docker exec pgvector df -h
docker exec pgvector ps aux
```

数据库内部：

```sql
SELECT pg_size_pretty(pg_database_size(current_database()));
SELECT pg_size_pretty(pg_total_relation_size('vector_store'));
SELECT * FROM pg_stat_activity WHERE state <> 'idle';
```

---

## 十、故障排查

### 10.1 `wsl --install` 提示「与服务器的连接被重置」

先重启电脑，再执行：

```powershell
wsl --install -d Ubuntu
```

仍然失败就换下载通道：

```powershell
wsl --install -d Ubuntu --web-download
```

或者走 Microsoft Store / 离线 rootfs 导入。根因通常是下载通道被重置，不是 WSL 本身坏了。详细步骤见 3.2 节。

### 10.2 `wsl --update` 报 403

```text
Wsl/UpdatePackage/HTTP_E_STATUS_FORBIDDEN
```

通常可以直接忽略。它只是检查更新失败，不影响已安装的 WSL。要修就检查 WinHTTP 代理：

```powershell
netsh winhttp show proxy
```

### 10.3 `pull access denied ... may require 'docker login'`

```powershell
curl.exe -v https://registry-1.docker.io/v2/
docker login
docker pull hello-world
```

返回 401 说明网络通，登录即可；403/超时说明代理没通，检查 Docker Desktop 的 Proxies 和 WSL2 的 `127.0.0.1` 问题。详见 4.3、4.4 节。

### 10.4 端口绑定失败 `access permissions`

报错：

```text
ports are not available: ... bind: An attempt was made to access a socket
in a way forbidden by its access permissions
```

这不是普通的“端口被占用”，而是端口落在 Windows 保留段里。诊断：

```powershell
netsh int ipv4 show excludedportrange protocol=tcp
netsh int ipv4 show dynamicport tcp
```

如果 5432 落在保留区间里，按简单到彻底排序：

**A. 换端口（推荐先做）**

把 compose 改成：

```yaml
ports:
  - "15432:5432"
```

Spring Boot 相应设置：

```powershell
$env:DB_PORT = "15432"
```

或者直接改 `application.yaml` 的 URL。

**B. 重启 winnat（临时缓解）**

管理员 PowerShell：

```powershell
net stop winnat
net start winnat
```

重启电脑后可能复发。

**C. 恢复动态端口范围（彻底，需重启）**

```powershell
netsh int ipv4 set dynamic tcp start=49152 num=16384
netsh int ipv6 set dynamic tcp start=49152 num=16384
```

### 10.5 `.wslconfig` 不生效

检查：

```powershell
Get-Item "$env:USERPROFILE\.wslconfig" | Select-Object FullName, Attributes
```

必须是**文件**，不能是文件夹；必须放在 `C:\Users\<用户名>\` 下。改完执行：

```powershell
wsl --shutdown
```

最常见的错误是记事本保存成了 `.wslconfig.txt`。

### 10.6 容器访问不到外部 API

Docker 容器走 NAT 再经 Windows 出去。如果日志里出现连接超时：

1. 在 Docker Desktop → Settings → Resources → Proxies 配代理；
2. 确认代理软件允许局域网；
3. WSL2 里不要填 `127.0.0.1`，填 Windows 的局域网 IP；
4. 测试：

```powershell
docker run --rm curlimages/curl -v https://api.openai.com
```

### 10.7 磁盘只增不减

```powershell
docker system df
docker system prune -a
```

`ext4.vhdx` 不会自动缩小。清理后在 Docker Desktop 设置里做磁盘压缩，或迁移 WSL 发行版到更大的磁盘。

### 10.8 psql 没有响应 / 提示符是 `-#`

- `vectordb=#`：正常，等待 SQL；
- `vectordb-#`：续行状态，上一条语句没结束；
- `vectordb'#`：单引号没闭合；
- `vectordb"#`：双引号没闭合；
- `vectordb(#`：括号没闭合。

按 `Ctrl+C` 或输入 `\r` 清空缓冲区。每条 SQL 都要以分号 `;` 结尾。

### 10.9 `port is already allocated`

```powershell
docker ps --format "table {{.Names}}\t{{.Ports}}"
```

已有容器占用端口。改映射：

```yaml
ports:
  - "5433:5432"
```

或者停止不用的容器：

```powershell
docker stop <容器名>
```

### 10.10 `database files are incompatible with server`

数据卷是另一个 PostgreSQL 大版本创建的。处理：

- 换新卷；
- 用 `pg_upgrade` 迁移；
- 用 `pg_dump` / `pg_restore` 逻辑迁移。

不要直接改镜像 tag 后继续用旧数据目录。

### 10.11 改了 `POSTGRES_PASSWORD` 但不生效

环境变量只在数据目录为空时初始化。进库修改：

```sql
ALTER USER postgres WITH PASSWORD 'new_password';
```

### 10.12 Spring Boot 连不上

按顺序检查：

```powershell
docker ps --filter "name=pgvector"
docker port pgvector
Test-NetConnection localhost -Port 5432
docker logs --tail 100 pgvector
docker exec -it pgvector psql -U postgres -d vectordb -c "SELECT 1;"
```

然后核对 `application.yaml`：

- URL 端口是否和 `docker port` 一致；
- 数据库名是否为 `vectordb`；
- 用户名密码是否正确；
- 是否多了空格或引号；
- `spring.sql.init.mode` 是否导致启动时执行了错误脚本。

### 10.13 pgvector 维度或索引错误

```sql
SELECT vector_dims(embedding) FROM items LIMIT 1;
SELECT extversion FROM pg_extension WHERE extname = 'vector';
\d items
```

常见错误：

| 报错 | 解决 |
|---|---|
| `expected 1536 dimensions, not 1024` | 统一模型、表、配置的维度 |
| `type "vector" does not exist` | `CREATE EXTENSION vector;` |
| `operator does not exist: vector <-> text` | 加 `::vector` |
| `column cannot have more than 2000 dimensions` | 用 `halfvec` 或降维 |
| 查询走 `Seq Scan` | 检查运算符、索引、`ANALYZE` |
| 召回结果少于 `LIMIT` | 提高 `ef_search`，开迭代扫描 |

### 10.14 应用启动时 `schema.sql` 报错

`spring.sql.init.mode: always` 会在每次启动时执行 `schema.sql`。检查：

- 当前数据源是不是 PostgreSQL；
- SQL 是否幂等（`IF NOT EXISTS`）；
- 是否有 MySQL 专用语法；
- 扩展是否已安装；
- `search_path` 是否正确。

如果同时接了 MySQL，不能让 PostgreSQL 的初始化脚本跑在 MySQL 上。

---

## 十一、最小验收清单

### 11.1 Docker 层

```powershell
docker version
docker ps --filter "name=pgvector"
docker logs --tail 50 pgvector
```

- [ ] 客户端和服务端版本都能打印
- [ ] 容器状态是 `Up`，最好是 `healthy`
- [ ] 日志没有反复重启或认证错误

### 11.2 PostgreSQL 层

```powershell
Test-NetConnection localhost -Port 5432
docker exec -it pgvector psql -U postgres -d vectordb -c "SELECT version();"
```

- [ ] 宿主机端口可连
- [ ] 能进入 `vectordb`
- [ ] `SELECT version()` 返回 PostgreSQL 17.x

### 11.3 pgvector 层

```powershell
docker exec -it pgvector psql -U postgres -d vectordb -c "SELECT extname, extversion FROM pg_extension;"
docker exec -it pgvector psql -U postgres -d vectordb -c "\dt"
```

- [ ] 扩展列表里有 `vector`
- [ ] `vector_store` 或 `items` 表存在
- [ ] `SELECT '[1,2,3]'::vector <-> '[4,5,6]'::vector;` 能返回数字

### 11.4 Spring Boot 层

- [ ] `pom.xml` 有 JPA/JDBC 和 PostgreSQL 驱动
- [ ] `application.yaml` 的端口、库名、用户密码正确
- [ ] 启动日志出现 `HikariPool-1 - Start completed`
- [ ] 查询接口能返回数据
- [ ] `ddl-auto` 和 `schema.sql` 策略明确

### 11.5 Spring AI 层

- [ ] `spring-ai-starter-vector-store-pgvector` 已引入
- [ ] `dimensions` 和 embedding 模型一致
- [ ] `initialize-schema` 与 `schema-validation` 组合正确
- [ ] `vectorStore.add(...)` 能写入
- [ ] `vectorStore.similaritySearch(...)` 能返回结果

### 11.6 备份层

```powershell
docker exec -t pgvector pg_dump -U postgres -d vectordb -Fc -f /tmp/check.dump
docker cp pgvector:/tmp/check.dump .\check.dump
```

- [ ] 备份文件能生成
- [ ] 备份文件大小正常
- [ ] 至少做过一次恢复演练

---

## 十二、附录：端口、目录与一条龙命令

### 12.1 推荐目录结构

```text
E:\docker\pgvector\
├─ docker-compose.yml
├─ init\
│  ├─ 01-vector.sql
│  └─ 02-schema.sql
└─ backup\
```

### 12.2 端口对照

| 服务 | 容器内 | 宿主机 | 说明 |
|---|---|---|---|
| 现有 pgvector | 5432 | 5432 | 本机当前 |
| 新 pgvector | 5432 | 15432 | 避开保留段 |
| 新 PostgreSQL | 5432 | 5433 | 与现有容器并存 |
| MySQL | 3306 | 3306 | 可选 |

### 12.3 一条龙命令

```powershell
# 1. 检查 WSL2
wsl --version
wsl -l -v

# 2. 检查 Docker
docker version
docker run hello-world

# 3. 启动 pgvector
cd E:\docker\pgvector
docker compose up -d
docker compose ps

# 4. 验证数据库
docker exec -it pgvector psql -U postgres -d vectordb -c "SELECT version();"
docker exec -it pgvector psql -U postgres -d vectordb -c "SELECT extname, extversion FROM pg_extension;"

# 5. 验证向量
docker exec -it pgvector psql -U postgres -d vectordb -c "SELECT '[1,2,3]'::vector <#> '[4,5,6]'::vector;"

# 6. 日常状态
docker compose logs -f pgvector
docker stats --no-stream pgvector
```

### 12.4 Spring Boot 启动前检查

```text
[ ] DB_PORT 和 docker port 一致
[ ] DB_NAME=vectordb
[ ] DB_USERNAME=postgres
[ ] DB_PASSWORD 正确
[ ] vector 扩展已安装
[ ] vector_store 表存在
[ ] dimensions 和 embedding 模型一致
[ ] initialize-schema 策略明确
```

### 12.5 相关文档

| 文档 | 用途 |
|---|---|
| `PostgreSQL&pgvector命令速查手册.md` | 全量 SQL、psql、运维、备份、排障命令 |
| `PostgreSQL-Docker-安装与SpringBoot对接指南.md` | 原始 PostgreSQL 安装与 JPA 对接 |
| `Windows-Docker-pgvector-安装指南.md` | 原始 Windows/WSL2/Docker 踩坑记录 |

### 12.6 一句话总结

Windows 上最省事的路线是：**WSL2 + Docker Desktop + `pgvector/pgvector:pg17` 一个容器**。业务表放 `app` schema，向量表放 `public`，Spring Boot 只配一个 `spring.datasource`；需要 MySQL 时再加第二个数据源，不要把 pgvector 的数据源挤掉。

