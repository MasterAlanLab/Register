# OpenAI 自动注册说明书

## 1. 功能概览
这个脚本支持下面几件事：

- 自动注册 OpenAI 账号
- 保存账号密码和 token 到本地
- 注册成功后自动上传到 Sub2API
- 注册成功后自动上传到 CPA
- 清理 Sub2API 中 `training_set_failed` 账号
- 控制 Sub2API / CPA 池子数量，达到目标后暂停注册

---

## 2. 环境准备
基础功能（注册 / Sub2API 上传 / Sub2API 清理）只需要：

```bash
cd openai-register
uv sync
```

如果还要启用 CPA 清理（`--cpa-clean`，依赖 `aiohttp`），再安装可选依赖：

```bash
cd openai-register
uv sync --extra cpa
```

---

## 3. 最简单的用法
只注册一次：

```bash
cd openai-register
uv run python openai_register.py --once
```

指定邮箱提供商：

```bash
uv run python openai_register.py --mail-provider luckmail --once
uv run python openai_register.py --mail-provider tempmail --once
uv run python openai_register.py --mail-provider gptmail --once
```

`--mail-provider auto` 为默认值，顺序是：

1. LuckMail
2. TempMail.lol
3. GPTMail

---

## 4. 邮箱配置

### 4.1 LuckMail
默认内置：

- `LUCKMAIL_BASE_URL=https://mails.luckyous.com`

一般至少配置：

```bash
export LUCKMAIL_API_KEY="YOUR_LUCKMAIL_API_KEY"
```

完整示例：

```bash
export LUCKMAIL_API_KEY="YOUR_LUCKMAIL_API_KEY"
export LUCKMAIL_API_SECRET="YOUR_LUCKMAIL_API_SECRET"
export LUCKMAIL_USE_HMAC="false"
export LUCKMAIL_PROJECT_CODE="openai"
export LUCKMAIL_EMAIL_TYPE="ms_graph"
export LUCKMAIL_DOMAIN="outlook.com"
export LUCKMAIL_ORDER_TIMEOUT="180"
export LUCKMAIL_POLL_INTERVAL="6"

uv run python openai_register.py --mail-provider luckmail --once
```

如果你使用的是别的 LuckMail 站点，再额外覆盖：

```bash
export LUCKMAIL_BASE_URL="https://your-luckmail.example.com"
```

---

## 5. Sub2API 用法

### 5.1 注册成功后自动上传
推荐使用全局 Admin API Key：

```bash
cd openai-register
uv run python openai_register.py \
  --sub2api-base-url http://203.0.113.10:8080 \
  --sub2api-admin-api-key YOUR_SUB2API_ADMIN_API_KEY \
  --sub2api-group-ids 2 \
  --sub2api-upload \
  --prune-local \
  --once
```

也可以用环境变量：

```bash
export SUB2API_BASE_URL="http://203.0.113.10:8080"
export SUB2API_ADMIN_API_KEY="YOUR_SUB2API_ADMIN_API_KEY"
export SUB2API_GROUP_IDS="2"
export AUTO_UPLOAD_SUB2API="true"

uv run python openai_register.py --once
```

### 5.2 使用邮箱密码登录后台
如果你不用 Admin API Key，也可以用管理员账号登录换 bearer：

```bash
uv run python openai_register.py \
  --sub2api-base-url http://203.0.113.10:8080 \
  --sub2api-email admin@example.com \
  --sub2api-password 'your_admin_password' \
  --sub2api-upload \
  --prune-local \
  --once
```

### 5.3 控制 Sub2API 池子数量
如果你希望 Sub2API 可用账号达到某个数量后暂停注册，用 `--sub2api-target-count`：

```bash
uv run python openai_register.py \
  --sub2api-base-url http://203.0.113.10:8080 \
  --sub2api-admin-api-key YOUR_SUB2API_ADMIN_API_KEY \
  --sub2api-group-ids 2 \
  --sub2api-upload \
  --sub2api-target-count 300
```

也支持环境变量：

```bash
export SUB2API_TARGET_COUNT="300"
```

规则很简单：

- 只有开启 `--sub2api-upload` 时，这个限制才有意义
- 当 Sub2API 当前可用账号数 `>= target_count` 时，脚本不会继续注册
- 循环模式下会等待一段时间后再检查
- `0` 表示不限制

### 5.4 清理 `training_set_failed` 账号

```bash
uv run python openai_register.py \
  --sub2api-base-url http://203.0.113.10:8080 \
  --sub2api-admin-api-key YOUR_SUB2API_ADMIN_API_KEY \
  --sub2api-clean-training-set-failed \
  --sub2api-clean-only \
  --once
```

清理逻辑说明：

- 会先全量分页拉取账号
- 再在本地精确匹配 `extra.privacy_mode == "training_set_failed"`
- 只有命中的账号才会删除

---

## 6. CPA 用法

### 6.1 注册成功后自动上传

```bash
cd openai-register
uv run python openai_register.py \
  --cpa-base-url http://203.0.113.10:8317 \
  --cpa-token YOUR_CPA_LOGIN_PASSWORD \
  --cpa-upload \
  --prune-local \
  --once
```

### 6.2 上传 + 清理 + 数量控制
如果还要使用 `--cpa-clean`，记得先执行：

```bash
uv sync --extra cpa
```

完整示例：

```bash
uv run python openai_register.py \
  --cpa-base-url http://203.0.113.10:8317 \
  --cpa-token YOUR_CPA_LOGIN_PASSWORD \
  --cpa-upload \
  --cpa-clean \
  --cpa-target-count 300 \
  --cpa-workers 1 \
  --cpa-timeout 12 \
  --cpa-retries 1 \
  --cpa-used-threshold 95 \
  --prune-local
```

