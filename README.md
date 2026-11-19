# 黎明智捷办公

> 给自己磨的桌面办公工具 — AI 调度台 + 日历 + 专注计时 + 论坛

![favicon](favicon.png)

## ✨ 功能

| 模块 | 说明 |
|---|---|
| **🧠 AI 调度台** | DeepSeek / 豆包 / 千问 / Kimi / GLM / MiniMax 六个 AI 网页内嵌，独立 Cookie，免登录切换 |
| **📅 日历 & 待办** | 自绘三态日历（今日蓝圆 / 选中空心蓝环 / 待办淡黄点），待办卡片化列表，每日 23:55 自动清空昨日已完成待办 |
| **⏱ 专注计时器** | 番茄钟 25/5 预设 + 自定义时段，秒表 + 专注会话记录，环形进度条动画 |
| **💬 大论坛** | Neon PostgreSQL 云端论坛，发帖 / 评论 / 👍👎 / 图片上传，帖子卡片 delegate + 投票持久化状态 |

## 🛠 技术栈

- **UI 框架**：PySide6 6.x (Qt 6)
- **WebEngine**：QWebEngineView 内嵌 Chromium 渲染 AI 网页
- **本地存储**：SQLite (`data/app.db`) — 待办、专注会话
- **云端存储**：Neon PostgreSQL — 论坛用户 / 帖子 / 评论 / 投票
- **认证**：bcrypt 加密 + 本地 token.json
- **DPI**：Windows 高 DPI 缩放 + QtWebEngine 高清字体

## 🚀 快速开始

### 源码模式（开发/调试）

```bash
# 1. 安装依赖
pip install -r requirements.txt

# 2. 运行
# Windows：双击 启动.bat  (pythonw main.py 无黑窗)
# 或命令行：
python main.py
```

**源码启动耗时 ~600ms**（Windows 高 DPI + PySide6 冷启动）

### EXE 模式（分发）

```bash
# 双击 打包.bat 一键打包
# 产物在 release/LimingOffice/ 目录
# 双击 LimingOffice.exe 即可运行
```

## 📁 项目结构

```
黎明智捷办公/
├── main.py                    入口（DPI 设置 + MainWindow）
├── 启动.bat                   pythonw main.py 无黑窗
├── 打包.bat                   PyInstaller onedir + 瘦身自动化
├── config.json                应用配置（AI 模型 / 专注预设 / Neon 连接串）
├── requirements.txt
├── .gitignore
├── favicon.ico · favicon.png  应用图标
│
├── core/                      业务核心（UI 无关）
│   ├── config.py              配置加载
│   ├── db.py                  SQLite 封装（待办 / 专注会话）
│   ├── evaluator.py           每日评测
│   └── filter.py              论坛违禁词 SensitiveWordFilter
│
├── forum/                     论坛后端
│   ├── neon_client.py         Neon PostgreSQL 客户端（单例 + psycopg2）
│   └── auth.py                bcrypt 注册 / 登录 / token 验证
│
├── ui/                        界面层（全部 PySide6）
│   ├── main_window.py         主窗口（侧边栏 + QStackedWidget）
│   ├── ai_dock.py             AI 调度台（6 个 QWebEngineView 懒加载）
│   ├── calendar.py            日历 Tab（自绘 _HiCal paintCell + _TodoDelegate）
│   ├── forum.py               论坛 Tab（_BackendAdapter + _PostDelegate + 投票）
│   ├── auth_gate.py           登录门禁（全屏遮罩 + 注册/登录卡片）
│   ├── focus_timer.py         专注计时器 Tab + 底部专注控制栏
│   ├── animations.py          AnimationHelper（entrance / ripple / toast）
│   ├── tokens.py              视觉 Token 单一来源（颜色 / 间距）
│   └── dialogs.py             ConfirmDialog（圆角卡片确认框）
│
├── resources/theme.qss        全局 QSS 主题
├── data/                      运行时数据（自动创建）
│   ├── app.db                 SQLite 本地数据库
│   ├── images/                论坛上传的本地图片
│   └── cookies/               6 个 AI 独立 Cookie Jar
└── release/LimingOffice/      打包产物
```

