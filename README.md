# AVDB115

AVDB115 是一个可自托管的 STRM 媒体库服务。服务端扫描本地 `.strm` 和常见视频文件，识别番号，抓取元数据与海报，并通过 Web 页面提供媒体库、收藏、历史、续播、随机播放和 HLS 播放能力。

项目由 FastAPI 服务端和 Vue Web 前端组成，不依赖 Emby 或 Jellyfin。当前仓库尚未包含原生 Android 工程；移动端接入设计见 [Android 客户端方案](docs/ANDROID_CLIENT.md)。

## 文档入口

- [文档总览](docs/README.md)
- [安装与部署](docs/DEPLOYMENT.md)
- [使用说明](docs/USER_GUIDE.md)
- [架构设计](docs/ARCHITECTURE.md)
- [API 参考](docs/API.md)
- [开发指南](docs/DEVELOPMENT.md)
- [产品设计](docs/PRODUCT_DESIGN.md)
- [提交与版本规范](docs/COMMIT_CONVENTION.md)

## 核心能力

- **媒体扫描**：递归扫描一个或多个媒体目录，支持 `.strm`、`.mp4`、`.mkv`、`.ts`、`.m2ts` 等格式。
- **番号识别**：清洗文件名中的广告、站点、分辨率和字幕标记，识别 FC2、HEYZO、GETCHU、259LUXU、MKB、IBW、T28、R18 等常见格式。
- **多源刮削**：支持 JavBus、DMM/FANZA 和 AVDB，可生成媒体元数据、NFO、海报、演员头像和在线字幕。
- **媒体库**：支持关键词、媒体库、演员、标签筛选，分页、排序、批量删除和默认海报。
- **播放**：支持本地文件、HTTP(S)、115 pickcode 和 HLS；hls.js/Media3 类客户端可接入播放接口。
- **账号与进度**：首次创建单管理员账号，保护管理 API；记录播放进度、历史、收藏和自定义收藏列表。
- **STRM 生成**：浏览 115 目录，配置目录映射并增量生成 STRM 文件。
- **运维能力**：模块日志、ZIP 完整备份、自动备份、Docker 部署和健康检查。

## 技术架构

```text
浏览器 / 原生客户端（规划）
              |
              | HTTP(S)
              v
     FastAPI /api/v1 + Vue 静态资源
          |              |
          |              +-- SQLite（配置、媒体、账号、进度）
          |              +-- data（海报、头像、字幕、备份）
          |              +-- logs（按模块轮转日志）
          |
          +-- 媒体文件系统 / STRM
          +-- 115 网盘
          +-- JavBus / DMM / AVDB
          +-- FFmpeg（MKV、TS 等转 HLS）
```

## 环境要求

- Python 3.11 或 3.12
- Node.js 20+、pnpm 10+
- FFmpeg 6+；播放 MKV、TS、M2TS 等需要服务端转 HLS 时必须安装
- Windows、Linux 或 macOS
- 推荐直接运行在能访问媒体目录和 115 的宿主机上

## 快速开始

### Windows PowerShell

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install --upgrade pip
.\.venv\Scripts\python.exe -m pip install -e ".\backend[dev]"

Set-Location frontend
pnpm install
pnpm build
Set-Location ..

Copy-Item .env.example .env
.\start.ps1
```

访问 `http://127.0.0.1:9527`。

### Linux / macOS

```bash
python3 -m venv .venv
. .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -e './backend[dev]'

cd frontend
pnpm install
pnpm build
cd ..

cp .env.example .env
cd backend
../.venv/bin/python -m uvicorn app.main:app --host 0.0.0.0 --port 9527
```

### Docker

```powershell
.\scripts\docker-publish.ps1
docker compose up -d
```

默认端口为 `9527`。`./data` 和 `./logs` 会持久化到宿主机；媒体目录需要按照 [docker-compose.yml](docker-compose.yml) 中的示例自行挂载，并在配置中心添加对应扫描根目录。

## 首次使用

1. 浏览器打开服务端地址，创建第一个管理员账号。账号创建后，设置、扫描、刮削和媒体管理接口都需要登录。
2. 进入「配置」，添加媒体扫描根目录，并按需填写 115 Cookie、公网地址和 HTTP(S) 代理。
3. 进入「刮削」，选择来源。默认 JavBus 通常无需配置；DMM/FANZA 需要 API ID 和 Affiliate ID；AVDB 需要服务地址与 API Key。
4. 触发扫描或等待后台自动扫描。扫描完成后，已识别媒体会自动进入刮削队列。
5. 在「媒体库」搜索、筛选、查看详情和播放；播放进度会自动同步到服务端。

