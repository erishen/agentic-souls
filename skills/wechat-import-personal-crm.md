# WeChat Import Personal CRM Skill

> 将微信聊天记录导入 `personal/personal-crm` 的端到端操作手册

## Skill 定义

```yaml
name: wechat-import-personal-crm
description: 从已导出的微信聊天记录导入 personal-crm，并同时写入 ChatMessage 与 ContactRecord
version: 1.0.0
category: workflow
```

## 适用场景

```yaml
use_when:
  - 已有 PyWxDump/CSV/JSON 格式的微信聊天记录
  - 需要让聊天记录同时出现在联系人详情、Dashboard、聊天分析/七元素评分里
  - 需要复现之前导入过 personal-crm 的流程

related_skills:
  - wechat-data-extract
  - wechat-contact-match
  - wechat-sync-pipeline
  - wechat-key-extract
```

## 本地数据库可获取的数据范围（2026-06-19 调查确认）

微信 Mac 4.x 本地数据库能提取的联系人数据**有边界**，以下数据**无法**从本地获取：

| 数据 | 本地可获取 | 来源 |
|------|-----------|------|
| 标签列表（标签名） | ✅ | `contact_label` 表（label_id/name/sort_order） |
| 联系人-标签关联 | ❌ | 云端存储，通过 CGI 请求按需拉取，本地无存储 |
| 朋友权限（聊天/朋友圈/看我的） | ❌ | 云端存储，通过 CGI 请求按需拉取 |
| 联系人基本信息（昵称/备注/头像） | ✅ | `contact` 表 |
| 联系人地区/公司 | ✅ | `extra_buffer` protobuf（field 5=国家, 6=省份, 7=城市, 9=公司） |
| 联系人状态位 | ✅ | `contact.flag`（bit0=好友, bit1=可发消息, bit16=有备注, bit28=群聊） |

**已排查的 11 处位置**（均无标签关联数据）：
1. `contact` 表 `extra_buffer` protobuf — 无 label_id 字段
2. `contact_label` 表 — 只有标签定义，无关联表
3. `FMessageTable.label_ids_` — 95 条记录全空
4. `oplog` 表 — 0 条记录
5. `ContactMMKV` — 只有 `Update_<wxid>` 版本信息
6. 所有 MMKV 文件 — 搜索 label/tag/标签 零匹配
7. `flag` 字段 — 类型/状态位掩码，非朋友权限
8. `general.db` / `session.db` — 无标签关联表
9. `favorite.db` — 有收藏标签，非联系人标签
10. `d41d8cd98f00b204e9800998ecf8427e` 目录 — 无数据库
11. 所有未解密 DB — 只有 WebKit/小程序相关

**结论**：微信架构是核心数据本地存储、管理类数据云端存储。标签关联和朋友权限属于管理类数据，只在用户打开「通讯录管理」时通过 CGI 请求拉取。如需标签关联数据，只能在 CRM 系统中手动维护。

## 当前可用入口

### 推荐入口：`backend/wechat_sync.py`

这是完整流水线，负责：

1. 读取 `pywxdump`、`csv`、`json` 数据源
2. 标准化消息字段
3. 用 CRM 联系人的 `wechat_id` 匹配消息里的 `counterpart_wxid`
4. 写入 `chat_messages`
5. 派生写入 `contact_records`

推荐命令：

```bash
cd personal/personal-crm/backend
uv run python wechat_sync.py --source pywxdump --file /path/to/wechat_export.json --api-base http://localhost:8000/api
```

### 旧入口：`POST /api/wechat/import-messages`

这个入口只写 `contact_records`，不写 `chat_messages`。它适合快速补 Dashboard 联系记录，但聊天分析、七元素评分、AI 深度分析看不到这些数据。

除非只想生成联系记录，否则不要把它作为主导入入口。

## 前置条件

### 启动 personal-crm API

方式 A：本地后端

```bash
cd personal/personal-crm/backend
uv run uvicorn app.main:app --host 0.0.0.0 --port 8000
```

方式 B：Docker

```bash
cd personal/personal-crm
docker compose up --build
```

健康检查：

```bash
curl http://localhost:8000/
```

### 确认联系人可匹配

`wechat_sync.py` 不是靠 `--contact-id` 强制导入。当前脚本虽然声明了 `--contact-id` 参数，但主流程没有使用它。

实际匹配规则：

