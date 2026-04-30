# Secretary Memory - 秘书式多分区记忆系统 v3.0

![封面](./secretary-memory-cover.png)

仿生现代秘书的多笔记本分工模式，将记忆分区管理，各司其职。

## 五大核心改进

| 改进 | 模块 | 触发时机 |
|------|------|---------|
| 1. 容量管理 | `capacity_manager.py` | 每次 consolidation 时自动运行 |
| 2. FTS5 + LLM 摘要 | `fts5_index.py`, `session_search.py` | 搜索时或会话开始 |
| 3. 跨会话召回 | `auto_loader.py` | OpenClaw 会话开始时 |
| 4. 用户建模 | `user_model.py` | 每次会话结束时增量 |
| 5. 自动 Skill 生成 | `skill-creator/` | 同一问题出现 ≥3 次 |

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
memory/                              # 根目录
├── daily/                          # 每日日志（7天内）
│   └── 2026-04-30.md
├── archive/                        # 归档（7天前）
│   ├── daily/                     # 归档日志
│   ├── projects/                  # 已结束项目
│   ├── decisions/                 # 历史决策
│   ├── index.json                # 话题索引
│   └── .restore_log.json          # 归档操作日志
├── agenda/                         # 待办
│   ├── today/
│   ├── this-week/
│   └── follow-ups/
├── profile/                         # 用户偏好
│   ├── preferences/               # 沟通风格、技术背景等
│   ├── habits/                   # 工作习惯偏好
│   └── contacts/                 # 联系人信息
├── projects/                       # 进行中项目
├── knowledge/                      # 知识沉淀
│   ├── tech/
│   └── domain/
├── scripts/                         # 脚本目录
│   ├── session_summary.py         # 会话摘要
│   ├── context_loader.py          # 上下文加载
│   ├── memory_search.py           # 多分区搜索
│   ├── consolidate.py             # 归档脚本 (集成容量管理)
│   ├── capacity_manager.py        # [NEW] 容量管理
│   ├── fts5_index.py             # [NEW] FTS5 全文搜索
│   ├── session_search.py          # [NEW] 统一搜索入口
│   ├── auto_loader.py             # [NEW] 跨会话召回
│   ├── user_model.py              # [NEW] 用户关系图谱
│   ├── profile_miner.py           # 偏好提取 (集成用户图谱)
│   ├── conflict_detector.py       # 冲突检测
│   ├── restore.py                 # 恢复/回溯
│   ├── migrate.py                 # 迁移脚本
│   ├── realtime_monitor.py         # 实时监控
│   ├── skill-creator/             # [NEW] 自动 Skill 生成
│   │   ├── SKILL.md
│   │   └── auto_skill_generator.py
└── references/
    └── SPEC.md
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

## 容量管理 (capacity_manager.py)

自动检测 memory 目录容量，防止爆满。

```bash
# 检查容量（每次 consolidation 自动运行）
python3 capacity_manager.py --check

# 详细输出
python3 capacity_manager.py --verbose --warn

# 仅检查不执行
python3 consolidate.py --check-capacity

# 自动归档触发检查
python3 capacity_manager.py --auto
```

**阈值配置：**
| 指标 | 警告线 | 临界值 |
|------|--------|--------|
| MEMORY.md 行数 | 80% (400行) | 95% (475行) |
| daily/ 文件数 | 80% (24个) | 95% (28个) |
| archive/ 大小 | 80% (80MB) | 95% (95MB) |

## FTS5 + LLM 摘要 (fts5_index.py, session_search.py)

基于 SQLite FTS5 的快速全文搜索，配合 LLM 总结结果。