填写规则：

- `--cpa-token` 填 CPA 后台登录密码
- `--cpa-base-url` 只写到协议 + IP/域名 + 端口
- 不要带后台页面路径

正确示例：

```bash
http://203.0.113.10:8317
```

错误示例：

```bash
http://203.0.113.10:8317/management.html#/
```

---

## 7. 常用运行场景

### 7.1 持续注册，不上传

```bash
uv run python openai_register.py
```

### 7.2 持续注册并上传到 Sub2API

```bash
uv run python openai_register.py \
  --sub2api-base-url http://203.0.113.10:8080 \
  --sub2api-admin-api-key YOUR_SUB2API_ADMIN_API_KEY \
  --sub2api-upload
```

### 7.3 持续注册并保持 Sub2API 池子在目标数量附近

```bash
uv run python openai_register.py \
  --sub2api-base-url http://203.0.113.10:8080 \
  --sub2api-admin-api-key YOUR_SUB2API_ADMIN_API_KEY \
  --sub2api-upload \
  --sub2api-target-count 300
```

### 7.4 持续注册并保持 CPA 池子在目标数量附近

```bash
uv run python openai_register.py \
  --cpa-base-url http://203.0.113.10:8317 \
  --cpa-token YOUR_CPA_LOGIN_PASSWORD \
  --cpa-upload \
  --cpa-target-count 300
```

### 7.5 同时上传到 Sub2API 和 CPA

```bash
uv run python openai_register.py \
  --sub2api-base-url http://203.0.113.10:8080 \
  --sub2api-admin-api-key YOUR_SUB2API_ADMIN_API_KEY \
  --sub2api-upload \
  --cpa-base-url http://203.0.113.10:8317 \
  --cpa-token YOUR_CPA_LOGIN_PASSWORD \
  --cpa-upload \
  --prune-local
```

---

## 8. 参数速查

### 8.1 基础参数
- `--proxy`：HTTP/S 代理
- `--mail-provider`：`auto` / `luckmail` / `gptmail` / `tempmail`
- `--once`：只跑一轮
- `--sleep-min` / `--sleep-max`：循环等待区间
- `--upload-delay-min` / `--upload-delay-max`：注册成功后、执行上传前的等待区间

### 8.2 LuckMail 参数
- `--luckmail-base-url`：LuckMail 平台地址；默认使用内置值 `https://mails.luckyous.com`。
- `--luckmail-api-key`：LuckMail API Key，使用 LuckMail 时通常必填。
- `--luckmail-api-secret`：LuckMail API Secret，可选。
- `--luckmail-use-hmac`：是否启用 HMAC 鉴权；不加这个参数时默认关闭。
- `--luckmail-project-code`：LuckMail 项目编码，默认 `openai`。
- `--luckmail-email-type`：LuckMail 邮箱类型，默认 `ms_graph`。
- `--luckmail-domain`：LuckMail 指定邮箱域名，默认 `outlook.com`。
- `--luckmail-order-timeout`：LuckMail 接码等待超时秒数。
- `--luckmail-poll-interval`：LuckMail 轮询邮件/验证码的间隔秒数。

### 8.3 Sub2API 参数
- `--sub2api-base-url`：Sub2API 地址，只写到协议 + IP/域名 + 端口。
- `--sub2api-admin-api-key`：Sub2API 全局管理员 API Key，推荐优先使用。
- `--sub2api-bearer`：兼容旧方式的 Bearer token。
- `--sub2api-email`：Sub2API 管理员邮箱，用于旧登录方式自动换 bearer。
- `--sub2api-password`：Sub2API 管理员密码，用于旧登录方式自动换 bearer。
- `--sub2api-group-ids`：上传后绑定的分组 ID，逗号分隔，默认 `2`。
- `--sub2api-upload`：注册成功后自动上传到 Sub2API。
- `--sub2api-target-count`：Sub2API 目标可用账号数；达到后暂停注册，`0` 表示不限制。
- `--sub2api-clean-training-set-failed`：清理 `extra.privacy_mode == "training_set_failed"` 的账号。
- `--sub2api-clean-only`：只执行 Sub2API 清理，不跑注册流程。

### 8.4 CPA 参数
- `--cpa-base-url`：CPA 管理地址，只写到协议 + IP/域名 + 端口。
- `--cpa-token`：CPA 后台登录密码。
- `--cpa-workers`：CPA 清理探测并发数。
- `--cpa-timeout`：CPA 请求超时秒数。
- `--cpa-retries`：CPA 清理探测失败后的重试次数。
- `--cpa-used-threshold`：CPA 用量阈值；超过这个 `used_percent` 会被视为需要清理。
- `--cpa-clean`：注册后自动探测并清理 CPA 中失效或超阈值账号。
- `--cpa-upload`：注册成功后自动上传到 CPA。
- `--cpa-target-count`：CPA 目标有效 token 数；达到后暂停注册。

### 8.5 本地清理参数
- `--prune-local`：当 Sub2API / CPA 任一上传成功后，删除本地 token 文件和 `tokens/accounts.txt` 中对应账号

---

## 9. 输出位置
- 账号密码：`tokens/accounts.txt`
  - 格式：`email----password`
- Token JSON：`tokens/token_<email>_<timestamp>.json`

`tokens/` 已加入 `.gitignore`，默认不会提交到仓库。

---

## 10. 注意事项
- 需要能访问 `https://auth.openai.com`
- 代理地区尽量避开 CN / HK
- 临时邮箱不稳定时，可以切换 `--mail-provider gptmail` 或 `--mail-provider tempmail`
- 如果用 `auto`，会自动按 `LuckMail -> TempMail.lol -> GPTMail` 顺序回退
