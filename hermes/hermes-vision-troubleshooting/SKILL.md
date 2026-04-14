---
name: hermes-vision-troubleshooting
description: Diagnose and fix vision_analyze failures in Hermes Agent — vision provider resolution, API key issues, and available vision backends.
version: 1.0.0
author: Hermes Agent
tags: [hermes, vision, troubleshooting, diagnosis]
---

# Hermes Vision 工具故障排查

## 症状

`vision_analyze` 返回 "I don't see any image" 或工具调用后一直卡住/超时。

## 排查步骤

### 1. 确认 vision provider 是否解析成功

```python
from agent.auxiliary_client import resolve_vision_provider_client
provider, client, model = resolve_vision_provider_client()
print(f"Provider: {provider}, Client: {client}, Model: {model}")
```

如果 `client is None` 或 `provider is None` → provider 没有解析到可用后端。

### 2. 查看当前配置

```bash
hermes config | grep -A5 auxiliary
```

`auxiliary.vision.provider` 应该是 `openrouter` 或具体 provider 名（非 `auto`）。

### 3. 检查可用 API Keys

| Provider | 需要的环境变量 | Vision 支持 |
|----------|--------------|------------|
| OpenRouter | `OPENROUTER_API_KEY` | ✅ 指定 `google/gemini-2.5-flash-image` 模型即可，支持且极便宜 |
| Google AI Studio | `GOOGLE_API_KEY` | ✅ 免费额度 |
| MiniMax CN | `MINIMAX_CN_API_KEY` | ❌ **不支持视觉**，M2.5 是纯文本模型 |
| MiniMax (Anthropic 兼容) | — | ❌ image/document input 明确不可用 |

**重要发现：**
- MiniMax 所有模型（MiniMax-M2.7 等）Anthropic 兼容接口**明确不支持 image 输入**
- OpenRouter 免费模型列表中的模型**均为纯文本模型**，没有支持视觉的
- 唯一免费方案：Google AI Studio (aistudio.google.com) 的 Gemini 免费额度

## 快速修复

### 方案 A：OpenRouter + Gemini Flash Image（✅ 推荐，便宜）

```bash
# 1. 注册 https://openrouter.ai 并充值（最少 $1，5.5% 手续费）
# 2. 获取 API key（sk-or-v1-... 格式）
# 3. 配置（provider=openrouter，model 手动指定）
```

配置示例（`~/.hermes/config.yaml`）：
```yaml
auxiliary:
  vision:
    provider: openrouter
    model: google/gemini-2.5-flash-image   # $0.0000003/1M tokens，极便宜
    timeout: 120
```

验证可用模型（查 OpenRouter API）：
```python
import urllib.request, json
key = "your-openrouter-key"
req = urllib.request.Request(
    'https://openrouter.ai/api/v1/models',
    headers={'Authorization': f'Bearer {key}'}
)
with urllib.request.urlopen(req) as r:
    models = json.loads(r.read())['data']
for m in models:
    mid = m['id'].lower()
    if any(k in mid for k in ['vision', 'vl', 'image', 'gemini-2.5-flash-image']):
        print(m['id'], f"${m['pricing']['prompt']}/1M tokens")
```

实测可用的 vision 模型（2026-04）：
- `google/gemini-2.5-flash-image` ✅ **已验证可用**，$0.0000003/1M tokens
- `google/gemini-3.1-flash-image-preview` ✅ 可用
- `nvidia/nemotron-nano-12b-v2-vl:free` ❌ 返回 NoneType error，不可用

### 方案 B：Google AI Studio（免费额度）

```bash
# 1. 去 https://aistudio.google.com 注册，获得免费 credits
# 2. 创建 API key
# 3. 配置
hermes config set auxiliary.vision.provider google
# 4. 在 ~/.hermes/.env 中写入 key
echo "GOOGLE_API_KEY=AI..." >> ~/.hermes/.env
```

### 验证修复

