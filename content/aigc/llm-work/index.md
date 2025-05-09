---
title: "GitLab MCP Server tools 功能扩展实战"
description: "使用 typescript 扩展 GitLab MCP Server tools"
author: 马景贺（小马哥）
image: "images/deepseek-vs-chatgpt-gemini.png"
categories: ["AIGC"]
tags: ["AI","MCP","GitLab","TypeScript"]
date: 2025-05-01T10:05:42+08:00
type: "post"
---


最近在 x 上看到 Akshay 分享的一组图解 LLM 工作原理的帖子，感觉内容通俗易懂，就搬运过来汉化一下，方便大家一起学习！

Akshay 是一位 AI/ML 工程师，他在 x 上的介绍如下图所示：

![akshay](images/akshay.png)

## LLM 工作原理解释

### 条件概率解释

他提到，在介绍 LLM 之前，需要先了解一下**条件概率**（conditional probability），应该是与高中、大学学的概率学相关。有一个很形象的例子：

有 14 个人，他们中的一部分人（7 个）喜欢网球、一部分人（8个）喜欢足球、少部分人（3 个）同时喜欢网球和足球、也有极少一部分人（2 个）都不喜欢网球和足球。用图表示如下：

![conditional probability](images/conditional-probability-1.jpeg)

所以如果要表示喜欢网球的人数概率，表示方法为 P(A)，结果是 7/14；喜欢足球的人数概率，表示方法为 P(B)，结果为 8/14；同时喜欢网球和足球的人数概率，表示方法为 P(A∩B)，结果是 3/14；同时表示既不喜欢网球又不喜欢足球的人数概率，表示方法为 P(AUB)，结果为 2/14。

那什么条件概率呢？

其实就是在另外一件事情发生的前提下，某件事情发生的概率。比如上面的事件 A 和事件 B，如果要表示在事件 B 发生的前提下，事件 A 发生的概率，那么表示方法是P(A∣B)。

所以，如果要计算一个人在喜欢足球的情况下，还喜欢网球的概率，计算方法为 P(A|B)=P(A∩B)/P(B)=(3/14)/(8/14)=3/8。

![conditional probability 2](images/conditional-probability-2.jpeg)

再拿阴天和下雨天为例来将条件概率：如果将今天下雨当作事件 A，阴天可能下雨作为事件 B（尝试来讲，阴天会有下雨的可能），而且事件 B 会影响下雨的预测。所以，阴天的时候就可能会下雨，这个时候就可以说条件概率 P(A|B) 是非常高的。

### LLM 预测解释

回到 LLM 上来说，这些模式的任务就是预测下一个出现的单词。这就和前面讲的条件概率类似：**如果给定已经出现过的单词，那下一个最可能出现的单词是哪一个？**

![conditional probability 3](images/conditional-probability-3.jpeg)

所以，要预测下一个单词，**模型就要根据之前给定的单词（上下文）来为每一个接下来可能出现的单词进行条件概率的计算，条件概率最高的单词就会被作为预测单词所选中**。

![conditional probability 4](images/conditional-probability-4.jpeg)