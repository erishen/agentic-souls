# WeChat Sync Pipeline Skill

> 微信数据同步流水线技能——解决数据割裂，统一双表写入

## Skill 定义

```yaml
name: wechat-sync-pipeline
description: 将微信聊天记录同步到 personal-crm，实现 ChatMessage + ContactRecord 双表写入，解决数据割裂问题
version: 1.0.0
category: implementation
```

## 能力范围

```yaml
capabilities:
  - 增量同步：基于 sent_at 时间戳，只同步新增消息
  - 双表写入：ChatMessage（供分析系统）+ ContactRecord 摘要（供 Dashboard）
  - 同步状态记录：记录最后同步时间戳，下次从该点继续
  - 错误隔离：单条消息失败不阻塞整批
  - 批量导入性能优化

limitations:
  - 不处理微信数据提取（由 wechat-data-extract skill 处理）
  - 不处理联系人匹配（由 wechat-contact-match skill 处理）
  - 不修改 personal-crm 现有管道代码
  - 不涉及实时监控或 WebSocket 推送
```

## 数据割裂问题与解决方案

### 当前问题

personal-crm 有两条独立管道，数据互不可见：

| 管道 | 写入表 | 消费方 | 缺失 |
|------|--------|--------|------|
| 管道 1（PyWxDump） | ContactRecord | Dashboard、关系评分 | 分析系统看不到（只存 date、截断 [:50]） |
| 管道 2（文件上传） | ChatMessage | BRM 七元素、弱连接、AI 深度分析 | Dashboard 看不到（ContactRecord 无数据） |

### 解决方案：双表写入

同步流水线统一写入 **ChatMessage 表**（分析系统的数据源），同时从 ChatMessage **派生 ContactRecord 摘要**（Dashboard 的数据源）。一次同步，两方都可见。

```
微信原始数据 → wechat-data-extract 标准化
             → wechat-contact-match 匹配+去重
             → wechat-sync-pipeline 双表写入
                 ├→ ChatMessage（完整时间戳、完整内容、消息类型）
                 └→ ContactRecord 摘要（从 ChatMessage 派生）
```

### ContactRecord 洘生规则

从 ChatMessage 派生 ContactRecord 时，遵循以下映射：

| ChatMessage 字段 | ContactRecord 字段 | 转换规则 |
|------------------|---------------------|----------|
| `sent_at` (datetime) | `contact_date` (date) | 取 sent_at 的日期部分。**注意 `wechat_sync.py` 当前实现取的是 UTC 日期，不是 Asia/Shanghai 日期** —— 因为脚本派生时循环用的是 `convert_sent_at_for_api` 转换后的 UTC 字符串（如 `"2025-06-19 09:20:00"`）。对上海时间 0:00–8:00 的消息，ContactRecord.contact_date 会比实际上海日期早一日 |
| `msg_type` | `contact_type` | 固定为 "微信" |
| `content` | `content` | 保留完整内容，不截断（注：schema 定义为"内容摘要"，但同步流水线写入完整内容，这是对现有字段的扩展使用，确保数据不丢失） |
| `direction` | `direction` | 直接映射（sent/received/unknown） |
| - | `mood` | 留空（mood 需人工判断或 AI 分析） |
| - | `follow_up_date` | 留空（follow_up 需人工设置） |

**ContactRecord.contact_date 时区修复**（推荐）：派生前把 sent_at 重新按上海时区解析一次，确保 contact_date 落在实际上海日期：

```python
from datetime import timezone, timedelta
SHANGHAI_TZ = timezone(timedelta(hours=8))
# msg['sent_at'] 此刻是已转换的 UTC 字符串，如 "2025-06-19 09:20:00"
sent_at_dt = datetime.fromisoformat(msg['sent_at']).replace(tzinfo=timezone.utc).astimezone(SHANGHAI_TZ)
contact_date = sent_at_dt.date().isoformat()  # 上海日期
```

**同一天同一联系人的多条消息**：每条消息生成一条 ContactRecord（与现有管道 1 行为一致）。不合并为一条摘要，因为 Dashboard 的联系频率计算依赖每条记录的粒度。

## 增量同步策略

### 基于时间戳的增量同步

每次同步完成后，记录**最后同步时间戳**（last_sync_timestamp）。下次同步只提取 sent_at > last_sync_timestamp 的消息。

