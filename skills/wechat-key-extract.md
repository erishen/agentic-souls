# WeChat Key Extract Skill

> 从 Mac 微信本地数据库提取解密密钥

## Skill 定义

```yaml
name: wechat-key-extract
description: 从 Mac 微信进程内存和 key_info.db 提取 SQLCipher 解密密钥
version: 1.0.0
category: implementation
```

## 能力范围

```yaml
capabilities:
  - 从 key_info.db 的 LoginKeyInfoTable.key_info_data BLOB 尝试多种偏移/长度组合提取密钥
  - 通过 ad-hoc resign WeChat.app 解除 hardened runtime 限制
  - 通过 task_for_pid 获取微信进程内存访问权限
  - 扫描进程内存中 x'<hex>' 格式的密钥模式
  - 使用 HMAC-SHA512 验证密钥对 (key+salt) 的有效性
  - 支持多种扫描策略(二进制、LLDB、Frida)

limitations:
  - 需要 sudo/root 权限运行内存扫描器
  - 需要先 ad-hoc resign WeChat.app（会杀掉微信进程）
  - Mac WeChat ≥4.1.7 的密钥存储方式可能有变动，社区工具未及时适配（注：2026-06-19 实测密钥提取实际是成功的，问题在解密方法，详见「已知问题」）
  - 扫描结果依赖 WeChat 版本，不同版本可能需要不同的扫描策略
```

## 准备工作

### 确认环境

```bash
# 1. 确认 sqlcipher 已安装
which sqlcipher

# 2. 确认微信路径
ls ~/Library/Containers/com.tencent.xinWeChat/Data/Documents/xwechat_files/

# 3. 确认解密工具套件
ls /tmp/wechat-decrypt/
```

### 微信登录信息

从 `.env` 获取（`personal/personal-crm/.env`）:

| 变量 | 说明 | 示例值 |
|------|------|--------|
| `WECHAT_BASE` | 微信数据根目录 | `~/Library/Containers/.../xwechat_files` |
| `WECHAT_DB_SUFFIX` | 用户目录后缀 | `erishen_c472` |
| `WECHAT_LOGIN_USER` | 系统用户名 | `erishen` |

### 确认数据库文件存在

```bash
# 消息数据库（加密状态，SQLite format 3 以外的文件头）
ls ~/Library/Containers/com.tencent.xinWeChat/Data/Documents/xwechat_files/*/db_storage/message/

# 联系人数据库
ls ~/Library/Containers/com.tencent.xinWeChat/Data/Documents/xwechat_files/*/db_storage/contact/contact.db

# key_info.db
ls ~/Library/Containers/com.tencent.xinWeChat/Data/Documents/xwechat_files/all_users/login/*/key_info.db
```

## 扫描策略

### 策略 1：自动流程脚本

`wechat_key_extract.sh` 一键完成所有步骤：

```bash
# 该脚本自动执行：退出微信 → resign → 启动微信 → 扫描密钥 → 保存
osascript -e 'do shell script "bash personal/personal-crm/backend/wechat_key_extract.sh" with administrator privileges'
```

脚本流程：
1. 杀掉微信进程
2. 备份 `/Applications/WeChat.app` 到 `WeChat_original_backup.app`（仅首次）
3. `codesign -s - --force --deep` 做 ad-hoc 签名
4. 重新启动微信
5. 编译并运行 `find_all_keys_macos.c` 内存扫描器
6. 保存密钥到 `all_keys.json`

### 策略 2：Python 内存扫描

```bash
# 手动编译 C 扫描器
cc -O2 -o /tmp/find_all_keys_macos /tmp/wechat-decrypt/find_all_keys_macos.c -framework Foundation

# 运行扫描
sudo /tmp/find_all_keys_macos $(pgrep -x WeChat)
```

### 策略 3：Python 版本扫描

```bash
# 使用 Python 版本的扫描器
sudo python3 /tmp/wechat-decrypt/find_all_keys_macos.py
```

### 策略 4：LLDB 搜索

```bash
# 用 lldb 附加到微信进程搜索密钥
sudo lldb -b -o "process attach -p $(pgrep -x WeChat)" \
  -o "command script import /tmp/wechat-decrypt/lldb_search3.py" \
  -o "detach" -o "quit"
```

### 策略 5：HMAC 验证的 key_info 深度分析

```python
# 使用 deep_keyinfo_analysis.py 验证所有可能的密钥偏移
python3 /tmp/wechat-decrypt/deep_keyinfo_analysis.py

# 使用 try_keyinfo.py 尝试不同组合
python3 /tmp/wechat-decrypt/try_keyinfo.py
```

### 策略 6：Frida hook

```bash
# 使用 Frida hook CCCrypt 函数（如果安装了 Frida）
sudo frida -p $(pgrep -x WeChat) -l /tmp/wechat-decrypt/frida_hook_cccrypt.py
```

## 已知问题

### 2026-06-19 修正：密钥提取实际是成功的，问题在解密方法

之前的 skill 文档错误地认为「Mac WeChat 4.1.7 密钥提取失败」，实际上：