1. 消息里的 `counterpart_wxid`
2. 匹配 CRM 联系人的 `wechat_id`
3. 未匹配时按联系人 `name` 做降级匹配
4. 默认忽略未匹配消息；加 `--match-mode create_contact` 才会自动创建联系人

因此，导入前要先确保联系人有正确的 `wechat_id`：

```bash
curl http://localhost:8000/api/contacts/
```

如果导入日志里出现大量 `未匹配`，优先补联系人 `wechat_id`，或临时使用：

```bash
uv run python wechat_sync.py --source pywxdump --file /path/to/wechat_export.json --match-mode create_contact
```

## 数据源格式

### PyWxDump JSON（推荐）

输入结构：

```json
{
  "messages": [
    {
      "wxid": "wxid_friend",
      "content": "明天见",
      "msg_type": 1,
      "create_time": 1740000000,
      "is_sender": false
    }
  ],
  "my_wxid": "wxid_me"
}
```

命令：

```bash
cd personal/personal-crm/backend
uv run python wechat_sync.py --source pywxdump --file /path/to/wechat_export.json
```

注意：当 `is_sender=true` 时，脚本会从同一聊天里的 received 消息推断对方 `counterpart_wxid`。单个文件最好只包含一个对话；如果一个文件混了多个对话，sent 消息可能无法准确归属。

### 标准 JSON

输入可以是数组，或带 `messages` 字段的对象。每条消息建议提供：

```json
{
  "messages": [
    {
      "counterpart_wxid": "wxid_friend",
      "sender_wxid": "wxid_friend",
      "direction": "received",
      "content": "明天见",
      "msg_type": "text",
      "sent_at": "2025-02-19T15:00:00+08:00"
    }
  ]
}
```

命令：

```bash
cd personal/personal-crm/backend
uv run python wechat_sync.py --source json --file /path/to/wechat_messages.json
```

### CSV

支持常见字段名：

| 目标字段 | 可识别字段名 |
|---|---|
| 时间 | `timestamp`, `time`, `create_time`, `sent_at`, `日期`, `时间` |
| 内容 | `content`, `text`, `message`, `内容`, `消息`, `正文` |
| 类型 | `type`, `msg_type`, `消息类型`, `类型` |
| 方向 | `direction`, `is_sender`, `发送者`, `方向` |
| 发送者/匹配键 | `sender`, `from`, `wxid`, `发送者ID`, `发送人` |

命令：

```bash
cd personal/personal-crm/backend
uv run python wechat_sync.py --source csv --file /path/to/wechat_chat.csv
```

CSV 模式里 `sender/wxid` 会被当作匹配键。当前 `--contact-id` 不生效，所以单联系人 CSV 也需要能匹配到 CRM 的 `wechat_id`。

## 从 Mac 微信本地数据库提取

如果已经有可解密的数据库密钥，可以走：

```bash
cd personal/personal-crm/backend
uv run python wechat_msg_extract.py
uv run python wechat_sync.py --source json --file /tmp/wechat_messages_standardized.json
```

### 2026-06-19 实测完整流程（成功导入真实数据）

之前的 skill 文档认为 Mac WeChat 4.1.7 密钥提取失败，实测纠正：**密钥提取是成功的，问题在解密方法**。完整流程：

```bash
# 1. 提取密钥（需要 sudo，会重启微信）
cd personal/personal-crm/backend
sudo bash wechat_key_extract.sh
# → 生成 wechat_keys/all_keys.json，22 个密钥

# 2. 解密数据库（用 pycryptodome 方式，不用 sqlcipher CLI）
#    wechat_msg_extract.py 的 decrypt_sqlcipher 用 PRAGMA key 走 KDF，对 raw key 无效
ln -sf personal/personal-crm/backend/wechat_keys/all_keys.json /tmp/wechat-decrypt/all_keys.json
cd /tmp/wechat-decrypt && python3 decrypt_db.py
# → 21 成功 0 失败，1.1GB，解密文件在 /tmp/wechat-decrypt/decrypted/

# 3. 提取消息（用已解密的 DB，跳过解密步骤）
cd personal/personal-crm/backend
uv run python wechat_msg_extract.py \
  --decrypted-dir /tmp/wechat-decrypt/decrypted/message \
  --contact-db /tmp/wechat_decrypted_dbs/contact.db \
  --my-wxid erishen \
  --output /tmp/wechat_real_messages.json
# → 35万条消息（含公众号 19.9万、群聊 14.3万、真人单聊 1.1万）

# 4. 过滤真人单聊（排除公众号 gh_*、群聊 @chatroom、系统通知）
python3 -c "
import json
d = json.load(open('/tmp/wechat_real_messages.json'))
msgs = d['messages']
real = [m for m in msgs
        if m.get('counterpart_wxid')
        and not m['counterpart_wxid'].startswith('gh_')
        and '@chatroom' not in m['counterpart_wxid']
        and m['counterpart_wxid'] not in ('notifymessage','filehelper','weixin')]
json.dump({'messages': real, 'my_wxid': 'erishen'}, open('/tmp/wechat_real_personal.json','w'), ensure_ascii=False, indent=2)
print(f'过滤后: {len(real)} 条')
"

# 5. 导入 CRM（自动创建联系人）
uv run python wechat_sync.py --source json \
  --file /tmp/wechat_real_personal.json \
  --api-base http://localhost:8000/api \
  --match-mode create_contact
# → 约 250 个有聊天记录的联系人，1万+ ChatMessage，1万+ ContactRecord
```

