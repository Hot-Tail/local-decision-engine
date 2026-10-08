# local-decision-engine

## 这是什么

这是一个基于 Ollama 0.35+ 的 `/v1/systemone` 决策模型接口，在本地部署意图分类与紧急度打分引擎的完整操作记录。内容涵盖决策模型原理、Ollama 配置、tev1:0.8b / tev1:4b 模型部署、请求格式、Dify 自定义工具集成、常见踩坑与解决方案。

## 解决了什么问题

- **企业级 Agent 的数据隐私问题**：本地部署决策引擎，数据不出内网，满足企业合规要求。
- **API 费用失控问题**：意图分类与工单路由完全本地运行，零 API 费用。
- **Dify HTTP 节点变量替换 Bug**：1.17.0/1.17.1 下 JSON Body 变量失效，改用自定义工具绕开。
- **决策模型请求格式错误问题**：choice 类型必须用 criteria 对象，score 类型必须用 criteria 数组，格式错误会导致概率均匀、置信度接近 0。
- **本地硬件限制问题**：AMD R7 M265 2GB 显存无法 GPU 加速，通过 CPU 推理 tev1:0.8b / tev1:4b 实现可用性能。

## 怎么用

下面按步骤操作即可。环境准备 → 安装 Ollama → 拉取决策模型 → 测试接口 → 接入 Dify。每一步都附有命令说明。如遇问题，可查阅「踩坑记录」章节。

## 环境准备

- 操作系统：Windows 10 IoT 企业版 LTSC 21H2
- WSL2：已启用，发行版 Ubuntu
- Ollama：已安装（0.35 或更高版本）
- Dify：已部署（1.17.1 或更高版本）
- 硬件：普通笔记本，16GB RAM（AMD 显卡无 CUDA，纯 CPU 推理）

## 架构说明

```
┌─────────────────┐     ┌──────────────┐     ┌─────────────────┐
│ Dify 工作流     │────▶│ Ollama       │────▶│ tev1 决策模型   │
│ (自定义工具)    │     │  :11434      │     │ (本地 CPU 推理) │
└─────────────────┘     └──────────────┘     └─────────────────┘
```

- **Ollama**：本地模型运行平台，0.35+ 支持 `/v1/systemone` 决策模型接口。
- **tev1 决策模型**：专为决策任务训练的模型，0.8b 秒回，4b 约 5 秒（CPU 推理）。
- **Dify 自定义工具**：通过 OpenAPI Schema 传参，绕开 HTTP 节点变量替换 Bug。

## 部署步骤

### 1. 升级 Ollama 到 0.35+

Windows 上下载最新 Ollama 安装包，或通过命令升级。升级后确认：

```bash
ollama -v
# 应显示 0.35.1 或更高
```

### 2. 拉取决策模型

```bash
ollama pull tev1:0.8b
ollama pull tev1:4b
```

确认模型可用：

```bash
ollama list
# 应显示 tev1:0.8b 和 tev1:4b
```

### 3. 测试 /v1/systemone 接口

**意图分类（choice 类型）**：

```bash
curl -X POST http://127.0.0.1:11434/v1/systemone \
  -H "Content-Type: application/json" \
  -d '{
    "model": "tev1:4b",
    "state": "我买的衣服尺码不对，想换货。",
    "questions": {
      "intent": {
        "type": "choice",
        "instructions": "这个客户想干什么？",
        "criteria": {
          "refund": "退款",
          "exchange": "换货",
          "consult": "咨询"
        }
      }
    }
  }'
```

预期返回：

```json
{
  "model": "tev1:4b",
  "answers": {
    "intent": {
      "type": "choice",
      "choice": "exchange",
      "probabilities": {"refund": 0.0007, "exchange": 0.9963, "consult": 0.0030},
      "confidence": 0.9761
    }
  },
  "usage": {"input_tokens": 127, "output_tokens": 1}
}
```

**紧急度评分（score 类型）**：

```bash
curl -X POST http://127.0.0.1:11434/v1/systemone \
  -H "Content-Type: application/json" \
  -d '{
    "model": "tev1:4b",
    "state": "我买的衣服尺码不对，想换货。",
    "questions": {
      "urgency": {
        "type": "score",
        "instructions": "这个工单有多紧急？",
        "criteria": ["Routine: 不着急", "Soon: 客户有些不便", "Urgent: 需要立刻处理"]
      }
    }
  }'
```

预期返回：

```json
{
  "model": "tev1:4b",
  "answers": {
    "urgency": {
      "type": "score",
      "score": 0.825,
      "legend": {"0": "Routine: 不着急", "1": "Soon: 客户有些不便", "2": "Urgent: 需要立刻处理"},
      "probabilities": {"0": 0.2885, "1": 0.5979, "2": 0.1136},
      "confidence": 0.1687
    }
  }
}
```

### 4. 接入 Dify 自定义工具

**在 Dify 里创建自定义工具**：

进 工具 → 自定义 → 创建自定义工具，粘贴以下 OpenAPI Schema：

