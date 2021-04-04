---
title: "Jenkins Operator —— Jenkins 在 Kubernetes 上打开的正确方式"
description: "Jenkins Operator 让 Jenkins 在 Kubernetes 上的安装使用变得 Kubernetes Native"
author: 马景贺（小马哥）
categories: ["Cloud Native"]
tags: ["cloud native","kubernetes"]
date: 2021-04-01T13:05:42+08:00
type: "post"
---

## 前言

Operator 是 Kubernetes 的一种扩展机制，用户可以利用这种扩展机制来让自己的应用以 Kubernetes native（k8s 原生）的方式在 kubernetes 平台上运行起来。关于 Operator 更多详细的内容，可以在[Kubernetes 官方文档](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)上查看。

Jenkins 是一款社区强大、API & 插件丰富、用户众多且开源的持续交付工具。为了让 Jenkins 能在 Kubernetes 更好的运行，Jenkins 社区提供了 Jenkins Operator。Jenkins Operator 是基于不可变性和声明性配置即代码来构建的。关于 Operator 的其他信息可以在[Jenkins Operator GitHub](https://github.com/jenkinsci/kubernetes-operator)和[Jenkins Operator 官网文档](https://jenkinsci.github.io/kubernetes-operator/)进行查看。

## 安装

安装的前提条件：

* 一个版本为 `1.11+` 的 Kubernetes 集群
* `kubectl` 的版本为 `1.11+`

### Step 1：CRD（Custom Resource Definition）配置

```
$ kubectl apply -f jenkins_crd.yaml
customresourcedefinition.apiextensions.k8s.io/jenkins.jenkins.io created
customresourcedefinition.apiextensions.k8s.io/jenkinsimages.jenkins.io created
```

### Step 2：Operator 安装

Jenkins Oprator 的安装有两种方式：

* 用 `kubectl apply` 来完成
* 用 `helm install` 来完成

关于两种方式的不同使用命令，可以在[这儿](https://jenkinsci.github.io/kubernetes-operator/docs/installation/)来查看，本文选择用 `helm install` 来完成。

创建一个 `jenkins` namespace
```
$ kubectl create ns jenkins
```

添加 helm repo

```
$ helm repo add jenkins https://raw.githubusercontent.com/jenkinsci/kubernetes-operator/master/chart
```

> 本文使用 helm 3.x 版本。

安装 Operator
```
$ helm install jenkins --namespace jenkins jenkins/jenkins-operator
```
