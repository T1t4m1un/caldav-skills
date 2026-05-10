# CalDAV 参考

## 资源层级

```
服务器根 (CALDAV_URL)
  └── 用户 Principal (CALDAV_ROOT, 如 /alice/)     ← PROPFIND 入口
       ├── 日历本 (CALDAV_CALENDAR, 如 personal/)   ← 一个 collection
       │    ├── uuid-1.ics  (VEVENT)                 ← 日程
       │    └── uuid-2.ics  (VTODO)                  ← 任务
       └── 其他日历本 ...
```

## URL 构造

```bash
# 日历本 URL（REPORT/PROPFIND 目标）
CALENDAR_URL="${CALDAV_URL}${CALDAV_ROOT}/${CALDAV_CALENDAR}/"

# 事件/任务 URL（GET/PUT/DELETE 目标）
EVENT_URL="${CALENDAR_URL}${UID}.ics"
```

## 字段对照表

| 字段 | VEVENT | VTODO | 必填 | 格式 |
|------|--------|-------|------|------|
| UID | 全局唯一标识 | 全局唯一标识 | 是 | uuid/任意唯一字符串 |
| DTSTART | 开始时间 | 开始日期 | VEVENT 是 | `YYYYMMDD` 或 `YYYYMMDDTHHMMSS[Z]` |
| DTEND | 结束时间 | — | 否 | 同 DTSTART |
| DUE | — | 截止日期 | 否 | `YYYYMMDD` |
| SUMMARY | 标题 | 标题 | 是 | 文本 |
| DESCRIPTION | 描述 | 描述 | 否 | 文本 |
| LOCATION | 地点 | — | 否 | 文本 |
| PRIORITY | — | 1-9（1=最高） | 否 | 整数 |
| STATUS | — | `NEEDS-ACTION` / `COMPLETED` / `CANCELLED` | VTODO 建议 | 枚举 |
| COMPLETED | — | 完成时间 | 标记完成时 | `YYYYMMDDTHHMMSSZ` |
| DTSTAMP | 时间戳 | 时间戳 | 是 | `YYYYMMDDTHHMMSSZ` |
| CATEGORIES | 分类 | 分类 | 否 | 逗号分隔文本 |
| RRULE | 重复规则 | — | 否 | `FREQ=D/W/M/Y;INTERVAL=N` |
| VALARM | 提醒 | 提醒 | 否 | 子组件 |

## .ics 模板

### VEVENT（日程）

```
BEGIN:VCALENDAR
VERSION:2.0
PRODID:-//CalDAV Skill//EN
BEGIN:VEVENT
UID:{uuid}
DTSTART:{YYYYMMDDTHHMMSS}
DTEND:{YYYYMMDDTHHMMSS}
DTSTAMP:{now_utc}
SUMMARY:{标题}
DESCRIPTION:{描述}
LOCATION:{地点}
END:VEVENT
END:VCALENDAR
```

### VTODO（任务）

```
BEGIN:VCALENDAR
VERSION:2.0
PRODID:-//CalDAV Skill//EN
BEGIN:VTODO
UID:{uuid}
DTSTART:{YYYYMMDD}
DUE:{YYYYMMDD}
DTSTAMP:{now_utc}
SUMMARY:{标题}
DESCRIPTION:{描述}
PRIORITY:{1-9}
STATUS:NEEDS-ACTION
END:VTODO
END:VCALENDAR
```

## 方法速查表

| 操作 | HTTP 方法 | 路径 | Content-Type | Depth |
|------|-----------|------|-------------|-------|
| 验证连通性 | PROPFIND | `${CALDAV_ROOT}/` | `application/xml` | 0 |
| 检查日历本 | PROPFIND | 日历本 URL | `application/xml` | 0 |
| 创建日历本（标准） | MKCOL | 日历本 URL | `application/xml` | — |
| 创建日历本（回退，PUT 空 VCALENDAR） | PUT | 日历本 URL | `text/calendar` | — |
| 创建事件/任务 | PUT | 日历本 URL`/${UID}.ics` | `text/calendar` | — |
| 查询事件/任务 | REPORT | 日历本 URL | `application/xml` | 1 |
| 获取事件/任务 | GET | 日历本 URL`/${UID}.ics` | — | — |
| 修改事件/任务 | PUT | 日历本 URL`/${UID}.ics` | `text/calendar` | — |
| 删除事件/任务 | DELETE | 日历本 URL`/${UID}.ics` | — | — |

## HTTP 状态码

| 状态码 | 含义 | 处理 |
|--------|------|------|
| 207 | PROPFIND/REPORT 成功（multistatus） | 正常解析 |
| 201 | MKCOL/PUT 创建成功 | 正常 |
| 204 | PUT/DELETE 更新/删除成功（无 body） | 正常 |
| 200 | GET 成功 | 正常 |
| 401 | 认证失败 | 引导用户重新输入凭据 |
| 403 | 权限拒绝 | 尝试回退方案（如 PUT 替代 MKCOL） |
| 404 | 资源不存在 | 返回友好提示 |
| 409 | ETag 冲突（并发修改） | 重新 GET 最新版后重试 |

## CalDAV XML Namespace

| 前缀 | URI |
|------|-----|
| `D:` | `DAV:` |
| `C:` | `urn:ietf:params:xml:ns:caldav` |

## 踩坑记录

### Radicale 3.7

| 问题 | 现象 | 解决 |
|------|------|------|
| MKCOL 403 | Radicale 3.7 默认权限配置拒绝 MKCOL | 回退为 PUT 空 VCALENDAR 创建日历本 |
| defaultMode 0440 | 容器内 radicale 用户无法读取 Secret 挂载 | 改为 0444 |
| `dns_lookup` 废弃 | Radicale 3.x 已移除该配置项，启动报错 | 删除该配置 |
| PUT 路径 vs UID | Radicale 从 .ics 内容提取 UID 作为实际文件名 | PUT URL 中的文件名不强制等于 UID |

### 通用注意事项

- **curl 参数**：`-k` 跳过自签名证书验证（内网），`-s` 隐藏进度但保留错误码
- **Basic Auth**：通过 `-u` 参数直接传递，无需手动编码
- **REPORT**：必须带 `Depth: 1` 头
- **并行操作**：同一日历本的多个独立操作可并行执行
- **UID 生成**：使用 `uuidgen`，确保全局唯一
