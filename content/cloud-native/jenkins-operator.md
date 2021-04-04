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

关于两种方式的不同使用命令，可以在[这儿](https://jenkinsci.github.io/kubernetes-operator/docs/installation/)来查看，本文选择用 `kubectl apply` 来完成。

创建一个 `jenkins` namespace
```
$ kubectl create ns jenkins
```

安装 Operator
```
$ kubectl  -n jenkins apply -f jenkins_operator.yaml
serviceaccount/jenkins-operator created
role.rbac.authorization.k8s.io/jenkins-operator created
rolebinding.rbac.authorization.k8s.io/jenkins-operator created
deployment.apps/jenkins-operator created
```
查看 `jenkins` namespaace 下面的 `pod`
```
$ kubectl -n jenkins get pods -w
NAME                                READY   STATUS    RESTARTS   AGE
jenkins-operator-548d76f664-hp6pm   0/1     Pending   0          0s
jenkins-operator-548d76f664-hp6pm   0/1     ContainerCreating   0          1s
jenkins-operator-548d76f664-hp6pm   1/1     Running             0          34s
```
### Step 3：创建 Jenkins 实例

此时如果创建一个 Jenkins 实例，Operator 就会监听到这个事件，从而根据我们对 Jenkins 实例的描述（Jenkins 实例的描述文件 ，以 yaml 格式出现）来创建实例。关于实例的描述文件的格式内容，可以在[这儿](https://jenkinsci.github.io/kubernetes-operator/docs/getting-started/latest/schema/)进行查看。本文使用的文件即内容解释如下
```
apiVersion: jenkins.io/v1alpha2
kind: Jenkins
metadata:
  name: jenkins
spec:
/*configurationAsCode 用来描述 jenkins configruation 部分的内容，比如 Github Server、Slack、LDAP 等的配置。上述内容通过 Jenkins Configuration As Code 的形式来组织的。下文会给大家展示用法*/ 
  configurationAsCode:
    configurations:
    - name: jenkins-config
  serviceAccount:
    annotations:
      kubernetes.io/service-account: jenkins
  master:
    basePlugins:
    /*配置与 master 相关的一些配置，比如想要安装的必要插件*/
    - name: kubernetes
      version: "1.29.2"
    plugins:
    /*配置与 master 相关的一些配置，比如想要安装的其他插件*/
    - name: ldap
      version: "2.4"
    - name: github
      version: "1.33.1"
    containers:
    /*master 容器的配置，包括镜像、资源限制、环境变量等*/
    - name: jenkins-master
      image: jenkins/jenkins:lts
      imagePullPolicy: Always
      env:
      - name: JENKINS_HOME
        value: /var/lib/jenkins
      resources:
        limits:
          cpu: 1500m
          memory: 2Gi
        requests:
          cpu: "1"
          memory: 500Mi
```

上述文件依赖于 `jenkins-config` 这个对 jenkins configruation 的描述，需要先创建 `jenkins-config` 这部分内容，此内容是以 `configmap` 的形式来描述 jenkins configuration 的配置的，诸如`LDAP、GitHub Server` 等。具体内容可参考如下
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: jenkins-config
data:
  jenkins.yaml: |
    security:
      scriptApproval:
      /*Script Approval 配置部分*/
        approvedSignatures:
        - "method groovy.json.JsonSlurper parse java.io.Reader"
    unclassified:
      slackNotifier:
      /*slack 配置部分*/
        teamDomain: slack-test 
        tokenCredentialId: "slack-token"
      gitHubPluginConfig:
      /*GitHub Server 配置部分*/
        configs:
          - name: "GitHub"
            apiUrl: "https://api.github.com/v3/"
            credentialsId: "github-token"
            manageHooks: true
    jenkins:
      clouds:
      /*kubernetes 配置部分*/
      - kubernetes:
          jenkinsTunnel: "jenkins-operator-slave-jenkins.jenkins.svc.cluster.local:50000"
          jenkinsUrl: "http://jenkins-operator-http-jenkins.jenkins.svc.cluster.local:8080"
          name: "kubernetes"
          namespace: "jenkins"
          retentionTimeout: 15
          serverUrl: "https://kubernetes.default.svc.cluster.local:443"
      systemMessage: "<Cloud Native DevSecOps>"
      securityRealm:
        ldap:
        /*LDAP 配置，根据你的 LDAP 配置信息修改即可*/
          configurations:
            - server: "xxxx"
              rootDN: "xxxx"
              userSearchBase: "xxxx"
              userSearch: "xxxx"
              groupSearchBase: "xxxxxx"
              groupMembershipStrategy:
                fromGroupSearch:
                   filter: ""
```