### 联系人全量同步（2026-06-19 新增，含分组推断）

`wechat_sync.py --match-mode create_contact` 只在有聊天记录时创建联系人，会漏掉没怎么聊天的微信好友。需要单独从微信 `contact.db` 全量同步联系人：

```python
import sqlite3, sys, os
# 复用 wechat_sync.py 的 infer_group 函数
sys.path.insert(0, os.path.join(os.path.dirname(__file__), '..', 'backend'))
from wechat_sync import infer_group

# 从微信 contact.db 读真人好友（排除公众号、群聊、企业微信、系统账号）
wx_conn = sqlite3.connect('/tmp/wechat_decrypted_dbs/contact.db')
wx_cur = wx_conn.cursor()
wx_cur.execute("""
    SELECT username, remark, nick_name FROM contact
    WHERE local_type = 1
    AND username NOT LIKE 'gh_%'
    AND username NOT LIKE '%@openim'
    AND username NOT LIKE '%@chatroom'
    AND username NOT IN ('weixin','filehelper','notifymessage','mphelper','weixinguanhaozhushou')
    AND username IS NOT NULL AND username != ''
""")
wx_friends = wx_cur.fetchall()
wx_conn.close()

# 读 CRM 已有 wechat_id 去重
crm_conn = sqlite3.connect('personal/personal-crm/data/crm.db')
crm_cur = crm_conn.cursor()
crm_cur.execute("SELECT wechat_id FROM contacts WHERE wechat_id IS NOT NULL AND wechat_id != ''")
existing = set(r[0] for r in crm_cur.fetchall())

# 创建缺失的联系人（自动推断分组）
crm_cur.execute("BEGIN")
created = 0
for username, remark, nick in wx_friends:
    if username in existing:
        continue
    name = (remark or nick or username)[:100]
    group = infer_group(name)  # 自动推断分组
    crm_cur.execute(
        "INSERT INTO contacts (name, wechat_id, \"group\", created_at, updated_at) VALUES (?,?,?,datetime('now'),datetime('now'))",
        (name, username, group)
    )
    created += 1
crm_cur.execute("COMMIT")
print(f"新建: {created}")
crm_conn.close()
```

⚠️ **分组推断规则**（`wechat_sync.py` 的 `infer_group` 函数，与 `scripts/infer_groups.py` 保持同步）：
- 家人关键词（爸/妈/哥/姐/亲戚等）→ 家人（⚠️ "XX爸爸/XX妈妈"家长群格式**不是**家人，是孩子同学的家长）
- 教育关键词（老师/学校/美术/培优等）→ 客户
- 服务关键词（中介/房产/猎头/HR/维修/客服等）→ 客户
- 同事关键词（携程/Ctrip/5173等）→ 同事
- 纯人名（2-4字中文或英文姓名）→ 朋友
- 其余 → 其他

**改名后重新推断**：联系人改名（从微信同步 remark/nick_name）后，如果当前 group 是空或"其他"，应重新推断。已有非"其他"分组的联系人**不覆盖**（保留用户手动设置）。

批量重新推断已有联系人的分组可用 `scripts/infer_groups.py`：
```bash
python3 scripts/infer_groups.py --dry-run --db ../data/crm.db  # 预览
python3 scripts/infer_groups.py --db ../data/crm.db             # 执行
```

### 联系人改名（从微信 contact.db 同步真实姓名）

导入后很多联系人名字是 `wxid_*` 或 `@openim` 形式，需要从微信 `contact.db` 同步 remark/nick_name。改名后如果当前分组是空或"其他"，同时重新推断分组：

