# CalDAV Skill

让 AI 代理（opencode/Claude Code/Cursor）通过标准 CalDAV 协议操作个人日程（VEVENT）和任务（VTODO）。

**兼容**：Radicale / Baikal / Nextcloud / iCloud 等任意 CalDAV 服务器。

## 安装

### 方式一：下载 .skill 包（推荐）

从 [Releases](../../releases) 下载 `caldav.skill`，放到 `~/.agents/skills/` 目录解压即可。

### 方式二：直接放到 skills 目录

```bash
git clone https://github.com/T1t4m1un/caldav-skills.git ~/.agents/skills/caldav
```

### 方式三：子模块

```bash
git submodule add https://github.com/T1t4m1un/caldav-skills.git .agents/skills/caldav
```

## 使用

首次使用时 AI 代理会自动引导你配置凭据（服务器地址、用户名、密码），写入 `~/.caldavrc`（权限 0600）。

配置完成后直接对 AI 代理说：

- "帮我加个日程，明天下午2点团队周会"
- "查一下这周的日程安排"
- "创建一个待办：完成 CalDAV Skill"
- "把「完成 CalDAV Skill」标记为已完成"

## 安全

- 凭据加密存储在 `~/.caldavrc`（权限 0600）
- 不进 git
- 密码以明文存储，依赖文件系统权限保护

## 文件说明

| 文件 | 用途 |
|------|------|
| `SKILL.md` | 入口，加载策略 + 子模块索引 |
| `init.md` | 凭据初始化与日历本创建 |
| `events.md` | 日程（VEVENT）CRUD |
| `todos.md` | 任务（VTODO）CRUD |
| `reference.md` | 字段表、模板、方法速查、踩坑记录 |

## 协议

遵循 RFC 4791（CalDAV）+ RFC 5545（iCalendar）。
