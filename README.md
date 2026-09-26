# 视频平台 (Video Platform)

一个前后端分离的视频播放平台（仿 B 站风格），支持 4K 视频上传与播放，内置会员(VIP)体系、评论、动态、私聊、收藏与观看进度记忆。

- **后端**: Spring Boot 3.3 (Java 17) · Gradle 构建 · MyBatis-Plus · HikariCP 连接池 · SQLite(默认)/MySQL
- **前端**: **Expo (React Native) 通用应用** — 一套代码运行 Web / Android / iOS，react-native-web 渲染 Web
- **认证**: JWT(access+refresh) · BCrypt 加盐 · 图形验证码 · 手机号登录（主）
- **视频**: 分片/断点上传 · HTTP Range 流式播放(4K) · 本地文件播放 · m3u8 导入 · 可选 ffmpeg HLS 自适应
- **存储**: 可插拔 — 本地磁盘(兜底) / MinIO / RustFS(S3 兼容)，配置切换
- **可选组件**: Redis(验证码存储)、MySQL、MinIO/RustFS OSS — **未配置时自动关闭并回退**（验证码内存存储 + SQLite + 本地磁盘文件夹）

---

## 🔑 默认账号（好记版）

启动时自动创建（`platform.seed.enabled: true`），**登录一律使用手机号 + 密码 + 验证码**：

| 角色 | 手机号（登录用） | 用户名 | 密码 | 说明 |
|---|---|---|---|---|
| 管理员 ADMIN | **13800138000** | admin | 123456 | 可管理用户、赠送/收回 VIP、删任意评论/动态 |
| 普通用户 USER | **13900139000** | user | 123456 | 演示普通用户功能 |

> 用户名登录入口已在前端隐藏，但后端 `POST /api/auth/login`（用户名+验证码）仍保留可用。
> 正式环境请修改 `platform.seed.*` 或关闭 seed。

---

## 功能总览

| 模块 | 说明 |
|---|---|
| 📱 手机号登录/注册 | 注册必填手机号（11 位格式校验+唯一）与邮箱；登录主入口为手机号，用户名登录入口隐藏（后端保留） |
| 🧩 图形验证码 | 登录/注册均需验证码；默认内存存储，`platform.redis.enabled: true` 时切 Redis |
| 👑 会员(VIP)体系 | 用户角色 `ADMIN/USER`、`vipExpireAt` 到期时间；视频可标记 `vipOnly`，非会员访问播放页返回 4007 门槛页并引导开通 |
| 💳 VIP 充值页 | 月度 ¥18 / 季度 ¥45 / 年度 ¥148 三档套餐，当前为**模拟支付**（调用 `POST /api/users/me/vip` +1 个月），支付渠道预留 |
| 💬 评论系统 | 视频下方评论/删除；点击用户名弹出资料卡（用户名、VIP 徽章、VIP 渐变用户名、加入时间），可从资料卡直达私聊 |
| 📝 动态广场 | 所有人可浏览，登录后发布/删除自己的动态，管理员可删任意动态 |
| 💗 私聊 | 左侧会话列表 + 右侧聊天窗口，3 秒轮询新消息，未读角标，支持 `?to=userId` 直达 |
| ⭐ 收藏夹 | 收藏/取消收藏；**视频被删除后仍保留记录**，显示"视频已失效"卡片 |
| ⏱️ 观看进度记忆 | 每 5 秒上报进度，退出后再次进入自动恢复到上次位置（提示"已恢复到上次观看位置"） |
| 🌗 深色/亮色主题 | Navbar 右上角 🌙/☀️ 切换，跟随 `localStorage` 持久化 |
| 📹 本地播放/m3u8 | 播放页可"播放本地视频"（File 对象直读）或粘贴 m3u8/mp4 地址导入播放 |
| 📊 日志 | 控制台 + 文件 `./logs/video-platform.log` 双输出，登录手机号脱敏打印 |

---

## 目录结构

