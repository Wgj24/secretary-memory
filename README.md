# Secretary Memory - 秘书式多分区记忆系统

仿生现代秘书的多笔记本分工模式，将记忆分区管理，各司其职。

## 核心概念

将 AI 助手的记忆系统改造成类似现代秘书的多笔记本分工模式：

| 笔记本 | 目录 | 说明 |
|--------|------|------|
| 已完成 | `archive/` | 7天前的日志、已结束项目、历史决策 |
| 待办 | `agenda/` | 今日待办、本周计划、跟进事项 |
| 偏好 | `profile/` | 用户偏好、习惯、联系人 |
| 项目 | `projects/` | 进行中的项目 |
| 知识 | `knowledge/` | 沉淀的技术/领域知识 |
| 日志 | `daily/` | 7天内的每日日志（原始流水） |

## 目录结构

```
memory/                          # 根目录
├── daily/                      # 每日日志（7天内）
│   └── 2026-04-30.md
├── archive/                    # 归档（7天前）
│   ├── daily/                 # 归档日志
│   ├── projects/              # 已结束项目
│   ├── decisions/             # 历史决策
│   ├── index.json            # 话题索引
│   └── .restore_log.json     # 归档操作日志
├── agenda/                    # 待办
│   ├── today/
│   ├── this-week/
│   └── follow-ups/
├── profile/                    # 用户偏好
│   ├── preferences/           # 沟通风格、技术背景等
│   ├── habits/               # 工作习惯偏好
│   └── contacts/             # 联系人信息
├── projects/                   # 进行中项目
├── knowledge/                  # 知识沉淀
│   ├── tech/
│   └── domain/
├── scripts/                    # 脚本目录
│   ├── consolidate.py         # 归档脚本
│   ├── memory_search.py       # 搜索脚本
│   ├── session_summary.py     # 会话摘要脚本
│   ├── context_loader.py      # 上下文加载脚本
│   ├── profile_miner.py       # 偏好提取脚本
│   ├── conflict_detector.py   # 冲突检测脚本
│   ├── restore.py             # 恢复/回溯脚本
│   ├── migrate.py             # 迁移脚本
│   └── realtime_monitor.py    # 实时监控脚本
└── .monitor_state.json         # 监控状态文件
```

## 各分区职责

### daily/ — 每日日志
- **来源**: 每次会话自动追加
- **格式**: 时间戳 + 会话摘要 + 关键决策点
- **保留**: 7天

### archive/ — 历史档案
- **来源**: daily 定期 consolidation（7天后）
- **内容**: 已完成项目、已定决策、旧日记
- **保留**: 永久

### agenda/ — 待办事项
- **来源**: 用户指令 + 自动提取
- **内容**: today/this-week/follow-ups 三级
- **保留**: 完成即移走

### profile/ — 用户画像
- **来源**: 初始化 + 增量更新
- **内容**: 偏好、习惯、联系人
- **保留**: 只增不改

### projects/ — 项目跟踪
- **来源**: 用户提及项目时创建/更新
- **内容**: 项目状态、进度、关键节点
- **保留**: 结束移至 archive

### knowledge/ — 知识沉淀
- **来源**: 会话中提取的事实性知识
- **内容**: 技术笔记、领域概念
- **保留**: 持久

## 脚本使用

### 会话摘要 session_summary.py

在会话结束时自动生成摘要，写入 `daily/`。

```bash
# 生成摘要
python3 session_summary.py --session "今天讨论了..." --topics "项目X, 决策Y"

# 预览（不写入）
python3 session_summary.py --session "..." --topics "..." --dry-run

# JSON 格式输出
python3 session_summary.py --session "..." --topics "..." --json

# 持续监控模式：实时增量追加会话内容
python3 session_summary.py --watch

# 带会话ID的持续监控
python3 session_summary.py --watch --session-id {session_id}
```

### 上下文加载 context_loader.py

新会话开始时召回相关历史记忆（每分区 3 条）。

```bash
# 召回相关记忆
python3 context_loader.py "项目X 设计方案"

# 简洁模式（只输出关键摘要）
python3 context_loader.py "项目X" --quiet

# 输出可直接注入 prompt 的格式
python3 context_loader.py "项目X" --format prompt

# 每分区限制为 5 条
python3 context_loader.py "项目X" --limit 5
```

### 偏好提取 profile_miner.py

从会话内容中增量提取用户偏好，只增不改。

```bash
# 从会话内容提取偏好
python3 profile_miner.py --session "用户说：喜欢简洁的回答"

# 从文件读取内容
python3 profile_miner.py --file /path/to/session.log

# 预览将要提取的偏好（不写入）
python3 profile_miner.py --session "..." --dry-run

# 列出当前已提取的偏好
python3 profile_miner.py --list
```

### 冲突检测 conflict_detector.py

检测新记忆与已有记忆的矛盾，主动提示。

```bash
# 检测冲突
python3 conflict_detector.py --new "决定用方案A" --topic "项目X架构"

# 从文件读取新内容
python3 conflict_detector.py --new-file /path/to/content.md --topic "架构设计"
```

### 恢复/回溯 restore.py

支持按日期、话题、文件类型精细化恢复。

