# CalDAV 初始化与凭据管理

## 检查现有凭据

```bash
source ~/.caldavrc 2>/dev/null

if [[ -z "$CALDAV_URL" || -z "$CALDAV_USER" || -z "$CALDAV_PASS" ]]; then
  echo "凭据缺失，需要初始化"
fi
```

## 引导用户填写

若凭据缺失，依次向用户询问：

1. **CalDAV 服务器地址** — 如 `https://caldav.example.com`
2. **用户名**
3. **密码**
4. **Principal 路径**（可选）— 默认 `/`+用户名，如 `/alice`
5. **默认日历本名称**（可选）— 默认 `personal`

在同一个问题中一次性问清所有必填项（URL、用户名、密码），减轻用户交互负担。路径和日历本名称若用户未提供则使用默认值。

## 生成 ~/.caldavrc

收到用户信息后：

```bash
cat <<EOF > ~/.caldavrc
# CalDAV 连接配置（由 caldav skill 自动管理，不进 git）
CALDAV_URL="<用户提供的URL>"
CALDAV_USER="<用户名>"
CALDAV_PASS="<密码>"
CALDAV_ROOT="/<用户名>"
CALDAV_CALENDAR="personal"
EOF
chmod 0600 ~/.caldavrc
```

**安全要求**：
- 权限强制 0600，仅当前用户可读写
- 文件位于 `$HOME`，不在仓库内，不进 git
- 密码以明文存储，依赖文件系统权限保护

## 连通性验证

```bash
source ~/.caldavrc

curl -sk -u "${CALDAV_USER}:${CALDAV_PASS}" \
  -X PROPFIND -H "Content-Type: application/xml" \
  -H "Depth: 0" \
  -d '<?xml version="1.0" encoding="utf-8"?>
<D:propfind xmlns:D="DAV:">
  <D:prop>
    <D:current-user-principal/>
  </D:prop>
</D:propfind>' \
  "${CALDAV_URL}${CALDAV_ROOT}/"
```

返回 HTTP 207 Multi-Status 表示连通成功。若 401 则凭据错误，引导用户重新输入。

## 日历本检查与创建

### 检查默认日历本是否存在

```bash
CALENDAR_URL="${CALDAV_URL}${CALDAV_ROOT}/${CALDAV_CALENDAR}/"

curl -sk -u "${CALDAV_USER}:${CALDAV_PASS}" \
  -X PROPFIND -H "Depth: 0" \
  "${CALENDAR_URL}"
```

404 表示日历本不存在，需要创建。

### 创建日历本

**优先方案** — MKCOL（标准 CalDAV）：

```bash
curl -sk -u "${CALDAV_USER}:${CALDAV_PASS}" \
  -X MKCOL -H "Content-Type: application/xml" \
  -d '<?xml version="1.0" encoding="utf-8"?>
<D:mkcol xmlns:D="DAV:" xmlns:C="urn:ietf:params:xml:ns:caldav">
  <D:set>
    <D:prop>
      <D:resourcetype>
        <D:collection/>
        <C:calendar/>
      </D:resourcetype>
    </D:prop>
  </D:set>
</D:mkcol>' \
  "${CALENDAR_URL}"
```

**回退方案** — PUT 空 VCALENDAR（Radicale 3.7 MKCOL 返回 403 时使用）：

```bash
curl -sk -u "${CALDAV_USER}:${CALDAV_PASS}" \
  -X PUT -H "Content-Type: text/calendar; charset=utf-8" \
  --data-binary @- "${CALENDAR_URL}" <<'EOF'
BEGIN:VCALENDAR
VERSION:2.0
PRODID:-//CalDAV Skill//EN
END:VCALENDAR
EOF
```

### 验证创建结果

```bash
curl -sk -u "${CALDAV_USER}:${CALDAV_PASS}" \
  -X PROPFIND -H "Content-Type: application/xml" \
  -H "Depth: 0" \
  -d '<?xml version="1.0" encoding="utf-8"?>
<D:propfind xmlns:D="DAV:" xmlns:C="urn:ietf:params:xml:ns:caldav">
  <D:prop>
    <D:resourcetype/>
  </D:prop>
</D:propfind>' \
  "${CALENDAR_URL}"
```

返回中包含 `<C:calendar/>` 即为成功。

## 初始化完成

完成后告知用户："CalDAV 初始化完成，日历本 `${CALDAV_CALENDAR}` 已就绪。现在可以对我说「加个日程」或「创建待办」开始使用。"
