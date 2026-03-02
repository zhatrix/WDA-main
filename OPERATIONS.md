# WDA 运营说明

## 架构概述

WDA（Warehouse Design Automation）由 4 个 Docker 服务组成：

| 服务 | 容器名 | 端口 | 说明 |
|------|--------|------|------|
| nginx | `cyborg-nginx` | 38088 | 反向代理 + 静态文件服务 |
| web-server | `cyborg-server` | 8008 | Python 后端 API |
| dwg2dxf | `cyborg-dwg2dxf` | 8001 | DWG→DXF 文件转换服务 |
| PostgreSQL | `shucheng-layout2-db` | 15434 | 数据库 |

---

## 启动与停止

### 一键启动所有服务

```bash
cd /Users/zenyu/Workspace/WDA-main
bash run.sh nginx
```

> `nginx` 参数表示同时启动 nginx 容器。省略则只启动数据库、dwg2dxf、web-server。

### 停止所有服务

```bash
# 停止 nginx
docker-compose -f nginx/docker-compose.yml down

# 停止 web-server
docker-compose -f web-server/docker/docker-compose.yml down

# 停止 dwg2dxf
docker-compose -f dwg2dxf/docker/docker-compose.yml down

# 停止数据库
docker-compose -f database/docker-compose.pg.yml down
```

### 重启单个服务

```bash
docker restart cyborg-server     # web-server
docker restart cyborg-dwg2dxf   # dwg2dxf
docker restart cyborg-nginx      # nginx
docker restart shucheng-layout2-db  # 数据库
```

---

## 访问应用

| 项目 | 值 |
|------|----|
| 访问地址 | http://localhost:38088 |
| 用户名 | `shucheng` |
| 密码 | `Shucheng@2021Shanghai` |

> **注意：** nginx 启用了 HTTP Basic Auth，直接访问会弹出登录框。

---

## 日志查看

```bash
# web-server 日志（实时）
docker logs -f cyborg-server

# dwg2dxf 日志
docker logs -f cyborg-dwg2dxf

# nginx 访问日志
docker logs -f cyborg-nginx

# 数据库日志
docker logs -f shucheng-layout2-db
```

---

## 容器状态检查

```bash
# 查看所有 WDA 相关容器状态
docker ps --filter "name=cyborg" --filter "name=shucheng"

# 所有应运行的容器：
# cyborg-nginx       Up
# cyborg-server      Up
# cyborg-dwg2dxf     Up
# shucheng-layout2-db Up
```

---

## 数据库

| 项目 | 值 |
|------|----|
| 类型 | PostgreSQL 10.15 |
| 宿主机端口 | 15434 |
| 数据库名 | `shucheng-layout2` |
| 用户名 | `admin` |
| 密码 | `admin` |
| 数据目录 | `database/db_data/layout2_database/` |

```bash
# 连接数据库
psql -h 127.0.0.1 -p 15434 -U admin -d shucheng-layout2

# 或进入容器
docker exec -it shucheng-layout2-db psql -U admin -d shucheng-layout2
```

**数据备份：**
```bash
docker exec shucheng-layout2-db pg_dump -U admin shucheng-layout2 > backup_$(date +%Y%m%d).sql
```

---

## 修改登录密码

nginx 使用 htpasswd 认证，密码文件位于 `nginx/htpasswd`。

```bash
# 修改密码（替换 shucheng 为用户名，NewPassword 为新密码）
docker run --rm httpd:2.4 htpasswd -nb shucheng NewPassword > /Users/zenyu/Workspace/WDA-main/nginx/htpasswd

# 重载 nginx 使其生效（无需重启）
docker exec cyborg-nginx nginx -s reload
```

---

## 首次构建（全流程）

全新环境部署时，需依次完成以下 5 步。已构建过则跳过对应步骤。

### Apple Silicon (M 系列芯片) 特别说明

`dwg2dxf`、`web-server`、`cad`、`core` 四个镜像均依赖 amd64 架构，在 M 系列芯片上构建时必须加 `--platform linux/amd64`。

---

### 步骤 1：构建 dwg2dxf 镜像

```bash
cd /Users/zenyu/Workspace/WDA-main/dwg2dxf/docker
docker build --platform linux/amd64 -f Dockerfile.cn -t cyborg/dwg2dxf:2.0 .
docker tag cyborg/dwg2dxf:2.0 cyborg/dwg2dxf:latest
```

### 步骤 2：构建 web-server 镜像

> 使用含重试逻辑的 `Dockerfile.local`（应对阿里云镜像源网络抖动）。

```bash
cd /Users/zenyu/Workspace/WDA-main/web-server/docker
docker build --platform linux/amd64 -f Dockerfile.local -t cyborg/webserver:3.0 .
docker tag cyborg/webserver:3.0 cyborg/webserver:latest
```

### 步骤 3：编译 CAD 解析模块（cad_core.so）

```bash
# 3a. 构建编译环境镜像
cd /Users/zenyu/Workspace/WDA-main/cad/docker
docker build --platform linux/amd64 -f Dockerfile.cn -t cyborg/cad:0.0 .

# 3b. 编译 C++ 扩展（输出到 cad/build/lib.linux-x86_64-3.6/）
docker run --rm --platform linux/amd64 \
  -v /Users/zenyu/Workspace/WDA-main/cad:/src \
  cyborg/cad:0.0 \
  /bin/bash -c "cd /src && rm -rf build && python3 setup.py build"
```

