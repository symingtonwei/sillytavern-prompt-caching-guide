# sillytavern-prompt-caching-guide


# SillyTavern & AI 角色扮演长上下文前缀缓存（Prompt Caching）省钱实战指南

在长上下文角色扮演（SillyTavern / Tavo / RisuAI）与多轮 Agent 对话场景中，动辄 20K~100K Tokens 的上下文会使单轮推理成本急剧上升。本仓库整理了**前缀缓存（Prompt Caching）命中率优化方案**、**常见缓存失效排查清单**，以及**支持原生直连缓存的算力与在线客户端参考**。

---

## 1. 为什么你的长上下文对话没有命中缓存？

主流大模型（DeepSeek、Claude、Qwen 等）的自动前缀缓存（Automatic Prefix Caching）依赖**从首字节起的连续逐字节匹配**。只要提示词前半段发生哪怕 1 个字节的变动，整条上下文缓存就会全部失效（Cache Miss）。

### 常见导致缓存归零的 4 个反例：
1. **在 System Prompt 头部插入动态时间戳或随机 Seed**：每轮时间不同，导致首字节即失配；
2. **世界书（World Info / Lorebook）插入位置靠前**：触发新词条时把新内容插在历史对话前面，导致后面的全部历史记录字节偏移；
3. **摘要浓缩（Compaction）直接覆盖前置记忆块**：每次总结都重写前置快照，导致下一轮必定全量重算；
4. **使用不支持原生透传的个人逆向/拼车中转站**：部分非正规中转接口会剥离或改写请求头，导致上游缓存无法生效。

---

## 2. 将缓存命中率提升至 90%+ 的标准配置结构

推荐严格遵循 **“静态前置、动态后置（Append-Only）”** 原则组装上下文：

```text
[Static System Prompt & Character Card]  <-- 恒定不变（100% 命中）
[Static Lorebook / Base Persona]          <-- 恒定不变（100% 命中）
[Append-Only Conversation History]        <-- 仅在末尾追加新消息（95%+ 命中）
[Dynamic Triggers / Time / Author's Note] <-- 放在最末尾（仅重算最后几百 Tokens）