```bash
# 构建 FTS5 索引（首次使用）
python3 session_search.py --build-index

# FTS5 + LLM 摘要搜索
python3 session_search.py "项目X 架构设计"

# 纯 FTS5（无 LLM）
python3 session_search.py "项目X" --no-llm

# 传统分区搜索
python3 session_search.py "项目X" --classic

# 混合搜索（默认）
python3 session_search.py "项目X"

# 会话开始时召回
python3 session_search.py "项目X" --context

# 查看索引统计
python3 session_search.py --stats

# 优化数据库
python3 fts5_index.py --vacuum
```

**搜索策略：**
- `fts5`: 仅 FTS5 全文搜索（BM25 排序）
- `classic`: 传统分区搜索
- `hybrid`: FTS5 + 经典搜索合并（默认）

## 跨会话召回 (auto_loader.py)

会话开始时自动加载相关记忆，按优先级排序。

```bash
# CLI 用法
python3 auto_loader.py --topic "项目X 架构"
python3 auto_loader.py --topic "项目X" --inject   # 输出注入格式
python3 auto_loader.py --stats                    # 查看加载状态

# Hook 用法
python3 auto_loader.py --hook-start --session-id {session_id} --topic {topic}
```

**优先级算法：**
```
综合得分 = 相关性 × 分区权重 × 时间加权

分区权重:
  agenda     → 2.0  (最高)
  projects   → 1.8
  profile    → 1.5
  knowledge  → 1.2
  daily      → 1.0
  archive    → 0.5  (最低)

时间加权:
  7天内 → 1.5x
  7天后 → 指数衰减，最少 0.3
```

## 用户建模 (user_model.py)

从会话中提取实体和关系，构建用户知识图谱。

```bash
# 构建图谱
python3 user_model.py --build

# 增量更新
python3 user_model.py --update

# 图谱统计
python3 user_model.py --stats

# 按话题查询
python3 user_model.py --query "项目X"

# 获取实体局部图谱
python3 user_model.py --entity "Python" --depth 2

# 清理低频实体
python3 user_model.py --vacuum
```

**实体类型：**
| 类型 | 权重 | 示例 |
|------|------|------|
| user | 2.0 | 用户、客户 |
| project | 1.8 | 项目、系统 |
| technology | 1.5 | Python、React |
| person | 1.3 | 开发者、工程师 |
| preference | 1.2 | 喜欢简洁回答 |
| concept | 1.0 | 架构、设计 |

**关系类型：** `works_on`, `uses`, `manages`, `interested_in`, `collaborates_with`

**数据文件：**
- 图谱：`memory/.user_graph.json`
- 统计：`memory/.user_graph_stats.json`

## 自动 Skill 生成 (skill-creator/)

检测复杂/重复任务，自动生成新 Skill 封装解决方案。

```bash
# 检测复杂任务（扫描 daily/ 历史）
python3 skill-creator/auto_skill_generator.py --detect

# 生成新 Skill
python3 skill-creator/auto_skill_generator.py --generate \
    --name "task-tracker" \
    --pattern "task_tracking" \
    --description "自动化任务追踪" \
    --triggers "任务,todo,追踪" \
    --steps "创建任务" "更新状态" "生成报告"

# 注册触发词
python3 skill-creator/auto_skill_generator.py --register \
    --skill "task-tracker" --triggers "任务,todo"

# 查看统计
python3 skill-creator/auto_skill_generator.py --stats

# 列出已生成的 Skills
python3 skill-creator/auto_skill_generator.py --list

# 自我改进
python3 skill-creator/auto_skill_generator.py --improve
```

**内置复杂模式识别：**
| 模式 | 关键词 | 建议名称 |
|------|--------|----------|
| task_tracking | 任务、todo、追踪、待办 | task-tracker |
| code_review | 代码审查、review、PR | code-review |
| bug_triage | bug、修复、问题 | bug-triage |
| meeting_notes | 会议、meeting、纪要 | meeting-notes |
| data_analysis | 分析、数据、报表 | data-analysis |
| doc_generation | 文档、doc、生成 | doc-generator |
| test_generation | 测试、test、用例 | test-generator |
| deployment | 部署、deploy、发布 | deploy-helper |
| api_design | API、接口、rest | api-designer |
| db_schema | 数据库、schema、表结构 | db-schema-manager |

