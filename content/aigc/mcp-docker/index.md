---
title: "Docker 发布 MCP Catalog & ToolKit，意欲打造 MCP 单一可信源？"
description: "Docker 发布 MCP & ToolKit 计划，积极推进 MCP 的标准化和安全化发展。"
author: 马景贺（小马哥）
image: "images/deepseek-vs-chatgpt-gemini.png"
categories: ["AIGC"]
tags: ["AI","MCP","Docker"]
date: 2025-04-24T13:05:42+08:00
type: "post"
---

4 月 22 日，Docker 正式宣布开启 **Docker MCP Catalog and Toolkit** 计划，官宣的完整内容通过名为 《Dockerizing MCP – Bringing Discovery, Simplicity, and Trust to the Ecosystem》的博客发布，大概内容如下。


## MCP 是真牛逼

AI agent 的发展很猛，已经从实验室走向大众化，而且从简单的生成文本演进到可以使用工具了，而这个过程中 MCP（Model Context Protocol）已经成为 agent 链接 tool 的**事实标准（de facto standard）**。Docker 相信 MCP 能够对 agentic AI 的交互产生重要影响 —— 将复杂且碎片化的领域进行标准化、简单化（是不是很熟悉？）

## MCP 存在以下问题

MCP 虽牛逼，但是目前还不具备生产就绪的能力，主要存在**发现碎片化、信任手动化、安全缺失化**一类的问题。要想让 MCP 从原型走向生产，就必须要去做一些事情，比如：

- **对于开发者来说**，需要一个可信的（trusted）、集中式的 hub（centralized hub）来很方便的发现自己想要的 MCP，这样就不用到处去找了；
- **对于工具作者来说**，一个可信、集中式的 hub 是一个**非常重要的分发渠道**，除了能够触达更多的新用户外，还能够确保在不同平台上的兼容性，比如 Claude、Cursor、OpenAI 以及 VS Code；
- **默认容器化**，下载仓库代码（MCP 仓库）并处理依赖不是开发者的目标，这些是没有必要去做的；
- **凭据管理**，凭据管理应该简单且安全，这种管理应该是**集中化、经过加密而且为适配现代化工作流而构建**；
- **安全基础**，要对 MCP 进行沙箱化，进行权限管理 & 安全审计，不能做事后诸葛亮，这些需要在第一天（Day one）就开始构建。此外，安全还需要简单易用，降低开发者的使用门槛。

总结来说，MCP 虽好，但是需要解决**可信、可发现、易用、安全**的问题。

Docker 一直擅长此事，所以它来了，这一波要支棱起来了！

## 为什么 Docker 特别适合干这个

现在的 MCP 时刻就跟多年前云计算和容器的早期时候很像 —— **潜力很大，虽然还有些许瑕疵，但是也意味着巨大机遇**。这种情况在每一轮新技术的发展中都会遇到，Docker 是有经验来解决这种问题的。比如云计算早期，Docker 通过将不可变性和隔离性确立为标准，内置身份验证功能，并推出 Docker Hub 作为中央发现层，为混乱无序的局面带来了秩序。这不仅仅是简化了研发，而是重新定义了软件的构建、共享和可信。 

> 原文：Docker brought structure to chaos by making immutability and isolation the standard, building in authentication, and launching Docker Hub as a central discovery layer.

发展这么多年，Dockerhub 已经成为**容器镜像的单一可信源**，那针对 MCP 这一波，Docker 同样有机会为 agent 和真实世界的自动化开创一个新时代，那核心手段就是**Docker MCP Catalog 和 MCP ToolKit**。

## Docker MCP Catalog 和 Docker MCP ToolKit 到底要干啥

**Docker MCP Catalog 将作为发现 MCP 工具的可信源**（原文： the Docker MCP Catalog will serve as the trusted home for discovering MCP tools），而且和 Dockerhub 是无缝集成的。目前这事还没完成，也不可能凭 Docker 一己之力来完成，还是要众人拾柴火焰高，所以目前 Stripe、Elastic、Heroku、Pulumi、Grafana Labs、Kong Inc.、Neo4j、New Relic、Continue.dev 都参与了，大家一起搞。将来发布的时候（大概 5 月份），将有超过 100 个已验证的工具可用，而且每一个工具的发布者都是验证过的、发布是通过版本化的，最终方便开发者能快速找到自己想要的工具。MCP 工具的分发和现在的 Docker 镜像是同一个方式 —— 经过 Docker 验证的基于拉取的基础设置（Docker’s proven pull-based infrastructure），这套设施支持每个月超10亿次的下载。

> 目前，MCP Catalog 的链接已经有了：https://hub.docker.com/catalogs/mcp

**Docker MCP ToolKit 是让这些工具发放异彩**（原文：Docker MCP Toolkit brings these tools to life），它能够让工具更加安全，而且只要有 Docker 的地方，这些工具就能够无缝运行。将来，只需要在 Docker Desktop 上简单点击就能启用一个 MCP Server，然后快速连接到客户端，整个过程都不需要再进行额外配置。更重要的是，内置了凭据和 OAuth 管理，而且和 Dockerhub 账号打通了，这就确保有很丝滑的认证，当然，在发现问题的时候还能够撤销凭据。另外，还有一个 Gateway MCP Server 能够将启用的工具暴露给兼容的客户端，同时还会配备一个 docker mcp 命令行来让用户构建、运行和管理这些工具。

所以，Docker MCP Catalog 和 Docker MCP ToolKit 的蓝图大概是**在 Dockerhub 上有数以百计的可用 MCP Server，很容易就能够启用它们而且连接到 agent 上，再也不需要对凭据进行硬编码，也不需要通过 npx 或 uvx 来启用工具，更不需要担心安全性。使用曾经熟悉的命令就能够玩转 MCP 了，零学习成本，无限可能**。

## 写在最后

最近在折腾 MCP 的时候，预感 MCP 会满天飞，如何解决**复用、可信、安全**的问题是让 MCP 真正走向生产的关键。Docker 确实在这方面有先天优势，就看这一波 MCP 的发展是否能让 Docker 再次支棱起来，我们拭目以待！