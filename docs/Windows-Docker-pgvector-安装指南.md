# Windows 上安装 Docker Desktop + PostgreSQL/pgvector 完整指南

> 适用环境：Windows 10 Pro 22H2（内部版本 19045 及以上）/ WSL2 后端
> 本文基于一台实机的完整踩坑过程整理，每一步都附了验证命令。

---

## 目录

1. [前置检查](#一前置检查)
2. [启用 WSL2](#二启用-wsl2)
3. [安装 Docker Desktop](#三安装-docker-desktop)
4. [打通镜像拉取（网络配置）](#四打通镜像拉取网络配置)
5. [部署 PostgreSQL + pgvector](#五部署-postgresql--pgvector)
6. [手动建表与索引](#六手动建表与索引)
7. [接入 Spring AI](#七接入-spring-ai)
8. [常见问题速查](#八常见问题速查)
9. [附录：本机实测记录](#九附录本机实测记录)

---

## 一、前置检查

开始之前先确认三件事，任何一项不满足后面都会失败。

**1. 系统版本**

```powershell
Get-ItemProperty 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion' |
  Select-Object ProductName, DisplayVersion, CurrentBuild
```

Docker Desktop 要求 Windows 10 22H2（内部版本 **19045** 及以上）或 Windows 11。低于这个版本只能装旧版 Docker Desktop。

**2. BIOS 里的虚拟化**

重启进 BIOS/UEFI，把 `Intel Virtualization Technology`（VT-x）或 `AMD-V / SVM Mode` 设为 Enabled。开机后在**任务管理器 → 性能 → CPU**，右下角应显示「虚拟化：已启用」。

**3. 有没有东西占着目标端口**

```powershell
Get-NetTCPConnection -State Listen -LocalPort 5432 -ErrorAction SilentlyContinue
```

没有输出就是空闲。**另外还要查保留端口段**（这一步很容易漏，见[第八节](#八常见问题速查)）：

```powershell
netsh int ipv4 show excludedportrange protocol=tcp
```

---

## 二、启用 WSL2

**管理员身份**打开 PowerShell：

```powershell
wsl --install
```

这条命令做三件事：启用「虚拟机平台」、启用「适用于 Linux 的 Windows 子系统」、下载安装 Ubuntu。**前两步是纯本地的，第三步需要联网。**

**执行完必须重启电脑**，否则功能不会真正生效。

### 如果卡在「正在安装: Ubuntu」或报「连接被重置」

前两步成功（屏幕显示「已安装 虚拟机平台」「已安装 适用于 Linux 的 Windows 子系统」）但发行版下载失败，这是国内网络最常见的情况。按下面顺序处理：

**第一步：先重启，再重试一次。** 功能刚启用时子系统还没就位，重启后重跑经常就过了：

```powershell
wsl --install -d Ubuntu
```

注意进度条**长期停在 0.0% 是正常的**，它不刷新。判断是否真的装好，看这个文件的大小：

```powershell
Get-ChildItem "$env:LOCALAPPDATA\wsl" -Recurse -File |
  Select-Object FullName, @{n='MB';e={[math]::Round($_.Length/1MB,1)}}
```

`ext4.vhdx` 有 1GB 以上就说明 rootfs 已经解包成功。

**第二步：换下载通道。**

```powershell
wsl --install -d Ubuntu --web-download
```

或者打开 **Microsoft Store** 搜 `Ubuntu 22.04 LTS` 手动安装——它走的是另一套 CDN，经常一边不通另一边通。

**第三步：离线导入（最可靠）。**

从国内镜像站（清华 TUNA 的 `ubuntu-cloud-images` 镜像下 `wsl/jammy/current/`，或中科大、阿里云镜像站）下载 rootfs 压缩包，然后：

```powershell
New-Item -ItemType Directory D:\wsl\ubuntu -Force
wsl --import Ubuntu D:\wsl\ubuntu "D:\Downloads\ubuntu-jammy-wsl-amd64-wsl.rootfs.tar.gz" --version 2
wsl --set-default Ubuntu
```

注意：`--import` 方式装出来的发行版默认直接进 root，不会引导你创建用户，需要的话自己 `adduser`。

### 验证 WSL2

```powershell
wsl --version
wsl -l -v              # VERSION 列应该是 2
wsl -d Ubuntu -- uname -a
```

### 配置资源上限

新建 **文件** `C:\Users\<用户名>\.wslconfig`（注意是文件，不是文件夹）：

```ini
[wsl2]
memory=8GB
processors=4
swap=2GB
systemd=true
```

`memory` 按物理内存的一半左右给。`systemd=true` 是为了后面在 WSL 里跑 Docker 用的。

改完执行 `wsl --shutdown` 重新加载。

> **易错点**：记事本保存会自动补 `.txt`，结果变成 `.wslconfig.txt`，WSL 不认。保存时「保存类型」选**所有文件**，或者文件名两边加英文引号。

---

## 三、安装 Docker Desktop

```powershell
winget install Docker.DockerDesktop
```

装完**新开一个终端**（PATH 需要刷新），验证：

```powershell
docker version
```

客户端和服务端版本号都能打印出来才算通。

如果 `docker` 命令找不到，CLI 的完整路径是：

```
C:\Program Files\Docker\Docker\resources\bin\docker.exe
```

它在安装时会被写进机器级 PATH，只是老终端读不到。

### 关键设置

打开 Docker Desktop，进入设置：

| 位置 | 要做什么 |
|---|---|
| `Settings → General` | 按需开启开机自启 |
| `Settings → Resources → WSL Integration` | **打开你那个 Ubuntu 发行版的开关**，之后在 Ubuntu 终端里也能直接用 `docker`，且不需要 `sudo` |
| `Settings → Resources → Proxies` | 见下一节 |
| `Settings → Docker Engine` | 配镜像加速器，见下一节 |

---

## 四、打通镜像拉取（网络配置）

这是整个过程中最容易卡住的一环。典型报错：

```
docker: Error response from daemon: pull access denied for hello-world,
repository does not exist or may require 'docker login'
```

**注意：`hello-world` 是官方公开镜像，根本不需要登录。** 报这个错说明请求被拒了，不是镜像不存在。

### 第一步：判断卡在哪一层

先看域名解析和连通性：

```powershell
curl.exe -v https://registry-1.docker.io/v2/
```

- 返回 **401 + `www-authenticate: Bearer realm="https://auth.docker.io/token"`** → 网络是通的，这是 Docker Hub 的正常回复，跳到第二步
- 返回 **403 / 超时 / 连接重置** → 根本没连上，跳到第三步

再看 WSL 侧解析是否一致（排查 WSL2 有没有被代理接管）：

```powershell
wsl -d Ubuntu -- getent hosts registry-1.docker.io auth.docker.io
```

如果返回的是 `198.18.x.x` 这类地址，说明流量经过了 fake-IP 模式的代理（Clash 之类的 TUN 模式），两边的网络路径是一致的。

### 第二步：网络通但拉取被拒 → 登录即可

这是**匿名拉取被限流**导致的。Docker Hub 对未登录请求限制很严，走代理节点（机房 IP）时更容易被直接拒。

```powershell
docker login
docker pull hello-world
```

免费账号即可，登录用户的配额宽松得多。**绝大多数「pull access denied」到这一步就解决了。**

### 第三步：网络不通 → 配代理

先检查系统代理配置在哪一层：

```powershell
netsh winhttp show proxy
Get-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings' |
  Select-Object ProxyEnable, ProxyServer
```

**常见坑**：代理只配在了 WinINET 层（浏览器和普通软件走它），而 `wsl.exe`、系统下载组件走的是 **WinHTTP**，那条路是直连的。表现就是「浏览器能上 GitHub，命令行下载全被重置」。

修法（管理员）：

```powershell
netsh winhttp set proxy proxy-server="127.0.0.1:7897" bypass-list="<local>"
```

端口换成你代理软件的实际端口。还原用 `netsh winhttp reset proxy`。

**给 Docker 守护进程单独配代理**：`Settings → Resources → Proxies → Manual proxy configuration`，HTTP/HTTPS 都填。

> **WSL2 专用注意**：守护进程跑在 WSL2 的 `docker-desktop` 发行版里，WSL2 有独立的网络命名空间，**它里面的 `127.0.0.1` 不是 Windows 的 `127.0.0.1`**。所以要：
> 1. 在代理软件里打开 **Allow LAN**
> 2. `ipconfig` 找到 Windows 的局域网 IPv4（如 `192.168.1.100`）
> 3. 填 `http://192.168.1.100:7897`，**不要填 127.0.0.1**
>
> （Windows 11 22H2 以上有镜像网络模式可以让 127.0.0.1 生效，Windows 10 没有这个模式。）

### 备选：配镜像加速器

`Settings → Docker Engine`，在 JSON 里加：

```json
"registry-mirrors": ["https://你的加速地址"]
```

**不要照抄网上教程里的公共地址**——2024 年之后大量公共加速器已经关停，填上去只会得到和现在一样的报错。相对稳的是去阿里云/腾讯云/华为云的容器镜像服务控制台拿**账号专属**的加速地址，因为绑定账号，比匿名公共地址可靠。

### 验证

```powershell
docker run hello-world
docker run -d --name web -p 8080:80 nginx
```

第二条起来后浏览器打开 `http://localhost:8080`，能看到 nginx 欢迎页就说明整条链路（WSL2 → 守护进程 → Windows 端口转发）全通了。

---

## 五、部署 PostgreSQL + pgvector

### 核心结论

**不需要单独安装 PostgreSQL，也不需要自己编译 pgvector。**

官方镜像 **`pgvector/pgvector`** 本身就是完整的 PostgreSQL，只是把 vector 扩展预编译好放进去了。它有按大版本打的 tag：`pg16` / `pg17` / `pg18`。

| 镜像 | 是什么 | 要不要额外装 pgvector |
|---|---|---|
| `postgres:17` | 官方纯净版 | **要**——得自己编译扩展 |
| `pgvector/pgvector:pg17` | 同一个 PostgreSQL 17 + 预装扩展 | **不要**，开箱即用 |

验证它确实是完整 PostgreSQL：

```powershell
docker run --rm pgvector/pgvector:pg17 postgres --version
```

### 目录与文件

新建目录（如 `E:\docker\pgvector`），放两个文件。

**`docker-compose.yml`**：

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
    ports:
      - "15432:5432"          # 见下方端口说明
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d vectordb"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  pgdata:
```

**`init/01-vector.sql`**：

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

> `/docker-entrypoint-initdb.d` 里的脚本**只在数据目录为空时执行一次**（即首次启动）。之后想加东西得手动执行，或者删掉数据卷重来。

**关于端口**：主机侧用 `15432` 而不是 `5432`，因为容器化开发机上 `5432` 经常落在 Windows 的保留端口段里导致绑定失败，详见[第八节](#八常见问题速查)。如果你的 5432 可用，改回 `"5432:5432"` 也可以。

### 启动与验证

```powershell
cd E:\docker\pgvector
docker compose up -d
docker compose ps
docker compose logs -f pgvector

docker exec -it pgvector psql -U postgres -d vectordb -c "SELECT extname, extversion FROM pg_extension;"
```

能看到 `vector | 0.8.x` 这一行就成功了。再用一行 SQL 确认向量类型可用：

```powershell
docker exec -it pgvector psql -U postgres -d vectordb -c "SELECT '[1,2,3]'::vector <-> '[4,5,6]'::vector AS distance;"
```

---

## 六、手动建表与索引

进入交互式客户端：

```powershell
docker exec -it pgvector psql -U postgres -d vectordb
```

### 1. 建扩展（库级别，每个库都要建一次）

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

### 2. 建表

```sql
CREATE TABLE IF NOT EXISTS items (
    id         bigserial PRIMARY KEY,
    content    text NOT NULL,
    metadata   jsonb DEFAULT '{}'::jsonb,
    embedding  vector(1536)          -- 维度必须和 embedding 模型一致
);
```

**1536 是硬约束**，写进去维度不对会报 `expected 1536 dimensions, not 1024`。OpenAI 的 `text-embedding-3-small` 是 1536，`-large` 是 3072。

### 3. 插入测试数据

```sql
INSERT INTO items (content, embedding)
SELECT '文档 ' || g,
       (SELECT array_agg(random())::vector FROM generate_series(1, 1536))
FROM generate_series(1, 500) g;
```

### 4. 相似度查询

```sql
SELECT id, content,
       1 - (embedding <=> (SELECT embedding FROM items WHERE id = 1)) AS similarity
FROM items
ORDER BY embedding <=> (SELECT embedding FROM items WHERE id = 1)
LIMIT 5;
```

三个距离运算符必须和索引匹配使用：

| 运算符 | 距离 | 索引 operator class |
|---|---|---|
| `<->` | 欧氏距离 | `vector_l2_ops` |
| `<=>` | 余弦距离 | `vector_cosine_ops` |
| `<#>` | 负内积 | `vector_ip_ops` |

### 5. 建索引（数据量上千条之后再建）

```sql
-- HNSW：查询快、召回率高，建索引慢且吃内存，当前首选
CREATE INDEX IF NOT EXISTS items_emb_hnsw
    ON items USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);

-- 或者 IVFFlat：建得快，召回率依赖 lists 调优，且必须在有数据后建
CREATE INDEX IF NOT EXISTS items_emb_ivfflat
    ON items USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);
```

**数据量小（几千条以内）先别建索引**，全表扫描反而更快更准。

### 6. 参数调优

| 索引 | 参数 | 含义 |
|---|---|---|
| HNSW | `m`（默认 16） | 每个节点连接数，越大越准越占内存 |
| HNSW | `ef_construction`（默认 64） | 建索引时的候选数，越大建得越慢、图质量越好 |
| HNSW | `SET hnsw.ef_search = 100;` | 查询时用，默认 40，越大越准越慢 |
| IVFFlat | `lists` | 一般取「行数 / 1000」（百万行以内） |
| IVFFlat | `SET ivfflat.probes = 10;` | 查询时用 |

### 7. 确认索引真的生效

```sql
EXPLAIN ANALYZE
SELECT id FROM items ORDER BY embedding <=> (SELECT embedding FROM items WHERE id = 1) LIMIT 5;
```

输出里出现 `Index Scan using items_emb_hnsw` 才对。**`ORDER BY` 必须直接写向量运算符**，写成 `ORDER BY similarity` 就不会走索引。

### 8. 超过 2000 维怎么办

HNSW 和 IVFFlat 对 `vector` 类型的**索引上限是 2000 维**。用 3072 维模型时两个选择：

- 建表改用 `halfvec(3072)`（索引上限 4000，精度损失极小）
- 或把向量截断到 1536 维

这个问题只在建索引时才会暴露，建议早点决定。

---

## 七、接入 Spring AI

### 依赖

`spring-ai-starter-vector-store-pgvector`（版本交给 `spring-ai-bom` 管）：

```xml
<dependency>
  <groupId>org.springframework.ai</groupId>
  <artifactId>spring-ai-starter-vector-store-pgvector</artifactId>
</dependency>
```

这个 starter 会把 PostgreSQL JDBC 驱动和 `spring-boot-starter-jdbc` 一起带进来。

### 配置

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:15432/vectordb
    username: postgres
    password: postgres
  ai:
    vectorstore:
      pgvector:
        index-type: HNSW
        distance-type: COSINE_DISTANCE
        dimensions: 1536
        initialize-schema: true
```

属性名以你实际用的版本为准，建议在 IDE 里对 `PgVectorStoreProperties` 做一次 Ctrl+Click 核实。

### `initialize-schema` 该怎么设

| 值 | 适用场景 |
|---|---|
| `true` | 快速跑通，框架自动建 `vector_store` 表 |
| `false` | 生产环境，表结构由 DDL 脚本管理 |

**推荐做法：先自动建一次，导出 DDL 再自己管。**

```powershell
docker exec pgvector pg_dump -U postgres -d vectordb --schema-only --table=vector_store > schema.sql
```

把 `schema.sql` 纳入版本管理，然后把 `initialize-schema` 改回 `false`。这样表结构一定和框架期望的一致，又避免了应用启动时隐式改表。

> 注意：Spring AI 自动建的表用的是 `json` 类型而不是 `jsonb`。如果你手动建表，类型要和框架保持一致，否则会报错。

### 维度必须对齐

`dimensions` 必须和 embedding 模型输出维度一致。**如果主模型用的是不提供 embedding 接口的服务（比如 DeepSeek），需要把 embedding 单独指向另一个服务**，别指望同一个 base-url 什么都有。

---

## 八、常见问题速查

### 1. `ports are not available: ... bind: An attempt was made to access a socket in a way forbidden by its access permissions`

**这不是端口被占用**（注意措辞是 `access permissions` 而不是 `address in use`），是端口落在 Windows **保留端口段**里了。启用 WSL2/虚拟机平台之后很常见。

诊断：

```powershell
netsh int ipv4 show excludedportrange protocol=tcp
netsh int ipv4 show dynamicport tcp
```

如果目标端口落在输出的某个区间内（比如 `5341-5440` 包含 5432），就是它。另外注意动态端口范围，Windows 默认是**从 49152 开始、16384 个**；如果被改成了从 1024 开始，大量常用端口都会被卷进去。

三个方案，按简单到彻底：

```powershell
# A. 换端口（立刻可用，推荐先做这个）
#    选一个既不在保留段、也在动态范围（1024~15000）之外的端口，比如 15432
#    改 docker-compose.yml 的 ports，然后 docker compose up -d

# B. 重启 winnat（临时缓解，重启电脑后失效，需管理员）
net stop winnat
net start winnat

# C. 把动态端口范围改回默认（彻底解决，需管理员 + 重启电脑）
netsh int ipv4 set dynamic tcp start=49152 num=16384
netsh int ipv6 set dynamic tcp start=49152 num=16384
```

### 2. `pull access denied ... may require 'docker login'`

见[第四节](#四打通镜像拉取网络配置)。先 `curl.exe -v https://registry-1.docker.io/v2/` 判断是网络问题还是限流问题；网络通就 `docker login`。

### 3. `wsl --install` 报「与服务器的连接被重置」

见[第二节](#二启用-wsl2)。核心是 `wsl.exe` 走 WinHTTP 通道，而代理常常只配在 WinINET 层。

### 4. `wsl --update` 报 403 `Wsl/UpdatePackage/HTTP_E_STATUS_FORBIDDEN`

**可以直接忽略。** `wsl --update` 只是检查有没有新版本，不影响已经装好的 WSL。错误码里的 `HTTP_E_*` 是 WinHTTP 的错误常量前缀，说明它走的是 WinHTTP 直连通道。

### 5. 容器里访问不到外部 API

容器走 Docker 的 NAT 再经 Windows 出去。如果代理是 TUN 模式一般能覆盖；如果日志里出现连接超时，需要在 `Settings → Resources → Proxies` 里给守护进程配代理。

### 6. 磁盘只增不减

`ext4.vhdx` 删了镜像也不会自动缩小，时间长了能涨到几十 GB。

```powershell
docker system df             # 看占用
docker system prune -a       # 清理无用镜像/容器/网络
```

实在太大就在 Docker Desktop 设置里做一次磁盘清理。

### 7. Windows 上项目代码放在哪

**放在 WSL 的 Linux 文件系统里，不要放在 `/mnt/c/...`。** 跨文件系统读写性能差几十倍，`npm install`、Maven 构建这类大量小文件操作会慢到无法忍受。

```bash
mkdir -p ~/projects && cd ~/projects
```

想在资源管理器里看这些文件，地址栏输入 `\\wsl$\Ubuntu\home\<用户名>\projects`。

### 8. 容器里的数据会不会丢

会。不挂 volume 的话容器删了数据就没了。本文的 compose 用了命名卷 `pgdata`，`docker compose down` 不会删数据，**加 `-v` 才会删**。

备份：

```powershell
docker exec pgvector pg_dump -U postgres vectordb > backup.sql
```

### 9. `.wslconfig` 不生效

检查两点：它必须是个**文件**不是文件夹（`Get-Item` 的 `Attributes` 里不能有 `Directory`）；必须放在 **Windows 用户目录**下（`C:\Users\<用户名>\`），放进 WSL 里的 `/home/xxx/` 是不起作用的。改完执行 `wsl --shutdown`。

---

## 九、附录：本机实测记录

以下是在一台 Windows 10 Pro 机器上实际观测到的环境和踩到的坑，供对照排查。

### 环境

| 项目 | 值 |
|---|---|
| 系统 | Windows 10 Pro 22H2，内部版本 19045.6466 |
| 虚拟化 | 已启用 |
| WSL | 3.0.1.0，内核 6.18.40.1-1 |
| Docker Desktop | 4.95.0 |
| Docker CLI | 29.8.2，路径 `C:\Program Files\Docker\Docker\resources\bin` |

### 遇到并解决的问题

| 现象 | 根因 | 解决 |
|---|---|---|
| `wsl --install` 卡在「正在安装: Ubuntu」0.0% | 进度条不刷新，实际在后台解包 | 查 `ext4.vhdx` 大小确认真实进度 |
| `wsl --update` 报 403 | WinHTTP 通道直连，代理只配在 WinINET 层 | 可忽略；要修就 `netsh winhttp set proxy` |
| `docker run hello-world` 报 `pull access denied` | 匿名拉取被限流 | `docker login` |
| 5432 绑定失败 `access permissions` | 保留端口段 `5341-5440` 包含 5432；动态端口范围被改成从 1024 开始 | 换用 15432 |
| `.wslconfig` 不生效 | 路径下是**文件夹**不是文件 | 删掉重建为文件 |

### 本机端口保留段（供参考）

```
5041  - 5140
5141  - 5240
5241  - 5340
5341  - 5440     ← 5432 落在这一段
50000 - 50059

动态端口范围：起始 1024，数量 13977（默认应为 49152 / 16384）
```

---

## 附：一条龙命令速查

```powershell
# 1. 启用 WSL2（管理员，执行后重启）
wsl --install

# 2. 重启后验证
wsl --version
wsl -l -v

# 3. 安装 Docker Desktop
winget install Docker.DockerDesktop

# 4. 新终端里验证
docker version
docker run hello-world

# 5. 部署 pgvector
cd E:\docker\pgvector
docker compose up -d
docker exec -it pgvector psql -U postgres -d vectordb -c "SELECT extname, extversion FROM pg_extension;"

# 6. 日常运维
docker compose ps
docker compose logs -f pgvector
docker compose down          # 停止（保留数据）
docker exec -it pgvector psql -U postgres -d vectordb
```
