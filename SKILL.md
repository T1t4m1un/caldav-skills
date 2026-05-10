---
name: caldav
description: CalDAV 个人日程（VEVENT）和任务（VTODO）的 CRUD 操作。当用户提到日历、日程、待办、任务、caldav、Radicale、Nextcloud、iCloud，或说"加个日程""查下周安排""标记任务完成""创建提醒"时使用此 skill。
---

# CalDAV Skill

个人日程（VEVENT）和任务（VTODO）的 CalDAV CRUD 操作，遵循 RFC 4791 + RFC 5545。

## 凭据

source ~/.caldavrc 加载环境变量：
$CALDAV_URL, $CALDAV_USER, $CALDAV_PASS, $CALDAV_ROOT, $CALDAV_CALENDAR

## 子模块

根据操作类型用 Read 工具加载对应文件：

| 操作 | 加载 | 关键方法 |
|------|------|----------|
| 初始化 / 凭据问题 | init.md | source, PROPFIND, MKCOL/fallback |
| 日程增删改查 | events.md | PUT, REPORT, DELETE |
| 任务增删改查 | todos.md | PUT (VTODO), REPORT (VTODO), STATUS |
| 字段定义 / 模板 | reference.md | .ics template, 字段表, 踩坑 |

## 加载策略

1. 先 source ~/.caldavrc
2. 若缺失 → 仅加载 init.md
3. 凭据就绪后按操作加载对应 md