1. **密钥提取成功**：`wechat_key_extract.sh` + C 扫描器 `find_all_keys_macos` 在 4.1.7 上正常工作，能从微信进程内存提取出 22 个数据库的 `(enc_key, salt)` 对，写入 `all_keys.json`
2. **真正的瓶颈是解密方法**：`wechat_msg_extract.py` 的 `decrypt_sqlcipher` 函数用 sqlcipher CLI 的 `PRAGMA key = "x'<hex>'"` 方式解密，这会让 SQLCipher 走 PBKDF2 KDF 流程。但微信 4.x 用的是**已经派生好的 raw key + 独立 salt**（从内存直接提取），不需要再 KDF。所以 sqlcipher CLI 解密必然失败
3. **正确解密方式**：用 `/tmp/wechat-decrypt/decrypt_db.py`（pycryptodome 直接 AES-256-CBC 逐页解密 + HMAC-SHA512 校验），不走 KDF。实测 21 个数据库全部解密成功，1.1GB 数据

### 历史扫描结果（2026-06 早期尝试，已过时）

| 策略 | 状态 | 结果 |
|------|------|------|
| C 内存扫描器 | ✅ 执行成功 | 扫描但未找到密钥 |
| Python 内存扫描器 | ⚠️ task_for_pid 失败 | 需要 root + ad-hoc |
| key_info BLOB 分析 | ⚠️ HMAC 验证失败 | 可能使用了新的密钥派生 |
| LLDB 搜索 | ⚠️ 解析错误 | 找到模式但验证失败 |
| Frida hook | ❌ 未测试 | 需要安装 Frida |

上述表里「C 内存扫描器未找到密钥」的结论是错误的 —— 重新跑 `wechat_key_extract.sh` 后 C 扫描器成功提取了全部 22 个密钥。早期失败可能是 ad-hoc 签名没生效或微信没正常重启。

### 2026-06-19 实测完整流程（成功）

```bash
# 1. 提取密钥（需要 sudo，会重启微信）
cd personal/personal-crm/backend
sudo bash wechat_key_extract.sh
# → 生成 wechat_keys/all_keys.json，22 个密钥

# 2. 解密数据库（用 pycryptodome 方式，不用 sqlcipher CLI）
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
# → 35万条消息（含公众号、群聊、真人单聊）

# 4. 过滤真人单聊后导入 CRM
uv run python wechat_sync.py --source json \
  --file /tmp/wechat_real_personal.json \
  --api-base http://localhost:8000/api \
  --match-mode create_contact
# → 253 个联系人，10840 条 ChatMessage，10207 条 ContactRecord
```

### wechat_msg_extract.py 的修复（2026-06-19）

原版 `wechat_msg_extract.py` 有两个 bug 导致提取的消息全部 `counterpart_wxid` 为空：

1. **`build_name_map` 找不到 id 列**：WeChat 4.x 的 `Name2Id` 表只有 `user_name` + `is_session` 两列，没有显式 id 列。原代码用 `pick(cols, ["id", "rowid", ...])` 找 id 列失败就返回空 dict。修复：用 `rowid` 兜底（`real_sender_id` 实际指向 `Name2Id` 的 rowid）
2. **`is_sent` 用 `local_type == 1` 误判方向**：实测同一会话里 `local_type=1` 既可能是自己也可能是对方。修复：优先用 `real_sender_id` 对比 `my_wxid` 在 `Name2Id` 里的 rowid 判断方向
3. **新增 `build_md5_to_wxid_map`**：WeChat 4.x 把每个会话的消息单独存一张 `Msg_<md5(wxid)>` 表，表名里的 md5 能反查出会话对端 wxid，作为 `counterpart_wxid` 的兜底来源

## 替代方案

如果扫描失败，可行的替代数据获取方式：

1. **微信桌面端导出聊天记录**：在微信聊天窗口右键 → 导出聊天记录 → 选择导出为 txt 文件
2. **导出后的 txt 文件**：包含聊天内容、时间、发送者，可通过 `wechat_sync.py --source txt` 导入（需要先写 txt 解析器）
3. **手机端备份**：通过 iTunes 备份后提取微信数据（需要额外工具）

## 执行规则

1. **必须先备份微信**：`wechat_key_extract.sh` 会自动备份 `WeChat.app`，恢复命令：
   ```bash
   cp -R /Applications/WeChat_original_backup.app /Applications/WeChat.app
   ```
2. **扫描时微信必须在运行**：且必须已成功 ad-hoc 签名
3. **密钥文件路径**：`wechat_keys/all_keys.json` 生成在 `backend/wechat_keys/` 目录（已被 .gitignore 排除）
4. **密钥格式**：`{"db_rel_path": {"enc_key": "hex_key", "salt": "hex_salt"}}`
5. **验证方法**：对加密的 `.db` 文件用 `hex_key` 执行 `sqlcipher` 解密，如果能读到表则密钥正确

## 必须产出

```yaml
outputs:
  required:
    - all_keys.json（包含所有可解密数据库的密钥对）
    - 扫描日志（成功/失败的原因）
    
  optional:
    - 扫描的内存区域统计
    - 找到未通过 HMAC 验证的候选密钥
```

## 质量标准

```yaml
quality:
  key_verification: "每对密钥必须通过 HMAC-SHA512 验证或实际 sqlcipher 解密验证"
  db_coverage: "至少覆盖 message/*.db 和 contact/contact.db"
  nondestructive: "不修改任何微信原始数据文件"
```