```python
# 同步前：获取最后同步时间
last_sync = get_last_sync_timestamp(contact_id)
if last_sync:
    # 只提取新增消息
    new_messages = [m for m in messages if m["sent_at"] > last_sync]
else:
    # 首次同步：全部导入
    new_messages = messages

# 同步后：更新时间戳
update_last_sync_timestamp(contact_id, max(m["sent_at"] for m in imported_messages))
```

### 同步状态记录方式

在 ChatImport 表记录每次同步的元数据：

| 字段 | 值 | 说明 |
|------|-----|------|
| `contact_id` | 被同步的联系人 ID | |
| `source` | "wechat_sync" | 区分手动上传和自动同步 |
| `file_name` | "sync_{timestamp}" | 标记为同步导入 |
| `total_messages` | 本次导入的消息数 | |
| `date_range_start` | 最早消息日期 | |
| `date_range_end` | 最新消息日期 | 用于下次增量同步的起点 |
| `status` | "completed" / "partial" | partial 表示有失败条目 |

### 首次同步 vs 增量同步

| 情景 | last_sync_timestamp | 行为 |
|------|---------------------|------|
| 首次同步 | None | 导入全部消息 |
| 增量同步 | 有值 | 只导入 sent_at > last_sync 的消息 |
| 全量重新同步 | 被清除 | 清除后下次导入全部 |

## 双表写入流程

```
步骤 1：创建 ChatImport 批次记录
步骤 2：批量写入 ChatMessage（核心数据）
步骤 3：从 ChatMessage 派生 ContactRecord（摘要数据）
步骤 4：更新 ChatImport 状态为 completed
步骤 5：记录 last_sync_timestamp
```

### 步骤 2：ChatMessage 批量写入

**去重键**：`contact_id + sent_at + content`（三字段组合键，不截断 content）

写入前必须做去重检查，防止重复导入：

```python
# 去重检查：查询该联系人已有的消息集合
existing_keys = {(m.sent_at, m.content) for m in 
    db.query(ChatMessage).filter(ChatMessage.contact_id == contact_id).all()}

# 过滤掉已存在的消息
new_msgs = [m for m in new_messages if (m["sent_at"], m["content"]) not in existing_keys]
```

批量插入优化：

```python
# 批量插入优化
batch_size = 500  # 每 500 条提交一次
for i in range(0, len(new_messages), batch_size):
    batch = new_messages[i:i+batch_size]
    db.bulk_save_mappings(
        ChatMessage.__table__,
        [{"import_id": import_id, "contact_id": m["contact_id"],
          "direction": m["direction"], "content": m["content"],
          "msg_type": m["msg_type"], "sent_at": m["sent_at"]} for m in batch]
    )
    db.commit()
```

### 步骤 3：ContactRecord 派生写入

```python
for msg in new_messages:
    contact_date = msg["sent_at"].date()  # 取日期部分
    record = ContactRecord(
        contact_id=msg["contact_id"],
        contact_date=contact_date,
        contact_type="微信",
        content=msg["content"],
        direction=msg["direction"]
    )
    # 去重检查：同一天同联系人同内容的记录不重复写入
    existing = db.query(ContactRecord).filter(
        ContactRecord.contact_id == msg["contact_id"],
        ContactRecord.contact_date == contact_date,
        ContactRecord.contact_type == "微信",
        ContactRecord.direction == msg["direction"],
        ContactRecord.content == msg["content"]
    ).first()
    if not existing:
        db.add(record)
```

## 错误处理

### 单条消息失败不阻塞整批

```python
success_count = 0
fail_count = 0
failed_messages = []

for msg in new_messages:
    try:
        # 写入 ChatMessage + ContactRecord
        write_message_pair(msg)
        success_count += 1
    except Exception as e:
        fail_count += 1
        failed_messages.append({"content": msg["content"][:100], "error": str(e)})
        continue  # 不阻塞，继续处理下一条

# 记录结果
import_record.status = "partial" if fail_count > 0 else "completed"
import_record.total_messages = success_count
```

### 错误记录

- 单条失败的消息记入 failed_messages 列表
- ChatImport 状态标记为 "partial"（有失败）或 "completed"（全部成功）
- partial 状态不影响后续增量同步——下次同步从 last_sync_timestamp 继续，不会重试已成功的消息

## 执行规则

