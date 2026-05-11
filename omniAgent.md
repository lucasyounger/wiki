# OmniAgent 设计文档

> OmniAgent 代码仓深度分析：基于 Mastra 的多智能体编排平台，以复合式工程和上下文工程为核心设计理念。

---

## 1. 概述

OmniAgent 是一个本地 Mastra 智能体团队（Agent Team），通过持久化协调协议将四个专业智能体编排在一起。它作为 Mastra 服务运行，以 LibSQL 支持的对话记忆和文件支持的 docs 系统作为规范长期记忆。Omni Gateway 作为独立的通道适配层，将移动消息应用接入 Team Runtime 协议。

**技术栈：** TypeScript, Mastra `1.32.1`, LibSQL, Vitest, Zod `4.x`
**模型供应商：** DeepSeek（通过 Mastra AI SDK）
**入口：** `src/mastra/index.ts` → `npm run dev`（Mastra Studio 默认运行在 `localhost:4111`）
**Gateway 入口：** `src/gateway/main.ts` → `npm run gateway`（HTTP 服务默认运行在 `localhost:4120`）