完整操作流程见 [使用说明](docs/USER_GUIDE.md)。

## 常用命令

```powershell
# 后端开发服务
.\.venv\Scripts\python.exe -m uvicorn app.main:app --app-dir backend --reload --host 0.0.0.0 --port 9527

# 前端开发服务
Set-Location frontend
pnpm dev
Set-Location ..

# 后端测试与语法检查
.\.venv\Scripts\python.exe -m pytest -q
.\.venv\Scripts\python.exe -m compileall -q backend\app

# 前端测试与构建
Set-Location frontend
pnpm test
pnpm build
Set-Location ..
```

开发服务页面为 `http://127.0.0.1:5173`，Vite 会把 `/api` 代理到 `http://127.0.0.1:9527`。

## 关键环境变量

| 变量 | 说明 | 默认值 |
| --- | --- | --- |
| `AVDB115_DATABASE_URL` | SQLAlchemy 数据库地址 | `sqlite:///./data/avdb115.db` |
| `AVDB115_DATA_DIR` | 数据目录，存放数据库、图片、字幕和备份 | `./data` |
| `AVDB115_LOG_DIR` | 日志目录 | `./logs` |
| `AVDB115_LIBRARY_ROOT` | 首次启动时的默认媒体根目录 | 用户目录下的 `AVDB115` |
| `AVDB115_PUBLIC_BASE_URL` | 生成 STRM 时使用的公网服务地址 | `http://127.0.0.1:9527` |
| `AVDB115_CORS_ORIGINS` | 允许跨域访问 API 的前端来源，逗号分隔 | `http://localhost:5173` |
| `AVDB115_SCHEDULER_ENABLED` | 是否启用后台自动扫描与刮削调度器 | `true` |

业务设置在 Web「配置」页中修改后会写入 SQLite，并优先于环境变量。完整说明见 [部署文档](docs/DEPLOYMENT.md#环境变量)。

## 目录结构

```text
backend/                 FastAPI、SQLAlchemy、业务服务与 pytest
frontend/                Vue 3、Vite、Pinia、hls.js
docs/                    项目文档
scripts/                 版本递增与 Docker 发布脚本
data/                    运行时数据，不纳入 Git
logs/                    运行时日志，不纳入 Git
docker-compose.yml       Docker Compose 部署
Dockerfile               多阶段镜像构建
start.ps1                Windows 单进程启动脚本
VERSION                  唯一版本号来源
```

## API 与接口文档

服务启动后可访问：

- Swagger UI：`http://127.0.0.1:9527/docs`
- OpenAPI JSON：`http://127.0.0.1:9527/openapi.json`
- 项目内 API 说明：[docs/API.md](docs/API.md)

除健康检查、版本查询、登录相关接口和播放端点外，其余 `/api/v1` 接口在管理员账号创建完成后均需要登录 Cookie。

## 测试与验证

提交前至少执行：

```powershell
.\.venv\Scripts\python.exe -m pytest -q
.\.venv\Scripts\python.exe -m compileall -q backend\app

Set-Location frontend
pnpm test
pnpm build
```

文档本身由 `backend/tests/test_documentation.py` 校验必需文件和相对链接。

## 版本与提交

- `VERSION` 是唯一版本来源，格式固定为 `主版本.次版本.修订号.构建号`。
- 每次代码、配置或文档提交前运行 `.\.venv\Scripts\python.exe .\scripts\bump-version.py`。
- Git 提交信息必须使用中文。
- 详细规则见 [提交与版本规范](docs/COMMIT_CONVENTION.md)。

## 安全提示

- 首次部署后应立即创建管理员账号，不要把尚未初始化且可访问管理端口的实例暴露到公网。
- 公网部署建议使用 HTTPS 反向代理，不要直接暴露 SQLite、日志或媒体目录。
- 115 Cookie、刮削站点 Cookie 和 API Key 属于敏感信息，备份 ZIP 和 `.env` 不应提交到 Git 或公开分享。
- 播放 API 按当前设计为免登录端点，因为它需要兼容播放器直连、302 和 HLS 分片请求。不要在播放地址中写入长期有效凭据。