```json
{
  "openapi": "3.0.0",
  "info": {
    "title": "意图-紧急度决策引擎",
    "description": "调用本地 Ollama /v1/systemone 接口，对工单进行意图分类和紧急度打分。",
    "version": "1.0.0"
  },
  "servers": [
    {
      "url": "http://172.29.80.1:11434"
    }
  ],
  "paths": {
    "/v1/systemone": {
      "post": {
        "summary": "意图与紧急度判定",
        "operationId": "systemoneDecision",
        "requestBody": {
          "required": true,
          "content": {
            "application/json": {
              "schema": {
                "type": "object",
                "required": ["model", "state", "questions"],
                "properties": {
                  "model": {"type": "string", "default": "tev1:4b"},
                  "state": {"type": "string"},
                  "questions": {"type": "string"}
                }
              }
            }
          }
        },
        "responses": {
          "200": {
            "description": "成功返回决策结果",
            "content": {
              "application/json": {
                "schema": {
                  "type": "object",
                  "properties": {
                    "model": {"type": "string"},
                    "answers": {"type": "object"}
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

**在工作流中接线**：

```
开始 → 代码节点（构造 questions 字符串）→ 自定义工具 → 代码节点（解析 JSON）→ 输出
```

**代码节点 1（构造 questions）**：

```python
import json

def main(query: str) -> dict:
    questions = {
        "intent": {
            "type": "choice",
            "instructions": "这个客户想干什么？",
            "criteria": {
                "refund": "退款",
                "exchange": "换货",
                "consult": "咨询"
            }
        }
    }
    return {"questions": json.dumps(questions, ensure_ascii=False)}
```

**代码节点 2（解析 JSON）**：

```python
import json

def main(raw_body: str) -> dict:
    try:
        data = json.loads(raw_body)
        answers = data.get("answers", {})
        intent = answers.get("intent", {})
        urgency = answers.get("urgency", {})
        return {
            "intent_choice": intent.get("choice", "unknown"),
            "intent_confidence": intent.get("confidence", 0.0),
            "urgency_score": urgency.get("score", 0.0),
            "urgency_confidence": urgency.get("confidence", 0.0)
        }
    except Exception:
        return {
            "intent_choice": "parse_error",
            "intent_confidence": 0.0,
            "urgency_score": 0.0,
            "urgency_confidence": 0.0
        }
```

## 模型选择建议

| 模型 | 参数量 | 推理速度（CPU） | 准确率 | 适用场景 |
|---|---|---|---|---|
| **tev1:0.8b** | 0.8B | 秒回 | 约 63.5% | 轻量测试、快速粗筛 |
| **tev1:4b** | 4B | 约 5 秒 | 约 88% | 生产环境、高准确率需求 |

**建议**：开发调试用 0.8b，生产交付用 4b。

## 踩坑记录

| 问题 | 现象 | 原因 | 解决方法 |
|---|---|---|---|
| 接口返回概率均匀 | 置信度接近 0 | 用了 options 数组 | choice 类型必须用 criteria 对象 |
| 接口返回 error: model is required | 400 Bad Request | 请求体缺少 model 字段 | 必须指定 model，如 tev1:4b |
| 接口返回 error: questions must contain 1-64 fields | 400 Bad Request | 缺少 questions 字段 | 必须用 questions 包裹决策项 |
| 接口返回 error: state: must be a string, object, or array | 400 Bad Request | state 格式错误 | state 必须直接是字符串，不要嵌套 |
| Dify HTTP 节点变量替换失效 | 请求体为空 {} | Dify 1.17.0/1.17.1 Bug | 改用自定义工具 |
| Dify 代码节点直接发请求被 403 | HTTP Error 403 | sandbox 禁止访问内网 | 用 HTTP 节点发请求，代码节点只做 JSON 解析 |
| ollama run tev1:4b 输出乱码 | 陷入决策思维，输出大段 Thinking | tev1 是决策模型，不适用聊天模式 | 用 curl 或 Dify HTTP 节点调用 /v1/systemone |
| Windows portproxy v4tov4 失败 | Empty reply from server | Ollama 监听 IPv6 回环 | 改用 v4tov6，转发到 [::1]:11434 |
| WSL2 无法访问 Ollama | 连接超时 | 防火墙网段不匹配 | 防火墙规则改为 172.29.0.0/16 |

## 长期使用建议

- **开发用 0.8b，生产用 4b**：0.8b 秒回适合快速迭代，4b 准确率更高适合交付。
- **Dify 走自定义工具**：不要用 HTTP 节点，1.17.1 有变量替换 Bug。
- **防火墙规则**：Ollama 的 11434 端口仅对 WSL2 网段（172.29.0.0/16）开放。
- **portproxy 用 v4tov6**：Ollama 监听 IPv6 回环，v4tov4 会失败。
- **数据不出内网**：本地决策引擎的最大卖点，适合企业级 Agent 场景。
- **定期检查 Ollama 版本**：0.35+ 才支持 /v1/systemone 接口。
- **Ollama 服务常驻**：本地决策引擎需要 Ollama 服务一直运行。

## 安全提醒

- Ollama 的 11434 端口仅对 WSL2 网段开放，Windows 防火墙规则限定 `172.29.0.0/16`。
- 不要将 11434 端口对公网开放。
- 本地决策引擎的最大卖点是数据不出内网，不要将决策请求发送到云端。
- 客户敏感信息通过本地模型处理，避免泄露。
- 使用 Dify 自定义工具时，注意不要把内网 IP 写在公开的 Schema 里。

## 许可

MIT License

Copyright (c) 2026 Author

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