```python
import sqlite3, sys, os
# 复用 wechat_sync.py 的 infer_group 函数
sys.path.insert(0, os.path.join(os.path.dirname(__file__), '..', 'backend'))
from wechat_sync import infer_group

# 微信通讯录映射
wx_conn = sqlite3.connect('/tmp/wechat_decrypted_dbs/contact.db')
wx_map = {}
for row in wx_conn.execute("SELECT username, remark, nick_name FROM contact WHERE username IS NOT NULL"):
    wx_map[row[0]] = (row[1] or '', row[2] or '')
wx_conn.close()

# 更新 CRM 里名字还是 wxid/@openim 形式的联系人
crm_conn = sqlite3.connect('personal/personal-crm/data/crm.db')
crm_cur = crm_conn.cursor()
crm_cur.execute("SELECT id, name, wechat_id, \"group\" FROM contacts WHERE name LIKE 'wxid_%' OR name LIKE '%@openim'")
rows = crm_cur.fetchall()
crm_cur.execute("BEGIN")
updated = 0
regrouped = 0
for cid, old_name, wxid, current_group in rows:
    info = wx_map.get(wxid)
    if not info:
        continue
    new_name = (info[0] or info[1])[:100]  # 优先 remark，其次 nick_name
    if not new_name:
        continue
    # 改名后，如果当前分组是空或"其他"，重新推断
    if not current_group or current_group == '其他':
        new_group = infer_group(new_name)
        crm_conn.execute("UPDATE contacts SET name=?, \"group\"=? WHERE id=?", (new_name, new_group, cid))
        if new_group != '其他':
            regrouped += 1
    else:
        crm_conn.execute("UPDATE contacts SET name=? WHERE id=?", (new_name, cid))
    updated += 1
crm_cur.execute("COMMIT")
print(f"改名: {updated}, 重新分组: {regrouped}")
crm_conn.close()
```

### 重复联系人合并（如多次导入导致）

`wechat_sync.py` 的 `create_contact` 没有先查 wechat_id 是否已存在，多次导入会产生重复。合并策略：保留消息最多的 id，把重复 id 的聊天记录和联系记录转移过去，然后删掉重复联系人：

```python
import sqlite3
conn = sqlite3.connect('personal/personal-crm/data/crm.db')
cur = conn.cursor()

# 找重复
cur.execute("SELECT wechat_id, GROUP_CONCAT(id) FROM contacts WHERE wechat_id IS NOT NULL AND wechat_id != '' GROUP BY wechat_id HAVING count(*) > 1")
for wxid, ids_str in cur.fetchall():
    ids = [int(x) for x in ids_str.split(',')]
    # 按 chat_messages 数排序，保留消息最多的
    counts = [(cid, cur.execute("SELECT count(*) FROM chat_messages WHERE contact_id=?", (cid,)).fetchone()[0]) for cid in ids]
    counts.sort(key=lambda x: (-x[1], x[0]))
    keep_id, dup_ids = counts[0][0], [c[0] for c in counts[1:]]
    for dup_id in dup_ids:
        cur.execute("UPDATE chat_messages SET contact_id=? WHERE contact_id=?", (keep_id, dup_id))
        cur.execute("UPDATE contact_records SET contact_id=? WHERE contact_id=?", (keep_id, dup_id))
        cur.execute("DELETE FROM chat_imports WHERE contact_id=?", (dup_id,))
        cur.execute("DELETE FROM contacts WHERE id=?", (dup_id,))
conn.commit()
conn.close()
```

### wechat_msg_extract.py 的修复（2026-06-19）

原版有两个 bug 导致提取的消息全部 `counterpart_wxid` 为空：

1. **`build_name_map` 找不到 id 列**：WeChat 4.x 的 `Name2Id` 表只有 `user_name` + `is_session` 两列，没有显式 id 列。原代码用 `pick(cols, ["id", "rowid", ...])` 找 id 列失败就返回空 dict。修复：用 `rowid` 兜底
2. **`is_sent` 用 `local_type == 1` 误判方向**：实测同一会话里 `local_type=1` 既可能是自己也可能是对方。修复：优先用 `real_sender_id` 对比 `my_wxid` 在 `Name2Id` 里的 rowid 判断方向
3. **新增 `build_md5_to_wxid_map`**：`Msg_<md5(wxid)>` 表名反查会话对端 wxid

### 数据源优先级（更新）

