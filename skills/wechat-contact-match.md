# WeChat Contact Match Skill

> 微信联系人匹配与去重技能

## Skill 定义

```yaml
name: wechat-contact-match
description: 将微信 wxid/sender 映射到 personal-crm 的 contact_id，处理未匹配联系人
version: 1.0.0
category: implementation
```

## 能力范围

```yaml
capabilities:
  - wxid → contact_id 映射（降级策略：wechat_id → name → 创建新联系人）
  - 批量匹配（先建映射表再逐条查找，避免 N+1 查询）
  - 消息去重（contact_id + sent_at + content 组合键）
  - 未匹配联系人处理（创建新联系人 or 忽略，可配置）

limitations:
  - 不处理微信数据提取（由 wechat-data-extract skill 处理）
  - 不处理数据写入和同步流水线（由 wechat-sync-pipeline skill 处理）
  - 不处理 OSINT 搜索（由 personal-crm 现有 osint 功能处理）
  - 不涉及前端展示
```

## 匹配策略

### 降级匹配顺序

对每条消息的 counterpart_wxid，按以下顺序尝试匹配：

1. **wechat_id 精确匹配**：查询 `Contact.wechat_id` 字段，做 lowercase + 清理（去除空格、特殊字符）后比对
   ```python
   cleaned = wxid.lower().strip().replace(" ", "")
   contact = db.query(Contact).filter(func.lower(Contact.wechat_id) == cleaned).first()
   ```

2. **name 降级匹配**：wechat_id 未匹配时，查询 `Contact.name` 字段
   ```python
   contact = db.query(Contact).filter(Contact.name == sender_name).first()
   ```

   ⚠️ **副作用警告**（实测 2026-06-19 暴露）：`wechat_sync.py` 的 `build_match_map` 实际用的是 `contact.name == wxid or contact.name in wxid`，会把 wxid 字符串本身当作姓名去匹配。如果 CRM 里存在名字正好是 `wxid_xxx`、`wxid_me`、`wxid_test_friend` 这类字符串的联系人，所有同名的 wxid 都会被错配过去，且日志只显示"匹配成功 N 个"看不出异常。

   **规避**：

   - 导入前检查 `contacts` 表里是否有名字形如 `wxid_*` 的联系人，提前改名或删除
   - 或在 name 降级匹配里加正则白名单，只匹配不以 `wxid_` 开头的姓名
   - 真实导入优先保证联系人 `wechat_id` 字段已填，让精确匹配优先命中

3. **未匹配处理**：两种选项，由调用者配置
   - **选项 A：创建新联系人** — 自动创建一条 Contact 记录，wechat_id 设为 counterpart_wxid，name 设为 counterpart_wxid（后续需从微信 contact.db 同步真实姓名，见下方「联系人全量同步与改名」）
   - **选项 B：忽略** — 将该消息标记为 unmatched，不创建联系人，消息不写入

### 联系人全量同步与改名（2026-06-19 新增）

`--match-mode create_contact` 只在有聊天记录时创建联系人，会漏掉没怎么聊天的微信好友（实测 965 个好友只创建了 253 个）。完整方案需要两步：

1. **全量同步联系人**：从解密的 `contact.db` 读 `local_type=1` 的真人好友，排除 `gh_*`/`@openim`/`@chatroom`/系统账号，按 `wechat_id` 去重后批量插入 CRM
2. **改名**：从 `contact.db` 的 `remark`（备注名优先）或 `nick_name` 同步到 CRM 的 `name` 字段，替换掉 `wxid_*` 和 `@openim` 形式的名字

详见 `wechat-import-personal-crm.md` 的「联系人全量同步」和「联系人改名」章节。

### 重复联系人合并（2026-06-19 新增）

`wechat_sync.py` 的 `create_contact` 函数没有先查 wechat_id 是否已存在，多次导入会产生重复联系人（实测 203 个重复）。合并策略：保留每个 wechat_id 下消息最多的 id，把重复 id 的 chat_messages 和 contact_records 转移到保留 id，然后删除重复联系人。

详见 `wechat-import-personal-crm.md` 的「重复联系人合并」章节。

### 批量匹配优化

不逐条消息查询，而是预先构建映射表：

```python
# 步骤 1：提取所有唯一的 counterpart_wxid
unique_wxids = set(msg["counterpart_wxid"] for msg in messages)

# 步骤 2：批量查询所有联系人，构建映射表
contacts = db.query(Contact).filter(Contact.wechat_id.in_([w.lower() for w in unique_wxids])).all()
wxid_map = {c.wechat_id.lower().strip(): c.id for c in contacts}

# 步骤 3：对未匹配的 wxid 做 name 降级匹配
unmatched = set(unique_wxids) - set(wxid_map.keys())
name_contacts = db.query(Contact).filter(Contact.name.in_(list(unmatched))).all()
name_map = {c.name: c.id for c in name_contacts}

# 步骤 4：合并映射表
match_map = {**wxid_map, **name_map}
```