```bash
# 列出归档操作历史
python3 restore.py --list

# 列出所有已归档的文件
python3 restore.py --archives

# 恢复指定日期的归档
python3 restore.py --date 2026-04-23

# 恢复指定话题的归档
python3 restore.py --topic "项目X"

# 预览恢复（不实际执行）
python3 restore.py --date 2026-04-23 --dry-run
```

### 归档 consolidation

```bash
# 干跑测试（不实际修改）
python3 consolidate.py --dry-run

# 实际执行
python3 consolidate.py --verbose

# 查看归档操作历史
python3 consolidate.py --restore-log
```

归档流程：
1. 扫描 `daily/` 下超过7天的日志
2. 提取项目信息 → `archive/projects/`
3. 提取决策信息 → `archive/decisions/`
4. 移动日志 → `archive/daily/`
5. 更新 `memory.md` 精选摘要
6. 更新 `archive/index.json` 话题索引

### 多分区搜索

```bash
# 搜索所有分区（默认，每分区最多 3 条）
python3 memory_search.py "用户偏好"

# 深度搜索（包含 archive 历史归档）
python3 memory_search.py "GUI Agent" --deep

# 仅搜索历史归档
python3 memory_search.py "之前那个项目" --archive-only

# 指定分区
python3 memory_search.py "项目" -p profile,projects

# 每分区限制为 5 条
python3 memory_search.py "项目" --per-partition-limit 5
```

### 迁移旧结构

```bash
# 干跑测试
python3 migrate.py --dry-run

# 执行迁移
python3 migrate.py
```

### 实时增量监控 realtime_monitor.py

**v2 修复版**：解决写时移动冲突、轮询延迟、文件句柄问题。

```bash
# 事件驱动监控（推荐，自动降级到轮询）
python3 realtime_monitor.py --daemon

# 强制使用轮询模式
python3 realtime_monitor.py --daemon --poll

# 单次运行（立即检查后退出）
python3 realtime_monitor.py --once

# 查看监控状态
python3 realtime_monitor.py --status

# 测试模式
python3 realtime_monitor.py --test
```

**v2 修复内容**：

| 问题 | 解决方案 |
|------|---------|
| 写时移动冲突 | `safe_move_to_daily()` 等待文件写完才移动，使用 `fuser`/`lsof` 检测文件句柄 |
| 轮询延迟 | 支持 FSEvents (macOS) / inotify (Linux) 事件驱动，零延迟 |
| 文件句柄问题 | 移动后创建 symlink，OpenClaw 继续写到正确位置；内容同步合并 |

## 自动运作流程

### 传统模式（定时批处理）

```
写 memory/YYYY-MM-DD.md
        ↓ (定时移动，延迟)
移动到 memory/daily/YYYY-MM-DD.md
        ↓ (每天 18:00 cron: consolidation)
7天前 → archive/daily/
项目 → archive/projects/
决策 → archive/decisions/
关联 → archive/index.json
```

### 推荐模式（实时增量）

```
写 memory/YYYY-MM-DD.md
        ↓ (realtime_monitor 实时检测)
立即移动到 memory/daily/YYYY-MM-DD.md
        ↓ (session_summary --watch 持续追加)
实时增量写入会话内容
        ↓ (每天 18:00 cron: consolidation)
7天前 → archive/daily/
```

## 归档触发机制

| 触发条件 | 说明 |
|---------|------|
| 用户显式查询 | "之前那个项目怎么设计的"、"上次讨论的结论是" |
| `--deep` 搜索 | 包含 archive/ 的所有历史归档 |
| `--archive-only` | 仅搜索历史归档 |
| consolidation 时 | 自动建立话题索引，关联相关历史归档 |

## Consolidation 时机

- **每日定时**: 每天 18:00 Asia/Shanghai 自动执行
- **手动触发**: 用户要求整理记忆时
- **迁移触发**: 首次使用此技能时，运行 migrate.py

## Hooks 配置（自动化）

### 推荐配置 - 实时增量写入

```json
{
  "hooks": {
    "session:start": "python3 <path-to-scripts>/realtime_monitor.py --daemon --interval 3",
    "session:end": "python3 <path-to-scripts>/session_summary.py --session-id {session_id}",
    "cron": "0 18 * * * python3 <path-to-scripts>/consolidate.py --verbose"
  }
}
```

### 传统配置 - 定时批处理

```json
{
  "hooks": {
    "session:end": "python3 <path-to-scripts>/session_summary.py --session-id {session_id}",
    "cron": "0 0,6,12,18 * * * python3 <path-to-scripts>/realtime_monitor.py --once"
  }
}
```

注意：
- `session:start` hook 在新会话开始时触发，启动实时监控
- `session:end` hook 在每次会话结束时触发，自动生成会话摘要
- `cron` 需要系统支持定时任务，Claude Code 的 settings.json 不直接支持 cron，需使用外部定时器或 Claude Code 的 scheduled_tasks
- 建议使用绝对路径指向 `secretary-memory/scripts/` 目录

## 搜索优先级策略

- **常规搜索**: profile > agenda > projects > knowledge > daily > archive
- **深度搜索 (--deep)**: 包含 archive/
- **仅归档搜索 (--archive-only)**: 仅查 archive/