1. 本地数据库解密提取（2026-06-19 实测成功，详见上流程）
2. 已导出的 PyWxDump JSON
3. 手工/工具整理出的标准 JSON
4. CSV
5. 旧的 `/api/wechat/import-messages` 快速导 ContactRecord

## 完整执行流程

### 1. 准备数据

优先整理成 PyWxDump JSON 或标准 JSON。确认至少有：

- 对方标识：`wxid` 或 `counterpart_wxid`
- 内容：`content`
- 时间：`create_time` 或 `sent_at`
- 方向：`is_sender` 或 `direction`

### 2. 启动 API

```bash
cd personal/personal-crm/backend
uv run uvicorn app.main:app --host 0.0.0.0 --port 8000
```

### 3. 试跑样例

```bash
cd personal/personal-crm/backend
uv run python wechat_sync.py --test --api-base http://localhost:8000/api
```

如果样例提示没有匹配联系人，说明 API 正常，匹配数据还没准备好。

### 4. 执行导入

```bash
cd personal/personal-crm/backend
uv run python wechat_sync.py --source pywxdump --file /path/to/wechat_export.json --api-base http://localhost:8000/api
```

日志里应看到：

```text
Step 1: 数据提取
Step 2: 联系人匹配
Step 3: 同步流水线
ChatMessage 导入成功
ContactRecord 派生写入
```

### 5. 验证写入

检查聊天统计：

```bash
curl http://localhost:8000/api/chat/stats/CONTACT_ID
```

检查联系记录：

```bash
curl http://localhost:8000/api/contacts/CONTACT_ID/records
```

页面验证：

- 联系人详情页能看到微信聊天记录
- Dashboard 的微信消息统计有数据
- 聊天分析/七元素评分能读取到消息

## 实测验证（2026-06-19）

在 `personal-crm` API 已启动（`uv run uvicorn app.main:app --port 8000`）的环境下，分别跑了：

- `wechat_sync.py --test` —— 用脚本内置 5 条样例数据
- `wechat_sync.py --source pywxdump --file /tmp/wx_test_real.json` —— 用一份 3 条新消息的 PyWxDump JSON

两次主流程都跑通，`ChatMessage` 与 `ContactRecord` 双表写入逻辑验证通过（`/api/chat/stats/{id}` 看到 ChatMessage 计数，`/api/contacts/{id}/records` 看到新增的微信类型 ContactRecord）。

实测暴露出 3 个需要注意的坑，使用前必须知晓：

### 坑 1：name 降级匹配会误匹配同名联系人

`build_match_map` 的降级策略是 `contact.name == wxid or contact.name in wxid`。如果 CRM 联系人表里存在以 wxid 字符串（如 `wxid_test_friend`、`wxid_me`）为 name 的联系人，所有测试用假 wxid 都会被错配过去。

实测时 5 条测试消息全部错配到了 ID 21/22/23 三个名字正好是 `wxid_me` / `wxid_another_friend` / `wxid_test_friend` 的现有联系人。

**规避**：

1. 导入前用 `curl http://localhost:8000/api/contacts/` 检查是否有名字形如 `wxid_*` 的联系人，提前改名或删除
2. 或在 `build_match_map` 里给 name 降级匹配加正则白名单（只匹配非 `wxid_` 前缀的姓名）
3. 真实导入时优先确保联系人 `wechat_id` 字段已填，让精确匹配优先命中

### 坑 2：`GET /api/chat/messages/{contact_id}` 不存在（返回 405）

`wechat_sync.py` 的 `get_last_sync_timestamp` 和 `sync_to_crm` 都调用了 `GET /api/chat/messages/{contact_id}` 做预去重，但 `app/routers/chat.py` 在该路径上**只注册了 DELETE**，没有 GET。

脚本用 `if resp.status_code == 200` 静默吞掉了 405，所以表面看主流程"正常"，但实际上：

- 脚本侧的预去重 `existing_keys` 永远是空集
- 真正起作用的去重是 `/api/chat/import/{contact_id}` 内部按 `content + sent_at` 的去重

这意味着脚本日志里"增量去重: 输入 N 条 → 新增 N 条 (跳过 0 条重复)"中的"跳过 0 条"实际上不可信 —— 真实去重发生在 API 内部，脚本侧统计不到。

**修复方向**（任选其一）：

1. 在 `chat.py` 增加 `@router.get("/messages/{contact_id}")` 端点
2. 改 `wechat_sync.py` 用其他方式查已有消息（如新增 `/api/chat/list/{contact_id}` 或直接读 SQLite）
3. 至少把脚本里 405 当作警告打印出来，避免静默失效

