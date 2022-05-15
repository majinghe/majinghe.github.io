---
title: "使用 cosign 来签名和验证 OCI 镜像，并和 Tekton CI/CD 集成"
description: "cosign 用来对 OCI 进行签名和验证，从而保证镜像的安全，同时和 CI/CD 集成，打造 DevSecOps。"
author: 马景贺（小马哥）
image: "images/gotc-ospo.jpg"
categories: ["DevSecOps"]
tags: ["Security","Image","Cosign"]
date: 2022-05-06T13:05:42+08:00
type: "post"
---

## 关于 cosign

cosign 是 `sigstore` 项目的一部分，也是一个开源项目，主要用来对 OCI 镜像进行签名和验证，从而让签名成为一种无形的基础设施。cosign 目前支持以下几种签名方式：


* 硬件和 KMS 签名
* 使用自己的 PKI
* 免费的 OIDC PKI
* 内置的二进制透明和时间戳服务


## cosign 安装

cosgin 有多种安装方式，详细安装可以查看 [GitHub Repo 安装指南](https://github.com/sigstore/cosign)，以 macOS 为例，执行如下命令即可完成安装：

```
$ brew install cosign
```

使用 `version` 命令查看安装：

```
$ cosign version
GitVersion:    1.4.1
GitCommit:     934567a4c606cf59e6ab17af889b4db3ee0a3f0b
GitTreeState:  "clean"
BuildDate:     2021-12-10T16:54:56Z
GoVersion:     go1.17.3
Compiler:      gc
Platform:      darwin/amd64
```

## cosign 的使用

使用 `cosgin` 来对镜像签名，一般有三步：

### 第一步：生成 keypair

使用如下命令即可完成 keypair 的生成：

```
$ cosign generate-key-pair
Enter password for private key:
Enter password for private key again:
Private key written to cosign.key
Public key written to cosign.pub
```

根据提示输入两次生成 private key 的 password 即可。在当前目录下可以看到如下两个文件：

```
-rw-------  1 xiaomage  staff    649 May  6 23:23 cosign.key
-rw-r--r--  1 xiaomage  staff    178 May  6 23:23 cosign.pub
```

两者的内容如下：

```
$ cat cosign.pub
-----BEGIN PUBLIC KEY-----
MFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAErZce/WXwxFR51Ol07BsVrMGhLOXY
9IHEXkNTwrfsUuIKb/P7hgDewfR9zc55Gvyi9xV7TvzKCuwL17gXN6pcGQ==
-----END PUBLIC KEY-----

$ cat cosign.key
-----BEGIN ENCRYPTED COSIGN PRIVATE KEY-----
eyJrZGYiOnsibmFtZSI6InNjcnlwdCIsInBhcmFtcyI6eyJOIjozMjc2OCwiciI6
OCwicCI6MX0sInNhbHQiOiJ6VkExYy9YcUJnc1JaZEFsbSsxT29zQXNhbFVzSGE5
Uk52VlFUYVk1Z0JvPSJ9LCJjaXBoZXIiOnsibmFtZSI6Im5hY2wvc2VjcmV0Ym94
Iiwibm9uY2UiOiJWTzJ2akkwMUtpREpvWll0MWg4WVpTd0I4ZlVkTTE4bCJ9LCJj
aXBoZXJ0ZXh0IjoiVTAvekhacXdZMThwdzFtL0VIZ1huNzRHanIreDQ3NUVCTGNw
RzAyR3ArVDZpaFFTL3lxYlJwZWtQdDRZWGdyakZPUEgxc2lkN1AxdXFQUmlLZDBa
R2haenByWExTVXo3djI4ZUw3OERJNE1oS2ZjQldxL1VXSG00d3E0TVBIZE5IWm54
WVVmK3FETW1EbW1GOVBkV3NwOVAxK2NQaTUxWGpvMHJMbkI2amNseXpQVnQvcUdz
N2tUclhGSDI0VHZDbng1TDFHcFhrb1Jpc2c9PSJ9
-----END ENCRYPTED COSIGN PRIVATE KEY-----
```

如果将 private key 所需的 password 以环境变量 `COSIGN_PASSWORD` 的形式注入的话，则可以不用手动输入 password，整个过程会自动读取环境变量并使用。

### 第二步：使用 private key 签名镜像

以 `dllhb/go-demo:v1.0.0` 为例来演示，使用如下命令来签名：

```
$ cosign sign --key cosign.key dllhb/go-demo:v1.0.0
Enter password for private key:
Pushing signature to: index.docker.io/dllhb/go-demo
```
可以在镜像仓库中看到镜像发生了如下变化：

![cosign-image](images/cosign-image.png)

### 镜像验证

使用如下命令即可对拉取的镜像进行签名验证：

```
$ cosign verify --key cosign.pub dllhb/go-demo:v1.0.0 | jq .
Verification for index.docker.io/dllhb/go-demo:v1.0.0 --
The following checks were performed on each of these signatures:
  - The cosign claims were validated
  - The signatures were verified against the specified public key
  - Any certificates were verified against the Fulcio roots.
[
  {
    "critical": {
      "identity": {
        "docker-reference": "index.docker.io/dllhb/go-demo"
      },
      "image": {
        "docker-manifest-digest": "sha256:888988496480bb0be8984b43a84970589e41110a2d25440f145031fd396dd2db"
      },
      "type": "cosign container image signature"
    },
    "optional": null
  }
]
```
而且上述命令返回 `0` 值。返回的 payload 里面有镜像的 digest 信息，和镜像仓库中展现的一致，说明镜像没有被篡改。如果被篡改了，那么验证结果是什么样的呢？

### 篡改镜像做验证

我们将上述镜像做一下篡改（修改里面应用程序的内容，然后以同样的 tag 构建并推送上去），然后用同样的方式去验证：

```
$ cosign verify --key cosign.pub dllhb/go-demo:v1.0.0 | jq .
Error: no matching signatures:

main.go:46: error during command execution: no matching signatures:
```

可以看到报错了，提示无法找到相匹配的签名。说明镜像有被篡改过，侧面反应了 cosign 是如何通过镜像签名来保证镜像安全的。


## 将镜像签名嵌入 Tekton CI/CD

只需要在 Tekton CI/CD 中添加一个额外的 `Step` 即可完成镜像的签名：

```
    - name: image-singature-verification
      image: bitnami/cosign:latest
      args:
        - verify
        - --key
        - /tmp/cosign.pub
        - $(resources.outputs.docker-image.url):$(params.image-tag)
      volumeMounts:
      - name: cosign-pub
        mountPath: /tmp/cosign.pub
        subPath: cosign.pub
  volumes:
    - name: cosign-pub
      secret:
        secretName: cosign-pub
```
此外，还可以将镜像构建、镜像扫描、镜像签名和镜像验证都集成都 Tekton CI/CD 中，来构建 DevSecOps CI/CD：

![image-sign-verify](images/image-devsecops.png)

整个流程中 Tekton CI/CD 代码如下：

```
  steps:
    - name: image-build-and-push
      image: gcr.io/kaniko-project/executor:v1.6.0
      env:
        - name: "DOCKER_CONFIG"
          value: "/tekton/home/.docker/"
      command:
        - /kaniko/executor
      args:
        - --dockerfile=$(resources.inputs.source-code.path)/Dockerfile
        - --destination=$(resources.outputs.docker-image.url):$(params.image-tag)
        - --context=$(resources.inputs.source-code.path)
    - name: image-scan
      image: aquasec/trivy
      env:
        - name: TRIVY_USERNAME
          valueFrom:
            secretKeyRef:
              name: dockerhub-user-pass
              key: username
        - name: TRIVY_PASSWORD
          valueFrom:
            secretKeyRef:
              name: dockerhub-user-pass
              key: password
      script: |
        #! /bin/sh
        trivy image --exit-code 1 --severity CRITICAL --vuln-type library  $(resources.outputs.docker-image.url):$(params.image-tag)
    - name: image-singature
      image: bitnami/cosign:latest
      securityContext:
        runAsUser: 0
      args:
        - sign
        - --key
        - /tmp/cosign.key
        - $(resources.outputs.docker-image.url):$(params.image-tag)
      env:
      - name: COSIGN_PASSWORD
        valueFrom:
          secretKeyRef:
            name: cosign-password
            key: cosign-password
      volumeMounts:
      - name: cosign-key
        mountPath: /tmp/cosign.key
        subPath: cosign.key
    - name: image-singature-verification
      image: bitnami/cosign:latest
      args:
        - verify
        - --key
        - /tmp/cosign.pub
        - $(resources.outputs.docker-image.url):$(params.image-tag)
      volumeMounts:
      - name: cosign-pub
        mountPath: /tmp/cosign.pub
        subPath: cosign.pub
  volumes:
    - name: cosign-pub
      secret:
        secretName: cosign-pub
    - name: cosign-key
      secret:
        secretName: cosign-key
```

Step 说明：

* **image-build-and-push**：用 `kaniko` 构建容器镜像，完成从源码到镜像的转换；

* **image-scan**：用 `trivy` 进行镜像扫描；

* **image-singature**：用 `cosign sign` 来完成镜像的签名；

* **image-singature-verification**：用 `cosgin verify` 来镜像签名验证；

> 签名和验证所需的 public 和 private key 都用 Kubernetes Secret 的形式存储，并以 `volume` 的形式进行了挂载。

用 `TaskRun` 触发一个 `Task`，可以查看两者的状态：

```
$ tkn -n tekton-cosign tr list
NAME                     STARTED         DURATION   STATUS
build-docker-image-run   3 minutes ago   1 minute   Succeeded
tkn -n tekton-cosign t list
NAME                 DESCRIPTION   AGE
build-docker-image                 4 minutes ago
```

> 整个演示过程的 Tekton 代码在[这个 tekton cosign demo GitHub Repo 中](https://github.com/majinghe/tekton-demo/tree/main/cosign-demo)。

如果查看 `TaskRun` 的内容，可以清晰的看到所有 Step 的状态：

```
$ tkn -n tekton-cosign tr describe build-docker-image-run
......
🦶 Steps

 NAME                              STATUS
 ∙ create-dir-docker-image-vvj5b   Completed
 ∙ git-source-source-code-bcxft    Completed
 ∙ image-build-and-push            Completed
 ∙ image-scan                      Completed
 ∙ image-singature                 Completed
 ∙ image-singature-verification    Completed
 ∙ image-digest-exporter-nmjct     Completed
......
```

以及整个构建过程中 `pod` 的状态：

```
$ kubectl -n tekton-cosign get pods
NAME                         READY   STATUS      RESTARTS   AGE
build-docker-image-run-pod   0/7     Completed   0          4m30s
```

如果要查看某一个过程，直接用 `logs -f ` 查看 log 即可。


因此，可以使用 `cosign` 来完成镜像的签名和验证、用 `trivy` 来进行容器镜像扫描，多种手段嵌入到 Tekton CI/CD 中来保证容器镜像安全。

