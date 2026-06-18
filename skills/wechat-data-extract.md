# WeChat Data Extract Skill

> 从微信数据源提取聊天记录的标准化技能

## Skill 定义

```yaml
name: wechat-data-extract
description: 从微信数据源提取聊天记录，标准化为 ChatMessage 格式
version: 1.0.0
category: implementation
```

## 能力范围

```yaml
capabilities:
  - 解析 PyWxDump 导出的 JSON 格式微信聊天数据
  - 解析 CSV/JSON 格式的微信聊天记录导出文件
  - 消息类型映射（微信数字编码 → 字符串类型）
  - 时间戳标准化（多种格式 → datetime with timezone）
  - 方向字段转换（is_sender → sent/received）
  - 输出标准化的 ChatMessage 格式数据

limitations:
  - 不处理微信数据库直接解密（由 wechat_export.py 单独处理）
  - 不处理 OCR 截屏识别（由 wechat_ocr_import.py 单独处理）
  - 不涉及联系人匹配（由 wechat-contact-match skill 处理）
  - 不涉及数据写入 personal-crm（由 wechat-sync-pipeline skill 处理）
```

## 数据格式规范

### 输入格式 1：PyWxDump JSON

PyWxDump 导出的数据结构：

```json
{
  "messages": [
    {
      "wxid": "wxid_sender123",
      "content": "明天下午见面吧",
      "msg_type": 1,
      "create_time": 1740000000,
      "is_sender": false
    }
  ],
  "my_wxid": "wxid_myself456"
}
```

字段映射规则：

| PyWxDump 字段 | ChatMessage 字段 | 转换规则 |
|----------------|-------------------|----------|
| `wxid` | → 用于联系人匹配（不直接写入 ChatMessage） | is_sender=true 时 wxid 是自己的，对方 wxid 需从 my_wxid 推断 |
| `my_wxid` | → 推断对方 wxid | is_sender=false 时对方=wxid，is_sender=true 时对方从上下文推断 |
| `content` | `content` | Text 类型直接映射；非文字类型内容可为空或保留文件路径 |
| `msg_type` | `msg_type` | 数字 → 字符串映射（见下表） |
| `create_time` | `sent_at` | Unix 时间戳（秒）→ datetime with Asia/Shanghai timezone |
| `is_sender` | `direction` | true → "sent"，false → "received" |

消息类型映射：

| 微信 msg_type | ChatMessage msg_type | 说明 |
|---------------|----------------------|------|
| 1 | text | 文字消息 |
| 3 | image | 图片 |
| 34 | voice | 语音 |
| 43 | video | 视频 |
| 47 | sticker | 表情包 |
| 49 | file | 文件 |
| 10000 | system | 系统消息（红包、群通知等） |

### 输入格式 2：CSV/JSON 文件上传

支持灵活字段名，自动识别并映射：

| 可能的字段名 | ChatMessage 字段 | 说明 |
|--------------|-------------------|------|
| `timestamp`, `time`, `create_time`, `sent_at` | `sent_at` | 支持 Unix 秒/毫秒、ISO 格式、datetime 对象 |
| `sender`, `from`, `wxid` | → 用于联系人匹配 | 发送者标识 |
| `content`, `text`, `message` | `content` | 消息内容 |
| `type`, `msg_type` | `msg_type` | 数字→字符串映射，或直接字符串 |
| `direction`, `is_sender` | `direction` | "sent"/"received"、true/false、"outgoing"/"incoming" 均支持 |
| `日期`, `发送者`, `内容` | 同上 | 中文字段名也支持 |

### 输出格式：标准化 ChatMessage 数据

> **注意**：`sender_wxid` 和 `counterpart_wxid` 是中间字段，供下游 `wechat-contact-match` skill 将 wxid 映射为 `contact_id`。它们不会直接写入 ChatMessage 表。经过 contact-match 处理后，这两个字段会被替换为 `contact_id` 字段，然后再进入 `wechat-sync-pipeline` 的双表写入流程。