### 坑 3：ContactRecord 的 contact_date 取的是 UTC 日期，不是上海日期

`sync_to_crm` 派生 ContactRecord 时，循环变量 `msg['sent_at']` 是已经过 `convert_sent_at_for_api` 转换的 UTC 字符串（如 `"2025-06-19 09:20:00"`，对应上海时间 17:20），随后 `sent_at_dt.date().isoformat()` 提取的是 **UTC 日期**。

对上海时间 0:00–8:00 的消息，UTC 日期会早一天，导致：

- `ContactRecord.contact_date` 比 `ChatMessage.sent_at` 的实际上海日期早一日
- Dashboard 按日统计的联系频率会偏移
- 派生去重键 `contact_date + direction + content[:200]` 也用 UTC 日期，与已有 ContactRecord（如果之前用上海日期写入）比对时可能 miss

**规避**：在派生 ContactRecord 前，把 `msg['sent_at']` 重新按上海时区解析一次：

```python
sent_at_dt = datetime.fromisoformat(msg['sent_at']).replace(tzinfo=timezone.utc).astimezone(SHANGHAI_TZ)
contact_date = sent_at_dt.date().isoformat()
```

## 常见问题

### 导入完成但聊天分析没有数据

大概率用了旧入口 `/api/wechat/import-messages`。它只写 `contact_records`，不写 `chat_messages`。

解决：改用 `wechat_sync.py` 重新导入。

### 大量未匹配

原因：消息的 `wxid/counterpart_wxid` 和 CRM 联系人的 `wechat_id` 对不上。

解决：

1. 在 CRM 里补联系人 `wechat_id`
2. 或用 `--match-mode create_contact` 自动创建联系人
3. 或先把 JSON 里的 `counterpart_wxid` 规范成现有联系人 `wechat_id`

### 传了 `--contact-id` 但仍未导入到指定联系人

当前 `wechat_sync.py` 没有在主流程使用 `--contact-id`。不要依赖这个参数。

如果要强制导入单联系人，需要改脚本逻辑，或直接调用 `/api/chat/import/{contact_id}` 上传 JSON，再手动补 `ContactRecord`。

### 重复导入

`chat_messages` 的去重键是：

```text
contact_id + sent_at + content
```

`contact_records` 的派生去重当前使用：

```text
contact_date + direction + content[:200]
```

因此重复跑同一份文件通常不会重复写入聊天消息。

注意：`wechat_sync.py` 当前会尝试请求 `GET /api/chat/messages/{contact_id}` 做预去重，但后端现有路由没有这个 GET endpoint，只有 DELETE endpoint。这个预去重失败时不会阻止导入，因为 `/api/chat/import/{contact_id}` 内部还会按 `contact_id + sent_at + content` 再去重。

### 时间偏移

`wechat_sync.py` 内部会把带 `+08:00` 的 ISO 时间转成 API 当前支持的 UTC 字符串。API 解析后按 UTC 存储。

验证时不要只看日期，尤其是凌晨消息，可能因为时区转换落到前一天。

## 质量标准

```yaml
quality:
  primary_path: "优先使用 wechat_sync.py，而不是旧的 /api/wechat/import-messages"
  double_write: "同一批消息必须同时写入 chat_messages 和 contact_records"
  matching: "导入前确认 counterpart_wxid 能匹配 CRM contact.wechat_id"
  verification: "导入后同时验证 /api/chat/stats/{id} 和 /api/contacts/{id}/records"
  privacy: "不要把微信原始数据库、密钥、all_keys.json 提交到版本库"
```

## 相关文件

```yaml
personal_crm:
  sync_script: personal/personal-crm/backend/wechat_sync.py
  db_extract_script: personal/personal-crm/backend/wechat_msg_extract.py
  key_extract_script: personal/personal-crm/backend/wechat_key_extract.sh
  old_api_router: personal/personal-crm/backend/app/routers/wechat.py
  chat_import_router: personal/personal-crm/backend/app/routers/chat.py
  models: personal/personal-crm/backend/app/models.py

skills:
  data_extract: frameworks/agentic-souls/skills/wechat-data-extract.md
  contact_match: frameworks/agentic-souls/skills/wechat-contact-match.md
  sync_pipeline: frameworks/agentic-souls/skills/wechat-sync-pipeline.md
  key_extract: frameworks/agentic-souls/skills/wechat-key-extract.md
```