```
video-platform/
├── backend/                  # Spring Boot 后端 (Gradle)
│   ├── config/application.yml     ← 唯一的统一配置文件
│   └── src/main/java/com/videoplatform/
│       ├── common/                 R<T> · ResultCode · 异常 · 全局异常处理器
│       ├── config/                 MyBatis-Plus · CORS · 安全 · 静态资源 · 建表与迁移
│       ├── security/               JWT · 过滤器 · UserDetails · 验证码(内存/Redis 可插拔)
│       ├── module/auth/            注册/手机号登录/用户名登录(保留)/刷新
│       ├── module/user/            资料 · VIP 开通/管理 · 用户管理(ADMIN)
│       ├── module/video/           分片上传 · 列表 · 详情 · 删除 · 流媒体 · VIP 门槛
│       ├── module/interact/        评论 · 收藏 · 观看进度 · 动态 · 私聊
│       └── storage/                StorageService · 本地 · MinIO · RustFS · 工厂
├── frontend/                 # ★ Expo 通用前端：一套代码 → Web / Android / iOS
│   ├── App.tsx                    主题/认证 Provider + 导航（10 个屏幕）
│   ├── app.json / index.ts        Expo 配置与入口
│   └── src/
│       ├── api/                   axios 客户端(自动刷新) · auth · videos · users · social
│       ├── components/            TopBar(导航+主题切换) · Screen · VideoCard
│       │                          ProfileModal(资料卡) · VipUsername(渐变用户名) · CaptchaImage
│       ├── screens/               Home · Video(播放/评论/收藏/进度/本地播放/VIP门槛)
│       │                          Login(手机号) · Register(手机+邮箱) · Upload(vipOnly)
│       │                          Favorites · Posts · Chat · Vip · LocalPlay
│       ├── context/               AuthContext(AsyncStorage 持久化)
│       ├── theme.tsx              深色/亮色主题（渐变 VIP 用户名用 expo-linear-gradient）
│       └── config.ts              API 地址（web/native 分端配置）
├── legacy/                   # 历史版本备份（frontend-web = 旧 Vite React 版，可删除）
└── docs/design.md            # 设计说明
```

> 前端已合并为**单个 Expo 通用应用**：旧版 Vite React Web 应用与独立 mobile RN 应用分别备份在
> `legacy/frontend-web`（功能一致但已停止维护）。确认无误后可整目录删除 `legacy/`。

---

## 后端

### 环境要求
- JDK 17+
- Gradle 8.10.2（项目自带 wrapper；无系统 Gradle 也可运行）
- **零外部依赖即可启动**：默认 SQLite + 本地磁盘存储 + 内存验证码，无需安装任何数据库/中间件
- 可选：MySQL 8、Redis、MinIO/RustFS、ffmpeg/ffprobe（HLS）

### 启动

```bash
cd backend
./gradlew bootRun        # 默认 http://localhost:8080
```

### 唯一配置文件
所有配置都在 **`backend/config/application.yml`**：数据库、可选组件、存储、JWT、日志、视频/HLS、种子用户。

### 数据库：SQLite(默认) / MySQL 可选

- **默认 SQLite**（`platform.db.type: sqlite`）：零配置，建表/迁移/种子用户启动时自动完成，数据文件在 `./data/` 下。
- **切换 MySQL**：取消 `spring.datasource` MySQL 段注释（先建库 `CREATE DATABASE video_platform;`），并把 `platform.db.type` 改为 `mysql`。连接池为 HikariCP（可在 `spring.datasource.hikari` 调优）。
- 不配置 MySQL 时一切功能照常以 SQLite 兜底运行。

### 可选组件开关（未配置 = 自动关闭并回退）

| 组件 | 开关 | 未启用时行为 |
|---|---|---|
| Redis | `platform.redis.enabled: false`（默认） | 验证码存内存（重启失效，单机够用）；`true` 时需配 `spring.data.redis.*` |
| RustFS OSS | `platform.storage.type: local`（默认） | 视频存本地文件夹 `storage.local.base-path`；`rustfs` 时按 `platform.storage.rustfs.*` 走 S3 兼容接口 |
| MinIO | 同上 `type: minio` | 同上，按 `platform.storage.minio.*` |
| MySQL | `platform.db.type: sqlite`（默认） | SQLite 兜底 |

### 存储说明
`storage.type: local | minio | rustfs`
- `local`（兜底）：视频存 `./data/videos`，通过 `/storage/**` 与 Range 流接口提供访问。
- `minio` / `rustfs`：S3 兼容对象存储，配置对应 endpoint/ak/sk/bucket/public-url。

### 日志
在 `config/application.yml` 的 `logging.level.*` 修改级别；日志输出到控制台与文件 `./logs/video-platform.log`。敏感信息（手机号）打印时脱敏为 `138****8000`。

### 主要接口

**认证**
| 方法 | 路径 | 说明 | 鉴权 |
|---|---|---|---|
| GET | `/api/auth/captcha` | 获取图形验证码 | 公开 |
| POST | `/api/auth/register` | 注册（username+password+**phone**+**email**+验证码） | 公开 |
| POST | `/api/auth/login/phone` | **手机号登录**（主入口，需验证码） | 公开 |
| POST | `/api/auth/login` | 用户名登录（旧入口，后端保留） | 公开 |
| POST | `/api/auth/refresh` | 刷新 access token | refresh token |