```python
from agent.auxiliary_client import resolve_vision_provider_client
provider, client, model = resolve_vision_provider_client()
assert client is not None, "Vision provider not resolved!"
### 确认 MiniMax 不支持 vision
MiniMax Anthropic 兼容接口明确文档说明：
- `type="image" ❌ Not supported`
- `type="document" ❌ Not supported`
- `_PROVIDER_VISION_MODELS` 只包含 `xiaomi: "mimo-v2-omni"`，不含 MiniMax

**不要尝试把 MiniMax 作为 vision 后端**。即使 MiniMax 平台上可能有独立的 VL 模型，OpenRouter 上也搜索不到有效的 minimax-vl 模型 ID。

### 排查流程图
```
vision_analyze 返回 "can't see image"
    ↓
resolve_vision_provider_client()
    ├─ 返回 None → provider 未解析 → 检查 .env 的 key
    │   └─ 解决：取消注释或添加有效 key
    └─ 返回 OpenAI client → provider 解析成功 → 测试 API key
        ├─ 402 Insufficient credits → 充值 OpenRouter
        ├─ 401 Unauthorized → key 无效 → 重新获取
        └─ 200 OK 但仍返回 "can't see" → 图片文件问题或 API 内部错误
```

### 快速诊断命令
```bash
# 1. 检查 provider 解析
cd /root/.hermes/hermes-agent && python3 -c "
import os, sys; os.environ['HERMES_HOME'] = '/root/.hermes'
sys.path.insert(0, '.')
from agent.auxiliary_client import resolve_vision_provider_client
p, c, m = resolve_vision_provider_client()
print(f'Provider={p}, Client={c}, Model={m}')
"

# 2. 检查 .env 中的 key（排除注释行）
grep -v "^#" ~/.hermes/.env | grep "_KEY\|_TOKEN"

# 3. 测试 OpenRouter key 余额
curl -s https://openrouter.ai/api/v1/balance \
  -H "Authorization: Bearer $(grep OPENROUTER_API_KEY ~/.hermes/.env | grep -v '^#' | cut -d= -f2)"

# 4. 检查图片文件是否真的是图片（不是 HTML 错误页）
file /path/to/image.jpg
xxd /path/to/image.jpg | head -3  # JPEG 应以 ffd8 开头
```

### Nous Portal 备选方案
如果主 provider 是 `nous`，vision 会走 Nous Portal：
```bash
hermes login --provider nous  # OAuth 认证
# 检查 ~/.hermes/auth.json 中 providers 是否有 nous 项
```

### 确认 MiniMax 不支持 vision
MiniMax Anthropic 兼容接口明确文档说明：
- `type="image" ❌ Not supported`
- `type="document" ❌ Not supported`
- `_PROVIDER_VISION_MODELS` 只包含 `xiaomi: "mimo-v2-omni"`，不含 MiniMax

不要尝试把 MiniMax 作为 vision 后端。

### OpenRouter 免费模型真相
OpenRouter 的免费模型（2026年4月）**大部分是纯文本**，但有一个 vision 模型：
- ❌ NVIDIA Nemotron 3 Super (free) — 纯文本
- ❌ Arcee AI Trinity Large Preview (free) — 纯文本
- ❌ Z.ai GLM 4.5 Air (free) — 纯文本
- ❌ MiniMax M2.5 (free) — 纯文本

**唯一接近免费的 vision 模型：**
- `google/gemini-2.5-flash-image` — $0.0000003/1M tokens（极便宜，充值 $1 可用很久）
- `google/gemini-3.1-flash-image-preview` — $0.0000005/1M tokens

有 vision 的模型（Gemini Flash、GPT-4o-mini）都需要付费，但价格极低。

## 已知限制

- **MiniMax 全系不支持 vision**
- **OpenRouter 免费模型无 vision**
- **vision provider = auto 时**：如果主 provider 不在 `_VISION_AUTO_PROVIDER_ORDER` 列表中，会跳过该 provider

## 相关路径

- 配置：`~/.hermes/config.yaml`
- Keys：`~/.hermes/.env`
- 图片缓存：`~/.hermes/image_cache/`