```json
{
  "direction": "sent",
  "content": "明天下午见面吧",
  "msg_type": "text",
  "sent_at": "2025-02-19T15:00:00+08:00",
  "sender_wxid": "wxid_myself456",
  "counterpart_wxid": "wxid_sender123"
}
```

## 执行规则

1. **只保留文字和有分析价值的消息类型**：text、system 保留；image、voice、video、sticker、file 保留记录但内容可标记为 "[图片]"、"[语音]" 等占位符
2. **时间戳必须保留完整精度**：不能用 date-only 丢失时分秒信息（现有管道 1 的 date-only 是错误设计）
3. **内容不做截断**：完整保留消息内容，不用 [:50] 截断（现有管道 1 的截断是错误设计）
4. **内容少于 2 字符的消息可过滤**：单字回复（"好"、"嗯"）在分析维度上价值低，但建议保留不做过滤，由下游分析引擎决定权重
5. **is_sender 推断 counterpart**：当 is_sender=true 时，该消息是"我发的"，对方 wxid 需从聊天上下文推断（通常从其他 is_sender=false 消息的 wxid 获取）；当 is_sender=false 时，对方 wxid 就是 msg.wxid

## 必须产出

```yaml
outputs:
  required:
    - 标准化的 ChatMessage 格式数据列表
    - 消息类型映射表
    
  optional:
    - 被过滤掉的原始消息列表（用于审计）
    - 解析失败的原始数据（用于调试）
```

## 质量标准

```yaml
quality:
  field_mapping: "所有字段必须正确映射到 ChatMessage schema"
  timestamp_precision: "时间戳必须保留完整 datetime，不能降级为 date-only"
  content_preservation: "消息内容完整保留，不做截断"
  type_coverage: "7 种微信消息类型全部有映射规则"
```

## 示例

### 输入：PyWxDump JSON

```json
{
  "messages": [
    {"wxid": "wxid_abc", "content": "项目进展怎么样", "msg_type": 1, "create_time": 1740000000, "is_sender": false},
    {"wxid": "wxid_me", "content": "还不错，下周出报告", "msg_type": 1, "create_time": 1740000060, "is_sender": true},
    {"wxid": "wxid_abc", "content": "[图片]", "msg_type": 3, "create_time": 1740000120, "is_sender": false}
  ],
  "my_wxid": "wxid_me"
}
```

### 输出：标准化数据

```json
[
  {
    "direction": "received",
    "content": "项目进展怎么样",
    "msg_type": "text",
    "sent_at": "2025-02-19T08:00:00+08:00",
    "sender_wxid": "wxid_abc",
    "counterpart_wxid": "wxid_abc"
  },
  {
    "direction": "sent",
    "content": "还不错，下周出报告",
    "msg_type": "text",
    "sent_at": "2025-02-19T08:01:00+08:00",
    "sender_wxid": "wxid_me",
    "counterpart_wxid": "wxid_abc"
  },
  {
    "direction": "received",
    "content": "[图片]",
    "msg_type": "image",
    "sent_at": "2025-02-19T08:02:00+08:00",
    "sender_wxid": "wxid_abc",
    "counterpart_wxid": "wxid_abc"
  }
]
```

## 常用命令

```bash
# 检查 PyWxDump 导出数据格式
cat wechat_export.json | python3 -c "import json,sys; d=json.load(sys.stdin); print(f'messages: {len(d.get(\"messages\",[]))}, keys: {list(d.keys())}')"

# 验证时间戳转换
python3 -c "from datetime import datetime, timezone, timedelta; ts=1740000000; tz=timezone(timedelta(hours=8)); print(datetime.fromtimestamp(ts, tz=tz).isoformat())"

# 检查 ChatMessage schema 字段
grep -A 10 'class ChatMessage' personal-crm/backend/app/models.py
```
