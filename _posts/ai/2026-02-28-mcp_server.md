---
title: MCP 服务原理
date: 2026-02-28 00:00:00 +0800
categories: [ai]
tags: [ai]
author: ahern
---

## MCP 是什么

![img.png](./assets/images/img_65.png){:height="70%" width="70%"}

MCP（[Model Context Protocol](https://modelcontextprotocol.io/docs/getting-started/intro)）是一种用于 AI 应用和外部系统通讯协议。

## MCP 客户端
大模型、IDEs 等 AI 应用本质上都是集成了 MCP Client。

## MCP 服务端
MCP Server 负责处理来自 MCP Client 的请求，并与外部系统进行交互。可以提供 3 种能力：
- Tools：LLM 可以调用的函数（需经用户批准）
- Resource：可供客户端读取的类文件数据（例如 API 响应或文件内容）
- Prompt：预先编写的模板，可帮助用户完成特定任务


<br>
client-server 在连接初始化时，会进行能力协商各自支持哪些能力。
不支持的能力，客户端不会主动调用`xx/list`接口获取能力列表。

### [Tools](https://modelcontextprotocol.io/docs/learn/server-concepts#tools)

Tools 是允许 AI 应用调用的函数集合。每个被调用函数带有描述，供 AI 应用理解其功能和使用方法。调用函数需要用户批准，确保安全性。


<br>
操作：
- `tools/list`：列出可用工具
- `tools/call`：调用工具函数
- `notifications/tools/list_changed` 通知client工具变更


<br>
使用场景：
- 数据查询：提供对数据库增删改查能力。
- 任务执行：提供执行特定任务的工具函数，例如发送电子邮件、创建日历事件或执行计算。
- 系统交互：提供与操作系统交互的工具函数，例如文件操作、网络请求等。

### [Resource](https://modelcontextprotocol.io/docs/learn/server-concepts#resources)

提供结构化的数据访问，AI 应用可以检索这些信息并将其作为上下文提供给模型。


<br>
操作：
- `resources/list`：列出可用的直接资源
- `resources/templates/list`：发现资源模板
- `resources/read`：读取资源内容
- `notifications/resources/list_changed`：通知client资源变更
- `resources/subscribe`：监控资源变化


<br>
使用场景：
- 日历数据（calendar://events/2024）- 检查用户可用性
- 旅行证件（file:///Documents/Travel/passport.pdf）- 获取重要文件
- 以往行程（trips://history/barcelona-2023） - 参考过去的旅行和偏好

### [Prompt](https://modelcontextprotocol.io/docs/learn/server-concepts#prompts)
Server 存储一些预定义可复用的 Prompts 模板，通过客户端的参数，生成定制化的Prompt。


<br>
操作：
- `prompts/list`：列出可用的 Prompt 模板
- `prompts/get`：获取 Prompt 模板内容
- `notifications/prompts/list_changed`：通知client Prompt变更


<br>
使用场景：
- 团队开发者用 Prompts 能力，生成风格统一代码
- 生成旅行计划，统一接受用户输入的目的地、日期、预算等参数，生成定制化的 prompt

## 传输层

### Stdio

一般用于本地 Demo 部署，Server 通过 Stdin 接受 JSON‑RPC 请求，以 Stdout 返回响应。

### HTTP + SSE（Server-Sent Events）（弃用）
![img.png](./assets/images/img_66.png){:height="70%" width="70%"}

核心设计是基于 Http 长连接，Server 主动向 Client 推送事件，依赖 2 个端点：
- Post 请求：短连接，Client 发起请求，Server 异步处理
- Get 请求：长连接，Client 发起请求，Server -> Client，轮询向 Client 推送事件 

> 为什么需要 2 个端点？
> 
> SSE 模式是一个 request → 一个长时间 response，在 response 结束前，无法发起第二个请求；
> 所以需要拆分 2 个端点，Post 请求用于发起请求，Get 请求用于接收事件。

### Streamable HTTP（推荐）
![img.png](./assets/images/img_67.png){:height="70%" width="70%"}

基于 Http，只需要 1 个端点，按需升级流式返回，引入 session 机制以支持会话管理。

### HTTP + SSE 对比 Streamable HTTP

| | HTTP + SSE | Streamable HTTP |
| --- | --- | --- |
| 连接复用 | 不复用，高并发场景消耗连接数 | 连接复用 |
| 稳定性 | 中；依赖长连接，需要调整防火墙超时阈值 | 高 |
| 实现复杂度 | 高；依赖 2 个端点，客户端实现复杂 | 低；只依赖 1 个端点 |

## Debug 工具
[interceptor](https://github.com/modelcontextprotocol/inspector)
