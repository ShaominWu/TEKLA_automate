# RECOVERY.md - 系统恢复手册

出问题了？按这个手册一步步来，能完整恢复整个系统。

---

## 第一步：安装 OpenClaw

```bash
npm install -g openclaw
```

---

## 第二步：克隆 workspace

```bash
git clone https://github.com/ShaominWu/TEKLA_automate.git
cd TEKLA_automate
git submodule update --init --recursive
```

---

## 第三步：重新填入 API Keys

以下 key 不在 GitHub 里，需要重新配置：

### Anthropic (Claude)
```bash
openclaw config set auth.profiles.anthropic:default.apiKey "你的 Anthropic API Key"
```
获取地址：https://console.anthropic.com/

### Google (Gemini)
需要在代理配置文件里填入：
`C:\Users\wuyon\.openclaw\agents\pdf_data_extractor\agent\auth-profiles.json`
```json
{
  "profiles": {
    "google:default": {
      "type": "api_key",
      "provider": "google",
      "key": "你的 Google API Key"
    }
  }
}
```
获取地址：https://aistudio.google.com/apikey

---

## 第四步：重新配置 openclaw.json

把以下内容写入 `~/.openclaw/openclaw.json`，替换对应的 key：

```json
{
  "auth": {
    "profiles": {
      "anthropic:default": {
        "provider": "anthropic",
        "mode": "api_key"
      }
    }
  },
  "models": {
    "providers": {
      "ollama": {
        "baseUrl": "http://127.0.0.1:11434",
        "apiKey": "ollama-local",
        "api": "ollama",
        "models": [
          { "id": "qwen3.5", "name": "qwen3.5", "reasoning": true, "input": ["text","image"], "cost": {"input":0,"output":0,"cacheRead":0,"cacheWrite":0}, "contextWindow": 262144 },
          { "id": "qwen3.5:27b", "name": "qwen3.5:27b", "reasoning": true, "input": ["text","image"], "cost": {"input":0,"output":0,"cacheRead":0,"cacheWrite":0}, "contextWindow": 262144 },
          { "id": "qwen2.5:32b", "name": "qwen2.5:32b", "reasoning": false, "input": ["text","image"], "cost": {"input":0,"output":0,"cacheRead":0,"cacheWrite":0}, "contextWindow": 131072 }
        ]
      }
    }
  },
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "【重新从 BotFather 获取或使用原 token】",
      "dmPolicy": "pairing",
      "groups": { "*": { "requireMention": true } }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-sonnet-4-6",
        "fallbacks": ["ollama/qwen3.5:27b"]
      },
      "workspace": "C:\\Users\\wuyon\\.openclaw\\workspace"
    },
    "list": [
      {
        "id": "pdf_data_extractor",
        "name": "PDF Data Extractor",
        "workspace": "C:\\Users\\wuyon\\.openclaw\\workspace-pdf-extractor",
        "agentDir": "C:\\Users\\wuyon\\.openclaw\\agents\\pdf_data_extractor\\agent",
        "model": "google/gemini-2.0-flash"
      },
      {
        "id": "model_generator",
        "name": "Model Generator",
        "workspace": "C:\\Users\\wuyon\\.openclaw\\workspace-model-generator",
        "agentDir": "C:\\Users\\wuyon\\.openclaw\\agents\\model_generator\\agent",
        "model": "ollama/qwen3.5:27b"
      }
    ]
  },
  "tools": { "profile": "coding" },
  "commands": { "native": "auto", "nativeSkills": "auto", "restart": true, "ownerDisplay": "raw" },
  "session": { "dmScope": "per-channel-peer" },
  "gateway": {
    "port": 18789,
    "mode": "local",
    "bind": "loopback",
    "auth": { "mode": "token", "token": "ollama" },
    "tailscale": { "mode": "off", "resetOnExit": false }
  }
}
```

---

## 第五步：配置子代理 workspace

把 GitHub 里的文件复制到对应目录：

| 文件 | 目标路径 |
|------|----------|
| `workspace-pdf-extractor/` 内容 | `C:\Users\wuyon\.openclaw\workspace-pdf-extractor\` |
| `workspace-model-generator/` 内容 | `C:\Users\wuyon\.openclaw\workspace-model-generator\` |
| `workspace-master/` 内容 | `C:\Users\wuyon\.openclaw\workspace-master\` |

---

## 第六步：拉起 Ollama 模型

```bash
ollama pull qwen3.5:27b
ollama pull qwen2.5:32b
ollama pull qwen3.5
```

---

## 第七步：启动 gateway

```bash
openclaw gateway start
```

---

## 第八步：重新配对 Telegram

```bash
# 在 Telegram 给 bot 发 /start
openclaw pairing list telegram
openclaw pairing approve telegram <配对码>
```

---

## 系统架构速查

```
PDF → PDF_Data_Extractor (Gemini) → XML
                                      ↓
                          Model_Generator (qwen3.5:27b)
                                      ↓
                              Tekla 主体模型
                                      ↓
                           节点代理（待配置）
```

## 关键账号

- GitHub: https://github.com/ShaominWu
- Anthropic Console: https://console.anthropic.com/
- Google AI Studio: https://aistudio.google.com/
- Telegram BotFather: @BotFather

---

_最后更新：2026-03-15_
