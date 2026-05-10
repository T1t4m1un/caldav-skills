# 任务（VTODO）操作

## 前置

```bash
source ~/.caldavrc
CALENDAR_URL="${CALDAV_URL}${CALDAV_ROOT}/${CALDAV_CALENDAR}/"
```

## 创建任务

### 构造 .ics

```bash
UUID=$(uuidgen)
NOW=$(date -u +%Y%m%dT%H%M%SZ)

cat <<EOF > /tmp/caldav-todo.ics
BEGIN:VCALENDAR
VERSION:2.0
PRODID:-//CalDAV Skill//EN
BEGIN:VTODO
UID:${UUID}
DTSTART:20260510T090000
DUE:20260515T180000
DTSTAMP:${NOW}
SUMMARY:完成 CalDAV Skill 文档
DESCRIPTION:写出 SKILL.md 及各子模块
PRIORITY:3
STATUS:NEEDS-ACTION
END:VTODO
END:VCALENDAR
EOF
```

**必填字段**：UID、SUMMARY、DTSTAMP、STATUS（初始值 `NEEDS-ACTION`）
**可选字段**：DUE、DESCRIPTION、PRIORITY、DTSTART

### 字段说明

| 字段 | 值 | 说明 |
|------|-----|------|
| STATUS | `NEEDS-ACTION` / `COMPLETED` / `CANCELLED` | 任务状态 |
| PRIORITY | 1-9 | 1=最高, 5=中等, 9=最低 |
| DUE | `YYYYMMDD` 或 `YYYYMMDDTHHMMSS` | 截止日期 |
| COMPLETED | `YYYYMMDDTHHMMSSZ` | 完成时间戳（标记完成时添加） |

### 优先级映射

用户说"高优先级"/"紧急" → `PRIORITY:1`
用户说"中等" → `PRIORITY:5`
用户说"低优先级"/"不急" → `PRIORITY:9`
未指定 → 默认 `PRIORITY:5`

### PUT 写入

```bash
curl -sk -u "${CALDAV_USER}:${CALDAV_PASS}" \
  -X PUT -H "Content-Type: text/calendar; charset=utf-8" \
  --data-binary @/tmp/caldav-todo.ics \
  "${CALENDAR_URL}${UUID}.ics"
```

## 查询任务

### 查询未完成任务

```bash
cat <<EOF > /tmp/caldav-todo-query.xml
<?xml version="1.0" encoding="utf-8"?>
<C:calendar-query xmlns:D="DAV:" xmlns:C="urn:ietf:params:xml:ns:caldav">
  <D:prop>
    <D:getetag/>
    <C:calendar-data/>
  </D:prop>
  <C:filter>
    <C:comp-filter name="VCALENDAR">
      <C:comp-filter name="VTODO">
        <C:prop-filter name="STATUS">
          <C:text-match collation="i;octet">NEEDS-ACTION</C:text-match>
        </C:prop-filter>
      </C:comp-filter>
    </C:comp-filter>
  </C:filter>
</C:calendar-query>
EOF

curl -sk -u "${CALDAV_USER}:${CALDAV_PASS}" \
  -X REPORT -H "Content-Type: application/xml" \
  -H "Depth: 1" --data-binary @/tmp/caldav-todo-query.xml \
  "${CALENDAR_URL}"
```

### 查询全部任务

去掉 `<C:prop-filter>` 即可。

### 按截止日期范围查询

```xml
<C:comp-filter name="VTODO">
  <C:time-range start="YYYYMMDDTHHMMSSZ" end="YYYYMMDDTHHMMSSZ"/>
</C:comp-filter>
```

### 搜索关键词

```xml
<C:prop-filter name="SUMMARY">
  <C:text-match collation="i;octet">关键词</C:text-match>
</C:prop-filter>
```

### 解析与展示

提取 SUMMARY、DUE、PRIORITY、STATUS，按 DUE 倒序排列（临近截止的排前面），未完成的优先显示。

## 标记任务完成

```bash
# 1. GET 获取当前 VTODO .ics
curl -sk -u "${CALDAV_USER}:${CALDAV_PASS}" \
  "${CALENDAR_URL}${UID}.ics" -o /tmp/caldav-todo.ics

# 2. 修改 .ics：
#    STATUS:NEEDS-ACTION → STATUS:COMPLETED
#    添加 COMPLETED:$(date -u +%Y%m%dT%H%M%SZ)
#    更新 DTSTAMP 为当前时间

# 3. PUT 更新
curl -sk -u "${CALDAV_USER}:${CALDAV_PASS}" \
  -X PUT -H "Content-Type: text/calendar; charset=utf-8" \
  --data-binary @/tmp/caldav-todo.ics \
  "${CALENDAR_URL}${UID}.ics"

# 4. GET 验证 STATUS 已变为 COMPLETED
curl -sk -u "${CALDAV_USER}:${CALDAV_PASS}" \
  "${CALENDAR_URL}${UID}.ics"
```

## 修改任务

流程同修改日程：GET → 编辑 .ics 字段 → PUT → GET 验证。

可修改：SUMMARY、DESCRIPTION、DUE、PRIORITY、STATUS。
将 STATUS 从 COMPLETED 改回 NEEDS-ACTION 即可取消完成状态。

## 删除任务

```bash
curl -sk -u "${CALDAV_USER}:${CALDAV_PASS}" \
  -X DELETE "${CALENDAR_URL}${UID}.ics"
```