## ⚙️ 配置

编辑 `config.json`：

| 字段 | 说明 |
|---|---|
| `ai_models` | AI 模型列表 — name / url / cookie_path |
| `focus_presets` | 专注预设 — work_min / break_min |
| `evaluator_weights` | 每日评测权重 — todo_weight / focus_weight |
| `forum.neon_database_url` | Neon PostgreSQL 连接串 |

## 🗄 数据库

**本地 SQLite** (`data/app.db`) — 单机数据：

| 表 | 字段 |
|---|---|
| `todos` | id, title, description, priority, status, due_date, created_at |
| `focus_sessions` | id, start_time, end_time, duration_sec, task_name |
| `focus_tasks` | id, name, target_min, created_at |

**云端 Neon PostgreSQL** — 论坛数据：

| 表 | 字段 |
|---|---|
| `users` | id, username, password_hash, created_at |
| `posts` | id, user_id, title, content, media_urls, likes, dislikes, created_at |
| `comments` | id, post_id, user_id, content, created_at |
| `votes` | user_id, post_id, vote_type (1/-1), PRIMARY KEY (user_id, post_id) |
| `filter_logs` | id, word, matched_text, created_at |

## 🔑 技术细节

### 启动性能

源码冷启动 **~600ms**（从之前的 25s + 优化）：

- ForumTab **懒加载**：切到论坛 tab 才初始化 Neon 连接 + 拉帖子列表
- AuthGate **不连 DB 探测**：启动时只检查 config 里有没有 URL，不连新加坡节点
- QWebEngineProfile **6 个独立实例**：每个 AI 一个 Cookie Jar，懒加载（切到 tab 才 load URL）

### 日历三态（自绘 paintCell）

```
今日:  蓝色实心圆 #2563eb + 白字  ← 优先级最高
选中:  空心 2px 蓝环 #2563eb + 深色字
待办:  右下角淡黄小圆点
```

### 论坛投票（持久化状态）

- `votes` 表唯一约束 `(user_id, post_id)` — 一人一票
- `vote_type=1` 点赞 / `-1` 点踩 / 同票再点 toggle off
- 帖子列表初始化时拉 `get_user_votes()` 缓存
- delegate 渲染时根据缓存给已投票按钮加**同色边框 + 加深背景**

### 每日打烊自动清理

- **23:55** Toast + 弹窗警告「还有 5 分钟打烊」
- **00:05** 自动清空昨日之前的：
  - ✅ 已完成的待办 (`status=1 AND due_date < 昨日`)
  - ✅ 昨日之前的专注会话 + `focus_tasks`
  - ❌ **未完成的待办不动**（不丢活）

### 打包瘦身（onedir）

PyInstaller onedir 模式（比 onefile 快一倍，因为 onefile 每次启动要解压 Chromium 到 %temp%）：

| 砍的 | 原大小 |
|---|---|
| `qml/` 整个目录 | -18.6 MB |
| `translations/` 只留 zh_CN + en | -52 MB |
| `plugins/` 只留 platforms + imageformats + styles | -4 MB |
| Qt3D / QtMultimedia / QtCharts / QtQuick3D 等 exclude | -18 MB |
| **合计砍 ~92 MB** | |

**最终 ~534 MB**（主要是 Chromium runtime，硬成本）

## ⚠️ 注意事项

1. **不要删 `PySide6.QtGui`** — 它是 PySide6 根基，排除整个 GUI 就崩
2. **不要动 `theme.qss` 里的 `transition` / `transform`** — 会导致 repaint 风暴卡死
3. **不要碰 `ForumTab.__init__` 里加 DB 调用** — 必须保持懒加载，否则启动卡
4. **QWebEngineView 上面不要叠带 `white background` 的 Qt Widget** — 会挡事件不能打字
5. **所有子侧边栏指示器用 `AnimationHelper.side_indicator_move`** — 防 QTimer GC

## 📝 License

自用软件，All Rights Reserved.
