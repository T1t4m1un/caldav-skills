# 日程（VEVENT）操作

## 前置

```bash
source ~/.caldavrc
CALENDAR_URL="${CALDAV_URL}${CALDAV_ROOT}/${CALDAV_CALENDAR}/"
```

## 创建日程

### 构造 .ics

```bash
UUID=$(uuidgen)
NOW=$(date -u +%Y%m%dT%H%M%SZ)

cat <<EOF > /tmp/caldav-event.ics
BEGIN:VCALENDAR
VERSION:2.0
PRODID:-//CalDAV Skill//EN
BEGIN:VEVENT
UID:${UUID}
DTSTART:20260511T140000
DTEND:20260511T150000
DTSTAMP:${NOW}
SUMMARY:团队周会
DESCRIPTION:讨论本周进度和下周计划
LOCATION:线上会议室
END:VEVENT
END:VCALENDAR
EOF
```

**必填字段**：UID、DTSTART、DTSTAMP、SUMMARY
**可选字段**：DTEND、DESCRIPTION、LOCATION

### 时间格式

| 类型 | 格式 | 示例 |
|------|------|------|
| 日期时间（本地） | `YYYYMMDDTHHMMSS` | `20260511T140000` |
| 日期时间（UTC） | `YYYYMMDDTHHMMSSZ` | `20260511T060000Z` |
| 全天日期 | `YYYYMMDD` | `20260511` |

### 时区处理

- 用户说"下午2点"且未指定时区 → 本地时间 `YYYYMMDDTHHMMSS`
- 用户指定时区或 UTC → 转换为 UTC 加 `Z` 后缀
- 用户说"明天全天" → 日期格式 `YYYYMMDD`，不设 DTEND

### PUT 写入

```bash
curl -sk -u "${CALDAV_USER}:${CALDAV_PASS}" \
  -X PUT -H "Content-Type: text/calendar; charset=utf-8" \
  --data-binary @/tmp/caldav-event.ics \
  "${CALENDAR_URL}${UUID}.ics"
```

### 验证写入

```bash
curl -sk -u "${CALDAV_USER}:${CALDAV_PASS}" \
  "${CALENDAR_URL}${UUID}.ics"
```

HTTP 200 + 返回 .ics 内容 → 成功。

## 查询日程

### 按日期范围

```bash
START="20260511T000000Z"   # Range 开始（UTC）
END="20260517T235959Z"     # Range 结束（UTC）

cat <<EOF > /tmp/caldav-query.xml
<?xml version="1.0" encoding="utf-8"?>
<C:calendar-query xmlns:D="DAV:" xmlns:C="urn:ietf:params:xml:ns:caldav">
  <D:prop>
    <D:getetag/>
    <C:calendar-data/>
  </D:prop>
  <C:filter>
    <C:comp-filter name="VCALENDAR">
      <C:comp-filter name="VEVENT">
        <C:time-range start="${START}" end="${END}"/>
      </C:comp-filter>
    </C:comp-filter>
  </C:filter>
</C:calendar-query>
EOF

curl -sk -u "${CALDAV_USER}:${CALDAV_PASS}" \
  -X REPORT -H "Content-Type: application/xml" \
  -H "Depth: 1" --data-binary @/tmp/caldav-query.xml \
  "${CALENDAR_URL}"
```

### 快捷时间范围

```bash
# 本周一 00:00 UTC → 本周日 23:59 UTC
START=$(date -u -v-mon +%Y%m%dT000000Z)
END=$(date -u -v+sun +%Y%m%dT235959Z)

# 今天 → 7 天后
START=$(date -u +%Y%m%dT000000Z)
END=$(date -u -v+7d +%Y%m%dT235959Z)

# 本月
START=$(date -u -v1d +%Y%m%dT000000Z)
END=$(date -u -v+1m -v-1d +%Y%m%dT235959Z)
```

### 搜索关键词

在 `<C:comp-filter name="VEVENT">` 内添加：

```xml
<C:prop-filter name="SUMMARY">
  <C:text-match collation="i;octet">关键词</C:text-match>
</C:prop-filter>
```

### 解析返回

返回 multistatus XML，提取：
- `<D:href>` — 事件路径（含 UID 文件名）
- `<C:calendar-data>` — 完整 .ics（含 SUMMARY/DTSTART/DTEND）
- `<D:getetag>` — 并发检查用

向用户展示时提取 SUMMARY、DTSTART、DTEND、LOCATION，按时间排序。

## 修改日程

```bash
# 1. GET 获取当前 .ics
curl -sk -u "${CALDAV_USER}:${CALDAV_PASS}" \
  "${CALENDAR_URL}${UID}.ics" -o /tmp/caldav-event.ics

# 2. 修改目标字段（SUMMARY / DTSTART / DTEND / DESCRIPTION / LOCATION）
#    保持 UID 不变，更新 DTSTAMP 为当前时间

# 3. PUT 覆盖写入
curl -sk -u "${CALDAV_USER}:${CALDAV_PASS}" \
  -X PUT -H "Content-Type: text/calendar; charset=utf-8" \
  --data-binary @/tmp/caldav-event.ics \
  "${CALENDAR_URL}${UID}.ics"

# 4. GET 验证
curl -sk -u "${CALDAV_USER}:${CALDAV_PASS}" \
  "${CALENDAR_URL}${UID}.ics"
```

## 删除日程

```bash
# 已知 UID → 直接 DELETE
curl -sk -u "${CALDAV_USER}:${CALDAV_PASS}" \
  -X DELETE "${CALENDAR_URL}${UID}.ics"

# 未知 UID → 先 REPORT 查询定位 href，再 DELETE
```

验证删除：GET 返回 404 即为成功。