验证：`cad/build/lib.linux-x86_64-3.6/cad_core.cpython-36m-x86_64-linux-gnu.so` 存在即成功。

### 步骤 4：编译布局算法模块（layout.so）

```bash
# 4a. 构建编译环境镜像（含 cmake）
cd /Users/zenyu/Workspace/WDA-main/core/docker
docker build --platform linux/amd64 -f Dockerfile.local -t cyborg/core:0.0 .

# 4b. 用 C++14 标准编译（输出到 core/）
docker run --rm --platform linux/amd64 \
  -v /Users/zenyu/Workspace/WDA-main/core:/src \
  cyborg/core:0.0 \
  /bin/bash -c "cd /src && rm -rf build && mkdir build && cd build && cmake -DCMAKE_CXX_STANDARD=14 .. && make"
```

验证：`core/layout.cpython-36m-x86_64-linux-gnu.so` 存在即成功。

> **注意**：必须用 `-DCMAKE_CXX_STANDARD=14`，C++11 下结构体默认成员初始化器语法不兼容。

### 步骤 5：构建前端

```bash
cd /Users/zenyu/Workspace/WDA-main/web/docker
docker run --rm \
  -v /Users/zenyu/Workspace/WDA-main/web:/web \
  -w /web cyborg/node14 \
  /bin/bash -c "cd app && npm install && npm run build"
```

验证：`web/dist/index.html` 存在即成功。

### 步骤 6：创建必要目录并启动

```bash
mkdir -p /Users/zenyu/Workspace/WDA-main/project/tmp/input
mkdir -p /Users/zenyu/Workspace/WDA-main/project/tmp/zip

cd /Users/zenyu/Workspace/WDA-main
bash run.sh nginx
```

---

## 重新构建镜像

> 仅在代码更新后需要重建时执行。

```bash
# 重建 dwg2dxf 镜像
cd /Users/zenyu/Workspace/WDA-main/dwg2dxf/docker
docker build --platform linux/amd64 -f Dockerfile.cn -t cyborg/dwg2dxf:2.0 .

# 重建 web-server 镜像
cd /Users/zenyu/Workspace/WDA-main/web-server/docker
docker build --platform linux/amd64 -f Dockerfile.local -t cyborg/webserver:3.0 .

# 重建后重启对应服务
docker-compose -f /Users/zenyu/Workspace/WDA-main/web-server/docker/docker-compose.yml up -d --force-recreate
```

---

## 目录结构

```
WDA-main/
├── nginx/           # nginx 配置、htpasswd 认证文件
├── database/        # PostgreSQL compose 文件
│   └── db_data/     # 数据库持久化数据（重要！勿删）
├── dwg2dxf/         # DWG 转换服务
├── web-server/      # Python 后端
│   ├── env/         # 环境变量配置
│   └── docker/      # Dockerfile 和 compose 文件
├── web/dist/        # 前端静态文件（2D）
├── 3d/dist/         # 前端静态文件（3D）
├── project/         # 用户上传文件临时目录
│   └── tmp/
│       ├── input/
│       └── zip/
└── run.sh           # 一键启动脚本
```

---

## 环境变量配置

配置文件：`web-server/env/default.sh`

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `ENV_PORT` | 8008 | 后端监听端口 |
| `ENV_DEBUG` | 0 | 调试模式（1=开启） |
| `ENV_DB_HOST` | 172.17.0.1 | 数据库宿主机 IP |
| `ENV_DB_PORT` | 15434 | 数据库端口 |
| `ENV_DB_DBNAME` | shucheng-layout2 | 数据库名称 |

修改后需重启 web-server 容器生效。

---

## 常见问题

**Q: 服务启动后访问 38088 返回 502**
A: 等待 `cyborg-server` 启动完成（约 10~30 秒），用 `docker logs -f cyborg-server` 确认后端就绪。

**Q: `project/tmp` 目录不存在导致启动报错**
```bash
mkdir -p /Users/zenyu/Workspace/WDA-main/project/tmp/input
mkdir -p /Users/zenyu/Workspace/WDA-main/project/tmp/zip
```

**Q: Docker 拉取镜像失败（EOF 错误）**
A: 通过 Docker Desktop GUI → Settings → Docker Engine 更新镜像源配置后点击 "Apply & Restart"。

**Q: Apple Silicon 上构建失败（架构不兼容）**
A: 所有 `docker build` 命令必须加 `--platform linux/amd64`。

**Q: `No module named 'cad_core'`**
A: CAD C++ 扩展未编译。执行"步骤 3"重新编译。

**Q: `No module named 'layout'`**
A: 布局算法 C++ 扩展未编译。执行"步骤 4"重新编译。注意必须使用 `-DCMAKE_CXX_STANDARD=14`。

**Q: 首页返回 `Internal Server Error`**
A: 前端未构建，`web/dist/` 目录不存在。执行"步骤 5"重新构建前端。
