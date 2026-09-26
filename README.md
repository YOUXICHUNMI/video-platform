# 视频平台 (Video Platform)

一个前后端分离的视频播放平台（仿 B 站风格），支持 4K 视频上传与播放。

- **后端**: Spring Boot 3.3 (Java 17) · Gradle 构建 · MyBatis-Plus · MySQL(默认)/SQLite
- **前端**: React 18 + TypeScript + Vite · 手写暗色主题 · hls.js
- **认证**: JWT(access+refresh) · BCrypt 加盐加密 · 图形验证码
- **视频**: 分片/断点上传 · HTTP Range 流式播放(4K) · 可选 ffmpeg HLS 自适应
- **存储**: 可插拔 — 本地磁盘 / MinIO(S3 兼容)，配置切换
- **工程化**: 统一响应 `R<T>` · 全局异常捕获 · 配置化日志级别 · 统一配置文件

---

## 目录结构

```
video-platform/
├── backend/                  # Spring Boot 后端 (Gradle)
│   ├── config/application.yml     ← 唯一的统一配置文件
│   └── src/main/java/com/videoplatform/
│       ├── common/                 R<T> · ResultCode · 异常 · 全局异常处理器
│       ├── config/                 MyBatis-Plus · CORS · 安全 · 静态资源 · 建表
│       ├── security/               JWT · 过滤器 · UserDetails · 验证码
│       ├── module/auth/            注册/登录/刷新/验证码
│       ├── module/user/
│       ├── module/video/           分片上传 · 列表 · 详情 · 删除 · 流媒体
│       └── storage/                StorageService · 本地 · MinIO · 工厂
├── frontend/                 # React 前端 (Vite)
│   └── src/
│       ├── api/                    axios 客户端 · auth · videos
│       ├── components/             Navbar · VideoCard · Player
│       ├── pages/                  Home · Video · Upload · Login · Register
│       ├── context/                AuthContext
│       └── styles/global.css       暗色主题
└── docs/design.md             # 设计说明
```

---

## 后端

### 环境要求
- JDK 17+
- Gradle 8.10.2（项目自带 wrapper；无系统 Gradle 也可运行）
- 数据库：MySQL 8（默认）或 SQLite（备用）
- 可选：ffmpeg/ffprobe（用于 HLS 转码与媒体信息探测）

### 唯一配置文件
所有配置都在 **`backend/config/application.yml`**：数据库、存储、JWT、日志级别、视频/HLS。

```bash
cd backend
# 生成 wrapper（首次）并启动
./gradlew bootRun
```

> 本机没有 gradle 时，项目内的 `gradlew` 会下载 Gradle 发行版。若首次下载较慢，可先用已安装的 Gradle 生成 wrapper：
> `gradle wrapper --gradle-version 8.10.2`

### 数据库切换
`config/application.yml` 中：
- 默认 `platform.db.type: mysql`，使用 `spring.datasource` 的 MySQL 配置。**MySQL 需先建库**：`CREATE DATABASE video_platform;`
- 切换 SQLite：注释掉 MySQL 的 `spring.datasource`，启用 SQLite 的 datasource，并把 `platform.db.type` 改为 `sqlite`。建表与种子管理员在启动时自动完成。

### 默认管理员
`platform.seed.enabled: true` 时启动自动创建管理员：
- 用户名 `admin` · 密码 `admin123`

> 正式环境请修改 `platform.seed.admin-username/admin-password` 或关闭 seed。

### 存储切换
`storage.type: local | minio`。
- `local`：视频存到 `storage.local.base-path`（默认 `./data/videos`），通过 `/storage/**` 与 Range 流接口提供访问。
- `minio`：S3 兼容对象存储，配置 `storage.minio.*`。

### 日志
在 `config/application.yml` 的 `logging.level.*` 修改级别（如 `root: DEBUG`、`com.videoplatform: INFO`），单点改动即可；日志同时输出到控制台与文件 `./logs/video-platform.log`。

### 主要接口

| 方法 | 路径 | 说明 | 鉴权 |
|---|---|---|---|
| GET | `/api/auth/captcha` | 获取图形验证码 | 公开 |
| POST | `/api/auth/register` | 注册（需验证码） | 公开 |
| POST | `/api/auth/login` | 登录（需验证码）→ JWT | 公开 |
| POST | `/api/auth/refresh` | 刷新 access token | refresh token |
| GET | `/api/videos/list` | 分页视频列表 | 公开 |
| GET | `/api/videos/{id}` | 视频详情 | 公开 |
| GET | `/api/videos/{id}/stream` | HTTP Range 流式播放 | 公开 |
| POST | `/api/videos/upload/init` | 初始化分片上传 | JWT |
| POST | `/api/videos/upload/{taskId}/{index}` | 上传分片 | JWT |
| POST | `/api/videos/upload/{taskId}/complete` | 合并分片并入库 | JWT |
| DELETE | `/api/videos/{id}` | 删除视频 | JWT |
| GET | `/api/users/me` | 当前用户 | JWT |

---

## 前端

```bash
cd frontend
npm install
npm run dev        # http://localhost:5173 （/api 代理到 :8080）
```

构建产物：
```bash
npm run build      # 输出到 dist/
```

页面：首页视频墙、视频播放页、分片上传页、登录 / 注册页（含验证码）。

---

## 播放说明

- **默认（Range/MP4）**：`GET /api/videos/{id}/stream` 支持 `Range`，浏览器原生播放 4K 并支持拖动。无需 ffmpeg。
- **HLS（可选）**：设置 `platform.video.hls.enabled: true`，上传完成后异步用 ffmpeg 生成 HLS（`./data/hls/{videoId}/index.m3u8`），前端在播放器检测到 `.m3u8` 时自动使用 hls.js。HLS 目前仅支持本地存储。

---

## 验证

- 后端编译：`cd backend && ./gradlew build`
- 前端构建：`cd frontend && npm run build`