**视频**
| 方法 | 路径 | 说明 | 鉴权 |
|---|---|---|---|
| GET | `/api/videos/list` | 分页视频列表 | 公开 |
| GET | `/api/videos/{id}` | 视频详情（vipOnly 时校验会员） | 公开/会员 |
| GET | `/api/videos/{id}/stream` | HTTP Range 流式播放（vipOnly 校验） | 公开/会员 |
| POST | `/api/videos/upload/init` | 初始化分片上传 | JWT |
| POST | `/api/videos/upload/{taskId}/{index}` | 上传分片 | JWT |
| POST | `/api/videos/upload/{taskId}/complete` | 合并入库（`vipOnly` 参数标记会员视频） | JWT |
| DELETE | `/api/videos/{id}` | 删除视频 | JWT(作者/管理员) |

**互动（评论/收藏/进度/动态/私聊）**
| 方法 | 路径 | 说明 | 鉴权 |
|---|---|---|---|
| GET / POST | `/api/videos/{videoId}/comments` | 评论列表 / 发表评论 | 公开 / JWT |
| DELETE | `/api/comments/{id}` | 删除评论 | JWT(作者/管理员) |
| POST / GET | `/api/videos/{id}/favorite` | 收藏切换 / 是否已收藏 | JWT |
| GET | `/api/users/me/favorites` | 我的收藏（含已删除视频"失效"记录） | JWT |
| POST / GET | `/api/videos/{id}/progress` | 保存 / 读取观看进度 | JWT |
| GET / POST | `/api/posts` | 动态列表 / 发布动态 | 公开 / JWT |
| DELETE | `/api/posts/{id}` | 删除动态 | JWT(作者/管理员) |
| GET | `/api/messages/contacts` | 会话列表（含未读数） | JWT |
| GET | `/api/messages/with/{userId}` | 与某用户的私聊记录 | JWT |
| POST | `/api/messages` | 发送私信 | JWT |

**用户 / VIP**
| 方法 | 路径 | 说明 | 鉴权 |
|---|---|---|---|
| GET | `/api/users/me` | 当前用户（含 vip/role/phone） | JWT |
| GET | `/api/users/{id}/profile` | 用户资料卡（评论/动态弹窗用） | 公开 |
| POST | `/api/users/me/vip` | 开通/续费 VIP（模拟支付，+1 个月） | JWT |
| GET | `/api/users` | 用户列表 | ADMIN |
| POST | `/api/users/{id}/vip?months=n` | 赠送 VIP | ADMIN |
| DELETE | `/api/users/{id}/vip` | 收回 VIP | ADMIN |
| DELETE | `/api/users/{id}` | 删除用户 | ADMIN |

---

## 前端（Expo 通用应用，Web / Android / iOS 同源）

```bash
cd frontend
npm install

# ---- Web ----
npx expo start --web         # 开发预览（浏览器访问，API 走 http://localhost:8080）
npx expo export --platform web   # 产出 dist/ 静态站，可挂 Nginx 或任意静态服务器

# ---- Android / iOS ----
npx expo start               # 开发（可用 Expo Go 体验）
npx expo run:android         # Android 生产构建（需 Android SDK）
npx expo run:ios             # iOS 生产构建（需 macOS + Xcode）
```

- API 地址在 `frontend/src/config.ts`：Web 用 `http://localhost:8080`；Android 模拟器用 `http://10.0.2.2:8080`；真机改成电脑局域网 IP。
- 功能与旧 Web 版完全对齐：手机号登录（用户名入口隐藏保留）、注册（手机+邮箱+验证码）、首页 VIP 徽章、播放页（VIP 门槛/收藏/评论/资料卡/进度记忆/本地播放/m3u8 导入）、上传（vipOnly 勾选）、收藏夹（失效视频卡）、动态、私聊（轮询+未读角标）、VIP 三档充值页、深色/亮色主题切换（顶栏 🌙/☀️）。

---

## 播放说明

- **默认（Range/MP4）**：`GET /api/videos/{id}/stream` 支持 `Range`，浏览器原生播放 4K 并支持拖动。无需 ffmpeg。
- **HLS（可选）**：设置 `platform.video.hls.enabled: true`，上传完成后异步用 ffmpeg 生成 HLS（`./data/hls/{videoId}/index.m3u8`），前端检测到 `.m3u8` 自动使用 hls.js。HLS 目前仅支持本地存储。
- **m3u8 导入**：播放页粘贴任意 m3u8/mp4 地址即可导入播放（不占用本站存储）。

---

## 验证

- 后端编译：`cd backend && ./gradlew build -x test` ✅
- 前端类型检查：`cd frontend && npx tsc --noEmit` ✅
- 前端 Web 构建导出：`cd frontend && npx expo export --platform web` ✅（产物 dist/）
- 前端原生构建：`cd frontend && npx expo run:android` / `npx expo run:ios`（需本机 SDK）
