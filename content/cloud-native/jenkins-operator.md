---
title: "Jenkins Operator —— Jenkins 在 Kubernetes 上打开的正确方式"
description: "Jenkins Operator 让 Jenkins 在 Kubernetes 上的安装使用变得 Kubernetes Native"
author: 马景贺（小马哥）
categories: ["Cloud Native"]
tags: ["cloud native","kubernetes"]
date: 2021-04-01T13:05:42+08:00
type: "post"
---

# 入门篇：jenkins-operator 的介绍及安装

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
接下来只需要将上述内容写入 `yaml` 文件创建 `configmap` 即可
```
$ kubectl -n  jenkins apply -f config.yaml
configmap/jenkins-config created
```
创建 jenkins 实例
```
$ kubectl -n jenkins apply -f jenkins.yaml
jenkins.jenkins.io/jenkins created
```
此时，可以在 operator pod 的 log 中看到如下信息
```
2021-04-04T08:21:44.049Z	INFO	controller-jenkins	base/pod.go:159	Creating a new Jenkins Master Pod jenkins/jenkins-jenkins	{"cr": "jenkins"}
```
查看 `jenkins` ns 下面的 pod
```
$ kubectl -n jenkins get pods -w
NAME                                     READY   STATUS    RESTARTS   AGE
jenkins-jenkins                          2/2     Running   0          13d
jenkins-operator-5cd7d8887c-klphr        1/1     Running   0          13d
```
如果想通过`ingress`来对外暴漏 jenkins 的服务，则可以按照下面的 yaml 文件来创建 ingress 资源
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
 name: jenkins
 annotations:
   kubernetes.io/ingress.class: "nginx"
   nginx.ingress.kubernetes.io/redirect-to-https: "True"
spec:
 tls:
 - hosts:
   - xiaomage.devops.com
   secretName: jenkins-tls
 rules:
 - host: xiaomage.devops.com
   http:
     paths:
     - path: /
       pathType: Prefix
       backend:
         service:
           name: jenkins-operator-http-jenkins
           port:
             number: 8080
```

接着通过访问 `https://xiaomage.devops.com` 来访问 jenkins 界面。登陆的用户名和密码可以通过如下命令获取：
```
$ kubectl -n jenkins get secrets jenkins-operator-credentials-jenkins -o jsonpath='{.data.user}' | base64 -D
kubectl -n jenkins get secrets jenkins-operator-credentials-jenkins -o jsonpath='{.data.password}' | base64 -D
```

此外，也可以通过 `port-forward` 的方式来使用 jenkins：
```
$ kubectl -n jenkins port-forward pods/jenkins-jenkins 8080:8080
```

直接访问 `http://localhost:8080` 即可访问 jenkins 界面。获取登陆用户名和密码的方法同上。

至此，通过 jenkins-operator 安装 jenkins 的过程已经完美实现，接下来是使用篇。

# 进阶篇：使用

传统的使用方法就是在界面上点击创建 jenkins job，然后进行配置，最后再使用。但是 jenkins-operator 提供了另外一个 operator—— Seed Jobs，顾名思义，就是能够实现 job 的自动发现。其背后的原理其实是借助 Jenkins Job DSL 和 Configuration As Code：也即将 job 通过 DSL 来进行描述（描述包括 Job 名称，配置，Pipeliine 脚本等），然后将这种描述的代码存放到 GitHub 上。Seed Jobs 根据配置来自动捕获 job 添加的动作，从而完成 job 的创建。

Seed Job 的使用前提是 job 定义文件和 job pipeline 文件需要具有如下的文件目录结构
```
cicd/
├── jobs
│   └── job-dsl-file
└── pipelines
    └── pipeline-file
```

Seed Job 可以通过在 `jenkins` 的配置文件中添加如下内容来启用
```
apiVersion: jenkins.io/v1alpha2
kind: Jenkins
metadata:
  name: jenkins
spec:
  master:
  # 可参考第一部分中的相关配置内容 
  seedJobs:
  - id: Demo
    targets: "cicd/jobs/demo_pipeline.groovy" # job dsl 脚本的位置
    credentialType: basicSSHUserPrivateKey
    credentialID: github-ssh-key
    description: "CI/CD Repo"
    repositoryBranch: main
    repositoryUrl: git@github.com:majinghe/jenkins-operator.git  # cicd 仓库地址
```
然后重新创建 jenkins 资源即可
```
$ kubectl -n jenkins apply -f jenkins.yaml
```

随后即可看到在 jenkins namespace 下面能看到 seed job 的pod
```
$ kubectl -n jenkins get pods
NAME                                      READY   STATUS    RESTARTS   AGE
jenkins-jenkins                           1/1     Running   0          122m # jenkins master pod
jenkins-operator-5cd7d8887c-tfhg4         1/1     Running   0          5h2m # jenkins operator pod
seed-job-agent-jenkins-7ff6d479db-6gqjl   1/1     Running   0          120m # see job pod
```

重新登陆 jenkins 就能看到两个 job，一个为Seed Job，一个为最终要真正使用的 Job。此后，只要 job 有修改，只需要修改 GitHub 上关于job的代码即可，然后重新运行 Seed Job 就能把实际使用 Job 的内容进行更新。其实也就做到了一切皆代码。一旦 jenkins 有任何问题，也可以通过重建来快速拉起相应的 job。相当于多一层备份机制（这个只能备份 job，job 历史会丢失，如果需要备份 job 历史，可以给 job 历史目录做持久化或者利用 jenkins-operator 的 backup 和 restore 机制，详细内容可以查看[这儿](https://jenkinsci.github.io/kubernetes-operator/docs/getting-started/latest/configure-backup-and-restore/))


# 高阶篇：利用 kustomize 来部署 jenkins-operator

上面的流程给大家展示了如何一步步来完成 jenkins-operator 的安装和使用，但是通过 `kubectl apply` 来一个个创建需要的资源谁比较繁琐的，而且在多套差异化环境下，这种重复的工作量没有任何意义。所以本文使用了 `kustomize` 来管理差异化环境下众多的 yaml 文件，目录结构如下
```
.
├── base
│   ├── config.yaml
│   ├── jenkins-rbac.yaml
│   ├── jenkins.yaml
│   ├── jenkins_crd.yaml
│   ├── jenkins_operator.yaml
│   ├── kops-secret.yaml
│   ├── kustomization.yaml
│   └── secret
│       └── credentials.secret.yaml
└── overlays
    ├── dev
    │   ├── ingress.yaml
    │   ├── jenkins.yaml
    │   ├── kops-secret.yaml
    │   ├── kustomization.yaml
    │   └── secret
    │       └── secret.tls.yaml
    ├── prod
    │   ├── ingress.yaml
    │   ├── jenkins.yaml
    │   ├── kops-secret.yaml
    │   ├── kustomization.yaml
    │   └── secret
    │       └── secret.tls.yaml
    └── svt
        ├── ingress.yaml
        ├── jenkins.yaml
        ├── kops-secret.yaml
        ├── kustomization.yaml
        └── secret
            └── secret.tls.yaml
```

关于 `kustomize` 的使用方法，大家可以看[这儿](https://kustomize.io/)。上述整个代码库在[这儿](https://github.com/majinghe/jenkins-operator)。文中使用了 [sops](https://github.com/mozilla/sops) 来加密 yaml 文件中的敏感信息，这样真正能够做到将一切代码化，然后托管到 GitHub 上。关于 sops 的使用敬请期待后面的文章。