## 去重策略

### 去重键定义

使用 `contact_id + sent_at + content` 三字段组合作为去重键：

```python
# 正确的去重方式（不截断内容）
existing = db.query(ChatMessage).filter(
    ChatMessage.contact_id == contact_id,
    ChatMessage.sent_at == msg.sent_at,
    ChatMessage.content == msg.content
).first()

# ❌ 错误的去重方式（现有管道 1 的 [:50] 截断）
# ContactRecord 的去重键是 contact_id + date + content[:50]
# 这会导致：两条内容前 50 字相同但后续不同的消息被误判为重复
```

### 去重优先级

- **ChatMessage 表**：分析系统依赖的数据源，去重键必须精确（contact_id + sent_at + content）
- **ContactRecord 表**：Dashboard 依赖的数据源，去重键为 `contact_id + contact_date + contact_type + direction + content`（5 字段组合键），确保同一天同一联系人的多条不同消息不被误判为重复

### 批量去重优化

```python
# 步骤 1：查询该联系人已有的所有 ChatMessage
existing_msgs = db.query(ChatMessage).filter(
    ChatMessage.contact_id == contact_id
).all()

# 步骤 2：构建去重集合（用 set 加速查找）
existing_keys = {(m.sent_at, m.content) for m in existing_msgs}

# 步骤 3：过滤新消息
new_msgs = [m for m in messages if (m.sent_at, m.content) not in existing_keys]
```

## 执行规则

1. **映射表必须预构建**：不能逐条消息查询数据库，必须先批量查询再逐条映射
2. **wechat_id 清理规则**：去除空格、特殊字符后 lowercase 比对，兼容 "wxid_abc" 和 "WXID_ABC" 等变体
3. **未匹配联系人默认选项 B（忽略）**：避免自动创建大量低质量联系人记录，除非调用者明确选择选项 A
4. **去重键不截断内容**：content 必须完整参与去重比较，不能用 [:50] 截断
5. **sent_at 时间精度一致**：去重比较时时间戳必须精确到秒，不能降级到 date-only

## 必须产出

```yaml
outputs:
  required:
    - wxid → contact_id 映射表
    - 已去重的新消息列表（标记 contact_id）
    - 未匹配 wxid 列表

  optional:
    - 被去重过滤掉的重复消息数量统计
    - 未匹配联系人的建议处理方式
```

## 质量标准

```yaml
quality:
  match_accuracy: "wechat_id 精确匹配优先，name 降级匹配仅在 wechat_id 未匹配时使用"
  dedup_precision: "去重键必须使用完整 content + sent_at，不截断"
  batch_performance: "必须预构建映射表，不能逐条查询数据库"
  unmatched_handling: "明确区分已匹配和未匹配消息，未匹配消息不丢失"
```

## 示例

### 输入

```json
{
  "messages": [
    {"direction": "received", "content": "项目进展怎么样", "msg_type": "text", "sent_at": "2025-02-19T08:00:00+08:00", "counterpart_wxid": "wxid_abc"},
    {"direction": "sent", "content": "还不错", "msg_type": "text", "sent_at": "2025-02-19T08:01:00+08:00", "counterpart_wxid": "wxid_abc"},
    {"direction": "received", "content": "项目进展怎么样", "msg_type": "text", "sent_at": "2025-02-19T08:00:00+08:00", "counterpart_wxid": "wxid_abc"}
  ],
  "match_mode": "ignore_unmatched"
}
```

### 输出

```json
{
  "match_map": {"wxid_abc": 15},
  "matched_messages": [
    {"contact_id": 15, "direction": "received", "content": "项目进展怎么样", "msg_type": "text", "sent_at": "2025-02-19T08:00:00+08:00"},
    {"contact_id": 15, "direction": "sent", "content": "还不错", "msg_type": "text", "sent_at": "2025-02-19T08:01:00+08:00"}
  ],
  "dedup_count": 1,
  "unmatched_wxids": [],
  "unmatched_count": 0
}
```

第一条和第三条消息的 (sent_at, content) 完全相同，第三条被去重过滤。第二条内容不同，保留。

## 常用命令

```bash
# 检查联系人 wechat_id 字段数据
sqlite3 /path/to/crm.db "SELECT id, name, wechat_id FROM contacts WHERE wechat_id IS NOT NULL LIMIT 10;"

# 检查 ChatMessage 去重键
sqlite3 /path/to/crm.db "SELECT contact_id, sent_at, content FROM chat_messages WHERE contact_id=15 ORDER BY sent_at DESC LIMIT 10;"

# 统计未匹配的 wxid
python3 -c "
import json
msgs = json.load(open('wechat_export.json'))['messages']
wxids = set(m['wxid'] for m in msgs if not m['is_sender'])
print(f'Unique counterparts: {len(wxids)}')
print(f'wxids: {wxids}')
"
```