## 原有脚本

### 会话摘要 session_summary.py

```bash
python3 session_summary.py --session "今天讨论了..." --topics "项目X, 决策Y"
python3 session_summary.py --session "..." --dry-run  # 预览
python3 session_summary.py --watch                    # 实时监控追加
python3 session_summary.py --watch --session-id {session_id}
```

### 上下文加载 context_loader.py

```bash
python3 context_loader.py "项目X 设计方案"
python3 context_loader.py "项目X" --quiet
python3 context_loader.py "项目X" --format prompt
python3 context_loader.py "项目X" --limit 5
```

### 偏好提取 profile_miner.py

```bash
python3 profile_miner.py --session "用户说：喜欢简洁的回答"
python3 profile_miner.py --file /path/to/session.log
python3 profile_miner.py --list
```

### 冲突检测 conflict_detector.py

```bash
python3 conflict_detector.py --new "决定用方案A" --topic "项目X架构"
python3 conflict_detector.py --new-file /path/to/content.md --topic "架构设计"
```

### 恢复/回溯 restore.py

```bash
python3 restore.py --list
python3 restore.py --date 2026-04-23
python3 restore.py --topic "项目X"
python3 restore.py --dry-run
```

### 归档 consolidation

```bash
python3 consolidate.py --verbose
python3 consolidate.py --dry-run
python3 consolidate.py --check-capacity  # 仅检查容量
```

### 多分区搜索 memory_search.py

```bash
python3 memory_search.py "用户偏好"
python3 memory_search.py "GUI Agent" --deep
python3 memory_search.py "项目" -p profile,projects
python3 memory_search.py "项目" --per-partition-limit 5
```

### 实时监控 realtime_monitor.py

```bash
python3 realtime_monitor.py --daemon          # 事件驱动（推荐）
python3 realtime_monitor.py --daemon --poll    # 降级轮询
python3 realtime_monitor.py --once            # 单次检查
python3 realtime_monitor.py --status          # 查看状态
python3 realtime_monitor.py --test            # 测试模式
```

## Hooks 配置（v3.0 推荐）

```json
{
  "hooks": {
    "session:start": "python3 <path>/auto_loader.py --hook-start --session-id {session_id} --topic {topic}",
    "session:end": "python3 <path>/session_summary.py --session-id {session_id} && python3 <path>/profile_miner.py --session-id {session_id}",
    "cron": "0 18 * * * python3 <path>/consolidate.py --verbose"
  }
}
```

注意：
- `session:start` hook 在新会话开始时触发，启动跨会话召回
- `session:end` hook 在每次会话结束时触发，自动生成会话摘要 + 提取偏好
- `cron` 需要系统支持定时任务，建议使用 Claude Code 的 scheduled_tasks
- 建议使用绝对路径指向 `secretary-memory/scripts/` 目录

## 自动运作流程 (v3.0)

```
会话开始
    ↓
auto_loader.py 加载相关记忆 (FTS5 + 优先级排序)
    ↓
用户交互...
    ↓
会话结束
    ↓
session_summary.py 生成摘要 → daily/
    ↓
profile_miner.py 提取偏好 + user_model.py 更新图谱
    ↓
(定时 18:00)
    ↓
consolidate.py 归档 (自动检查容量)
    ↓
capacity_manager.py 容量警告 (≥80%)
    ↓
容量临界 (≥95%) → 自动触发更多归档
```

## 搜索优先级策略

- **常规搜索**: profile > agenda > projects > knowledge > daily > archive
- **深度搜索 (--deep)**: 包含 archive/
- **仅归档搜索 (--archive-only)**: 仅查 archive/
- **FTS5 搜索**: BM25 排序 + LLM 摘要

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

## 参考文档

详细规格说明见 `references/SPEC.md`