1. **ChatMessage 是主表**：所有消息必须先写入 ChatMessage，ContactRecord 是派生数据
2. **时间戳不降级**：ChatMessage 保留完整 sent_at (datetime)，ContactRecord 取 sent_at 的 date 部分
3. **内容不截断**：ChatMessage.content 和 ContactRecord.content 都保留完整消息内容
4. **增量同步优先**：避免每次全量重新导入，只在缓存未命中时做全量
5. **同步状态持久化**：last_sync_timestamp 存在 ChatImport 的 date_range_end 字段，下次增量同步从这里开始
6. **partial 状态不阻塞后续**：单条失败不影响整体流程，下次增量同步继续正常工作
7. **脚本侧预去重当前失效**（实测 2026-06-19）：`wechat_sync.py` 的 `get_last_sync_timestamp` 和 `sync_to_crm` 都调用了 `GET /api/chat/messages/{contact_id}`，但 `app/routers/chat.py` 在该路径只注册了 DELETE，没有 GET，返回 405。脚本用 `if resp.status_code == 200` 静默吞错，所以 `existing_keys` 永远是空集，脚本日志里"跳过 0 条重复"不可信 —— 真实去重靠 `/api/chat/import/{contact_id}` 内部按 `content + sent_at` 的去重兜底。修复方向：在 `chat.py` 增加 GET endpoint，或改脚本走其他查询路径

## 必须产出

```yaml
outputs:
  required:
    - ChatMessage 记录（写入 chat_messages 表）
    - ContactRecord 洘生记录（写入 contact_records 表）
    - ChatImport 批次记录（写入 chat_imports 表）
    - 同步结果摘要（success_count, fail_count, failed_messages）

  optional:
    - 同步日志（时间、消息数、耗时）
    - 增量同步起点记录
```

## 质量标准

```yaml
quality:
  data_consistency: "ChatMessage 和 ContactRecord 数据对齐——同一条消息在两张表都可见"
  incremental_efficiency: "增量同步只导入新增消息，不重复导入已存在的消息"
  error_isolation: "单条消息失败不阻塞整批导入"
  dedup_accuracy: "去重键使用完整 content + sent_at，不截断"
  timestamp_preservation: "ChatMessage 保留完整 datetime，ContactRecord 降级为 date"
```

## 示例

### 输入：已匹配+已去重的消息列表

```json
{
  "messages": [
    {"contact_id": 15, "direction": "received", "content": "项目进展怎么样", "msg_type": "text", "sent_at": "2025-02-19T08:00:00+08:00"},
    {"contact_id": 15, "direction": "sent", "content": "还不错，下周出报告", "msg_type": "text", "sent_at": "2025-02-19T08:01:00+08:00"}
  ],
  "last_sync_timestamp": "2025-02-18T00:00:00+08:00",
  "contact_id": 15
}
```

### 输出：同步结果

```json
{
  "import_id": 42,
  "contact_id": 15,
  "success_count": 2,
  "fail_count": 0,
  "failed_messages": [],
  "status": "completed",
  "date_range_start": "2025-02-19",
  "date_range_end": "2025-02-19",
  "chat_messages_written": 2,
  "contact_records_written": 2,
  "last_sync_timestamp": "2025-02-19T08:01:00+08:00"
}
```

两张表各写入 2 条记录。ChatMessage 保留完整时间戳，ContactRecord 取日期部分。下次增量同步从 "2025-02-19T08:01:00+08:00" 开始。

## 常用命令

```bash
# 检查 ChatImport 同步状态
sqlite3 /path/to/crm.db "SELECT id, contact_id, source, total_messages, date_range_end, status FROM chat_imports WHERE source='wechat_sync' ORDER BY id DESC LIMIT 5;"

# 检查 ChatMessage 写入结果
sqlite3 /path/to/crm.db "SELECT id, contact_id, direction, content, msg_type, sent_at FROM chat_messages WHERE contact_id=15 ORDER BY sent_at DESC LIMIT 10;"

# 检查 ContactRecord 洘生写入结果
sqlite3 /path/to/crm.db "SELECT id, contact_id, contact_date, contact_type, direction, content FROM contact_records WHERE contact_id=15 AND contact_type='微信' ORDER BY contact_date DESC LIMIT 10;"

# 验证双表数据对齐
sqlite3 /path/to/crm.db "
SELECT 'chat_messages' as table, count(*) FROM chat_messages WHERE contact_id=15;
SELECT 'contact_records' as table, count(*) FROM contact_records WHERE contact_id=15 AND contact_type='微信';
"
```
