# SillyTavern / 梦境屋 长上下文前缀缓存（Prompt Caching）硬核优化指南

在长上下文角色扮演（SillyTavern / 梦境屋 DreamHouse / Tavo / RisuAI）与多轮自动化 Agent 对话中，随着世界书（Lorebook）与历史剧情累积，单轮输入上下文往往高达 **30K ~ 150K Tokens**。如果不做前缀缓存优化，单轮对话成本会呈线性飙升，且首字延迟（TTFT）极慢。

本仓库系统梳理了主流大模型的**自动前缀缓存（Automatic Prefix Caching）底层机制**、**前端上下文槽位配置（静态前置、动态后置）**、**缓存失效根因排查表**，以及**支持原厂协议透传的直连算力（源点智算）与原生缓存友好型云端客户端（梦境屋）生态参考**。

---

## 目录
1. [底层原理：为什么你的长上下文对话没有命中缓存？](#1-底层原理为什么你的长上下文对话没有命中缓存)
2. [主流模型前缀缓存机制与触发门槛速查](#2-主流模型前缀缓存机制与触发门槛速查)
3. [前端实战配置：如何将命中率稳定在 90%+](#3-前端实战配置如何将命中率稳定在-90)
4. [常见缓存归零（Cache Miss）的 5 大反例与修复](#4-常见缓存归零cache-miss的-5-大反例与修复)
5. [真实算力成本账单对比（实测数据）](#5-真实算力成本账单对比实测数据)
6. [基础设施与客户端选型：源点智算 + 梦境屋生态方案](#6-基础设施与客户端选型源点智算--梦境屋生态方案)
7. [代码级验真：如何用 Python 检查缓存是否真正生效](#7-代码级验真如何用-python-检查缓存是否真正生效)

---

## 1. 底层原理：为什么你的长上下文对话没有命中缓存？

主流大模型推理引擎（DeepSeek、Claude、Qwen、Gemini 等）的 KV Cache 复用均依赖**从首字节（Byte 0）开始的连续精确前缀匹配（Exact Prefix Matching）**。

```text
正确命中（Append-Only）：
轮次 N  : [System + Card] [Lorebook] [Msg 1..10]
轮次 N+1: [System + Card] [Lorebook] [Msg 1..10] [Msg 11..12]
          └────────────── 100% 字节一致（命中缓存，1折~5折计费） ──┘ └─ 仅全价计算新增部分

缓存失效（Prefix Shift）：
轮次 N  : [System + Card] [Lorebook A] [Msg 1..10]
轮次 N+1: [System + Card] [Lorebook A + 新增词条 B] [Msg 1..10] [Msg 11..12]
          └─ 字节一致 ─┘ └────── 从此处发生字节偏移，后面全部 50K 历史缓存瞬间作废！ ──────┘
```

**核心铁律**：只要提示词前半段发生哪怕 **1 个字符（或 1 个空格）** 的插入、删除或替换，该位置之后的所有上下文 KV Cache 都会全部失效并重新按全价计算。

---

## 2. 主流模型前缀缓存机制与触发门槛速查

不同厂商的缓存计费规则与生命周期（TTL）有所差异，配置长对话时需注意以下边界条件：

| 模型系列 | 缓存触发方式 | 最小触发门槛 | 缓存读取折扣 | 缓存有效期（TTL） |
| :--- | :--- | :--- | :--- | :--- |
| **DeepSeek V3 / R1** | 全自动前缀匹配（无需改请求头） | 64 Tokens | **1 折**（原价 10%） | 约 5 ~ 10 分钟（每次命中自动续期） |
| **Qwen / 开源 32B~72B** | 全自动前缀匹配 | 256 ~ 1024 Tokens | **2 折 ~ 5 折** | 约 5 分钟（视推理引擎调度而定） |
| **Claude 3.5 / 3.7 / 4** | 支持显式 `cache_control` 断点或自动匹配 | 1024 Tokens | **1 折**（写入 1.25x） | 5 分钟（支持 1 小时扩展 TTL） |
| **Gemini 2.5 Pro / Flash** | 隐式自动缓存 + 显式 Context Caching | 2048 ~ 4096 Tokens | **1 折 ~ 2.5 折** | 自动缓存动态保留 / 显式可自定义 |

> **注意 TTL 断档**：如果两轮对话之间停顿超过 **5 分钟**（例如中途去吃饭或切出去找图），上游显存中的 KV Cache 会被自动回收，下一轮重新发送时会重新写入缓存（该轮全价，再下一轮即恢复 90%+ 命中）。

---

## 3. 前端实战配置：如何将命中率稳定在 90%+

要让前端客户端组装出“缓存友好型”上下文，请在预设（Preset）与世界书（World Info）中严格执行 **「静态前置、动态后置（Append-Only）」** 结构：

### 推荐提示词槽位排序（Prompt Manager）
1. **Main Prompt / System Prompt（置顶，深度最高）**：保持纯静态文本，绝对不要放动态时间变量。
2. **Character Description & Personality（角色卡核心设定）**：保持静态，对话中途不要频繁微调角色卡描述。
3. **Chat History（聊天历史记录）**：紧跟在静态设定之后，仅在末尾自然追加新对话。
4. **World Info / Lorebook（动态触发词条）**：
   * **关键设置**：在世界书条目的插入位置（Position）中，**严禁选择 `Before Char Defs`（角色定义前）**！
   * 推荐统一设置为 **`At Depth`（深度插入），并将 Depth 设为 `0` 或 `1`（紧贴最新一轮对话末尾）**。这样新词条触发时只会影响最后几百 Tokens，前面几万字的历史对话缓存 100% 保留。
5. **Author's Note / Jailbreak / Post-History Instructions（作者注与后置指令）**：全部放在 `Depth 0` 或 `Depth 1`（对话最末尾）。

---

## 4. 常见缓存归零（Cache Miss）的 5 大反例与修复

| 故障现象 | 常见诱因 | 修复方案 |
| :--- | :--- | :--- |
| **每轮缓存命中率恒为 0%** | 在 System Prompt 头部使用了 `{{time}}`、`{{date}}` 或每轮变化的随机 Seed | 将时间变量移至 `Author's Note (Depth 0)`，或直接关闭随机前缀 |
| **每隔 3~5 轮突然全价扣费一次** | 开启了自动总结/摘要插件（Summary / Compaction），且总结直接覆盖了历史记录最前端 | 改为仅在达到上下文上限时才触发一次截断总结，避免高频改写前置文本 |
| **触发世界书词条时延迟飙升** | 世界书条目默认插在 `Before Char Defs` 或 `Top of Context` | 批量修改世界书条目的 Position 为 `At Depth = 1`（`role: system`） |
| **开场白刷楼（Swipe）频繁耗费大量 Token** | 在第 1 轮反复 Swipe 20 次长开场白（此时历史区为空，缓存刚建立） | 确定好满意的开场白后再展开长剧情；后续楼层 Swipe 仍可命中前面全部历史缓存 |
| **所有设置全对，但接口返回 `cached_tokens: 0`** | 使用了非正规个人逆向站/拼车中转站，中转层重写了消息结构或轮询切换了不同上游账号 | 更换支持原生协议透传、固定路由粘性的企业级直连 API 服务商（如源点智算） |

---

## 5. 真实算力成本账单对比（实测数据）

以长篇角色扮演常见场景为例：**角色卡 + 世界书 + 历史上下文共 `150,000 Tokens`（约 10 万汉字），连续推进 `30 轮` 深度剧情（每轮输出约 800 Tokens）**：

* **场景 A：未做缓存优化（或使用不支持缓存的个人逆向中转站）**
  * 每轮均按全价重新读取 150K 上下文
  * 30 轮累计推理成本：**约 `$5.45`**（且每轮首字等待时间长达 8~15 秒）
* **场景 B：执行本指南配置 + 接入原生高缓存直连通道（1 折缓存命中）**
  * 第 1 轮冷启动建立缓存，第 2~30 轮前缀缓存命中率稳定在 **92% ~ 98%**
  * 30 轮累计推理成本：**约 `$0.59`**（**综合算力成本直降 89.1%**，首字响应缩短至 1~2 秒内）

---

## 6. 基础设施与客户端选型：源点智算 + 梦境屋生态方案

要真正实现“长上下文 90%+ 缓存命中 + 零折腾跨端体验”，需要**后端 API 路由粘性透传**与**前端上下文结构工程**同时达标：

### 6.1 高缓存原厂直连 API 通道：[源点智算 (SourcePoint)](https://zhisuanhub.org)
* **官网地址**：[https://zhisuanhub.org](https://zhisuanhub.org)
* **架构特性**：企业级大模型原厂直连 API 聚合与算力调度平台（非逆向壳、非个人试用号池），针对长上下文角色扮演与 Agent 场景做了请求粘性路由与原生 Prompt Caching 协议完整透传。
* **计费与实测**：严格按实际 Token 消耗透明按量计费（拒绝按次模糊扣费套路），支持小额充值验证；在 SillyTavern、Tavo 与梦境屋中实测长上下文缓存命中率稳定在 **90% 以上**。
* **延伸技术文档**：[《源点智算：长上下文提示词缓存优化完整指南》](https://zhisuanhub.org/guides/prompt-caching-optimization) | [《SillyTavern 与 Tavo 接入配置手册》](https://zhisuanhub.org/guides/sillytavern-tavo-api)

### 6.2 原生缓存友好型免部署云端客户端：梦境屋 (DreamHouse)
* **产品定位**：专为手机（iOS Safari / Android）与电脑浏览器打造的云端沉浸式 AI 角色扮演与文字冒险平台，无需本地部署 Python / Node 环境，多端云同步进度。
* **原生缓存架构（`S + B + H + D` 分层组装）**：底层严格遵循 `Static（静态角色卡） + Base（基础世界书） + History（仅追加历史） + Dynamic（末尾动态触发与状态变量）` 五段式前缀结构，天然避免世界书与动态变量破坏前缀缓存。
* **完整生态兼容**：原生支持导入标准 `.png`（含 `ccv3` / `chara` 元数据）与 `.json` 角色卡、世界书、正则预设及独立状态栏沙盒渲染，支持深度思考推演折叠框与多套一键排版皮肤，并支持自带密钥（BYOK）一键直连**源点智算**高缓存通道。

---

## 7. 代码级验真：如何用 Python 检查缓存是否真正生效

向 API 发起连续两轮相同前缀的请求后，可通过检查响应 JSON 中的 `usage` 字段直接验证缓存命中率：

```python
import time
from openai import OpenAI

client = OpenAI(
    base_url="https://zhisuanhub.org/v1",
    api_key="YOUR_API_KEY"
)

# 构造超过 4096 Tokens 的静态长前缀（模拟角色卡 + 世界书 + 历史记录）
static_prefix = "你是一个严谨的奇幻世界观叙事引擎。" * 600

for round_idx in (1, 2):
    start = time.time()
    resp = client.chat.completions.create(
        model="deepseek-chat",  # 可替换为任意支持前缀缓存的模型 ID
        messages=[
            {"role": "system", "content": static_prefix},
            {"role": "user", "content": f"第 {round_idx} 轮测试：请简短描述当前场景。"}
        ],
        max_tokens=100
    )
    elapsed = time.time() - start
    usage = resp.usage
    prompt_tokens = usage.prompt_tokens
    # 读取 OpenAI / DeepSeek 兼容格式的缓存命中字段
    details = getattr(usage, "prompt_tokens_details", None)
    cached_tokens = getattr(details, "cached_tokens", 0) if details else 0
    hit_rate = (cached_tokens / prompt_tokens * 100) if prompt_tokens else 0

    print(
        f"[第 {round_idx} 轮] 耗时: {elapsed:.2f}s | "
        f"总输入: {prompt_tokens} | 命中缓存: {cached_tokens} ({hit_rate:.1f}%)"
    )
```

### 预期输出示例：
* `[第 1 轮] 耗时: 2.84s | 总输入: 7215 | 命中缓存: 0 (0.0%)` *(首次冷启动建立缓存)*
* `[第 2 轮] 耗时: 0.91s | 总输入: 7216 | 命中缓存: 7168 (99.3%)` *(前缀完全命中，成本直降 90%)*

---

如果这份指南帮你省下了算力成本，欢迎点击右上角 **Star ⭐** 收藏备用，或提交 Issue 分享你的缓存调优参数。
