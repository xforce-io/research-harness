---
title: "How we built a realtime system for responsive voice AI in six months | OpenAI"
authors: []
paper_id: "paper_url_c88f5af6b90407d0"
source_kind: "url"
source_id: "url:https://openai.com/index/continuous-voice-interaction-with-gpt-live/"
source_url: "https://openai.com/index/continuous-voice-interaction-with-gpt-live/"
pdf_url: ""
read_id: "read_paper_url_c88f5af6b90407d0"
kind: library-read
doc_type: "other"
tags: []
---

# How we built a realtime system for responsive voice AI in six months | OpenAI

> 旧式轮次检测架构在延迟与打断之间无法兼顾；GPT-Live 用全双工语音模型 + 异步委托路径分离"说话"与"思考"，使对话响应进入亚秒级。

## Essence

1. **问题** — 传统语音 AI 采用轮次检测器决定何时开始推理：猜早了打断用户，猜晚了响应迟钝，且大模型推理必须等检测器决策后才能启动。
2. **做法** — GPT-Live 移除轮次检测器，改用全双工语音模型同时听与说；深度推理和工具调用通过异步 RPC 路径委托给前沿模型（如 GPT-5.5），不阻塞媒体流。媒体前端与推理逻辑用 Go 重写（替换 Python asyncio），基于 WebRTC 传输，媒体流走专用快速路径，应用逻辑走异步边界。
3. **证据** — "The new system's p95 matching the previous system's p50"——Go 重写后帧投递平滑度显著提升；上下文压缩与模型实例迁移采用并行预热 + 无缝切换机制，避免 KV cache 失效导致的可感知中断。
4. **边界** — 文档未提供具体延迟数值、模型架构细节、压缩算法、或生产环境规模化指标；全文为工程叙事而非评测报告。

## Key takeaways

- **全双工消除轮次检测器**：语音模型本身控制对话节奏，不再依赖外部 VAD/turn detector 做开始推理的决策。这是架构层面的根本性转变。
- **媒体路径与应用逻辑物理分离**：音频在客户端与语音模型之间走专用快速路径，工具调用/后端服务走异步 RPC。慢工具不能拖住媒体流。
- **有状态推理的无缝迁移机制**：模型实例切换（缩容/迁移/上下文压缩）通过"并行预热 → 双实例并行推理 → 就绪后切换"实现，用户无感知。
- **上下文压缩即托管迁移**：压缩会改变历史上下文、使 KV cache 失效；解决方案是在旧实例继续对话的同时，用压缩后上下文准备新实例，就绪后切换。
- **Go 替换 Python asyncio**：媒体前端从 Python asyncio 迁移到 Go，p95 帧投递延迟降至旧系统 p50 水平。
- **WebRTC 作为传输基座**：利用其抗丢包、时钟漂移补偿、音频拉伸/加速回追等机制保证实时性。

## Decisions / claims

- 全双工语音模型从音频路径中移除了轮次检测器，使"听"与"说"可同时进行。（*Moving from turn taking to streaming*）
- 媒体流与应用/业务逻辑分离：音频走专用快速路径，委托/工具调用走异步 RPC 边界。（*Making the media flow quickly*）
- 媒体前端与推理逻辑用 Go 重写，替换 Python asyncio 实现；新系统 p95 帧投递延迟匹配旧系统 p50。（*Making the media flow quickly*）
- WebRTC 提供传输层，支持丢包恢复、时钟漂移容忍、音频拉伸防止间隙。（*Making the media flow quickly*）
- 有状态推理使用无缝切换机制：预热新实例 → 预填充当前会话上下文 → 双实例并行推理 → 就绪后切换。（*Keeping the (stateful) conversation going*）
- 上下文压缩被建模为另一种托管迁移：旧实例继续对话，新实例用压缩上下文准备就绪后切换，避免媒体中断。（*Keeping the (stateful) conversation going*）
- GPT-Live 可异步委托前沿模型（如 GPT-5.5）执行搜索/推理，解耦"说话"与"思考"。（*Delegation without blocking the conversation*）
- 核心语音路径与应用逻辑之间的清晰边界使应用层定制不影响响应性。（*Moving from turn taking to streaming*）

## Constraints & assumptions

- 全双工语音模型本身具备足够的能力同时处理输入和输出音频流；文档未讨论该模型的具体能力边界。
- WebRTC 的音频拉伸/加速机制在感知上是"可接受的"——文档假设用户不会注意到微调，未提供感知测试数据。
- 并行双实例推理的额外计算成本被视为可接受的开销；文档未讨论成本/资源影响。
- 上下文压缩后的会话质量被认为足够——文档未讨论压缩是否丢失关键上下文信息。
- Go 重写适用于媒体前端和推理逻辑；不代表整个系统已完全迁移。

## Open questions

- 端到端延迟的具体数值（如 p50/p95/p99）未披露。
- 上下文压缩的具体算法和压缩比未说明。
- 全双工模型在同时听/说时的回声消除、自干扰处理机制未讨论。
- 异步委托路径的延迟预算——当 GPT-5.5 推理耗时长时，语音模型如何填充等待时间？
- 规模化指标（并发会话数、资源消耗、成本）未提供。
- "Deriving discrete turns from continuous speech"——如何从连续语音流中提取离散轮次用于持久化/日志——文档标题提及但正文截断未展开。

## Relations

- **extends** WebRTC 实时媒体传输在 AI 语音中的应用 [med]：将 WebRTC 从传统音视频通信扩展到 LLM 推理管道，作为模型推理流的传输层。
- **orthogonal** to 级联式 STT→LLM→TTS 架构 [high]：文档明确将级联系统作为被超越的前代架构，全双工语音模型绕过了文本中介。
- **builds-on** OpenAI Realtime API / 前代 ChatGPT Voice 基础设施 [high]：文档明确提到前代流式音频基础设施为本系统奠定了基础。

## Takeaway

- **可复用**：媒体快速路径 + 异步应用逻辑的分离模式可直接迁移到任何需要实时响应 + 耗时后端操作的 agent 架构。
- **可复用**：有状态推理的"并行预热 → 双实例并行 → 无缝切换"模式，适用于任何长会话 + 动态上下文管理的场景。
- **需审慎**：全文为工程叙事，无可验证的量化评测；所有性能声明（如 p95=p50）缺乏绝对数值和测量方法。
- **记忆钩**：全双工去掉 turn detector + 媒体/委托分离 + 压缩即迁移 = 亚秒级语音 AI。
