---
title: "Kubernetes secrets 加密的几种方式"
description: "处理好 Kubernetes secretes 是实现 GitOps 的关键"
author: 马景贺（小马哥）
categories: ["Security"]
tags: ["cloud security","security"]
date: 2021-2-5T13:05:42+08:00
type: "post"
---


## Sealed Secrets

Sealed Secrets 是加密 kubernetes secret 的一种方式，充分利用 kuberntes 的高扩展性，通过 CRD 来创建一个 `SealedSecret` 对象，然后将 secret 加密变成一个 `SealedSecret` 对象，而 `SealedSecret` 只能够被运行于目标集群上的 controller 解密。其他人员和方式都无法正确解密原始数据。可以将加密后的文件直接推送至版本控制系统，而不用担心敏感信息被泄漏。

### 安装

Sealed Secrets 有两部分组成：

* 安装与集群侧的 `controller/operator`
* 客户端工具：`kubeseal`

所以安装也分两步，安装`controller`和`kubeseal`。安装的方法在[这儿](https://github.com/bitnami-labs/sealed-secrets/releases)可以找到，可以先安装`controller`，执行如下命令即可：

```
$ $ kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.14.1/controller.yaml
```

随后查看`kube-system` ns 下面的`controller` pod
```
$ kubectl -n kube-system get pods | grep seal
sealed-secrets-controller-64b74f67b-4wtj7   1/1     Running   0          153m
```

接着安装客户端工具 `kubeseal`。这个可以根据自己的 OS 来选用不同的安装方式，以 MacOs 为例：
```
$ brew install kubeseal
```

通过查看 `kubeseal` 的版本来确定是否安装成功：
```
$ kubeseal --version
kubeseal version: v0.14.1
```

此时，两部分均安装成功，下面可以使用了。

### 使用

选择一个包含有需要加密的 secret 的文件，内容如下：
```
apiVersion: v1
data:
  username: YXNkZg==
  password: cGFzc3cwcmQ=
  token: MWU0ZGdyNWZncmgzcmZmZ3JodG9ubmhyaHI=
kind: Secret
metadata:
  name: seal-test-secret
  namespace: test
type: Opaque
```

将上述内容写入一个 yaml 文件，比如 test-secret.yaml，然后执行如下命令加密：
```
$ kubeseal < test-secret.yaml > test-seal-secret.yaml
```
可以查看 test-seal-secret.yaml 文件的内容
```
{
  "kind": "SealedSecret",
  "apiVersion": "bitnami.com/v1alpha1",
  "metadata": {
    "name": "seal-test-secret",
    "namespace": "test",
    "creationTimestamp": null
  },
  "spec": {
    "template": {
      "metadata": {
        "name": "seal-test-secret",
        "namespace": "test",
        "creationTimestamp": null
      },
      "type": "Opaque"
    },
    "encryptedData": {
      "password": "AgCHLFSlGFpX2B9QDhWbMTfT83aopXMisR5XnUPZNcbvvnQzqgyG8fBVknT8LCNF5ExtUCCcNLsvWRrZY+9BJqf5dlBl6DkLV1acuPicP0vuGaUQwmc5BY/5Bj53Oj9uMYLNdVHoQ3E6kQgeJPa5v4rvwRXsB0EneYPcT88KyMg+tn4OY9JH+hpg2XMXZudyyZsocE852J5nfN4P7WZQYaG2eIBqRSQvQXUflQQvZ5wBCkTvmaZYfxz+Lxuf3wDWdSlPjcgSVnn5tWNP7u0ErdVy8LwTL5HzJdcoSviDysTq3VVA8W9Nmn0CM3QS0R0Nbi3JfalUdxfBMK+yb7t6Z72oJyoxGfCa07iKnkM37SSw4vl1nXiYy3FMWuzDtWLVk6XzjBZR2ChoeClqbGDg8KeSWZg/rO3Xku98vCmCa004OetJKBMc9Db3q+gX53ThdU70VvRol9rLPFBPHB8NTjD+Bu0Ss4XzIzZzi8J+Ov5xE7G8LnPLSZtZQyD/qGZK4n7pU1YNLROJ+fz1W5edPdpb5szUOqs1bpFfGleUiPZo1sGA0f1EsDvJShptgtT44YzGRkkgrP1LGp2AVIpnt9meE5WNCoSEPZJVx7wWMV9CHMOyyUi8zi+oG/S2NkI3rc2sC8AFp0DqP9m/HaX1GG+6vw9oHAbhxpR4v3mDyBIq+/8UEMtkybIEDQGHqyQ5CfRow+A80cA4Hw==",
      "token": "AgBRi4NZunaJtHyP5aAoWmGtEXBipbFIb/n4ep8wdg+eka5xbDeLZwNCLofbUL+u0pP/CHSJeWl62mVPJhZdOKE+Su0b8a78im0+xsochaMQf0AI1GGL/Fo08HI8paP8k404gwAtonocIFSis3YooU0nyVD+lYH+k09FGABI+RmVLc2XkuIr96TTL4xsdSM5L0Ks2SFQKcQ42JfFWtNdXz6lr/IODsZop0/xAk6ffbsGGmCUjwusUU3Wp0MR25ntYT8ySuO6W7xkfGozEFzztteBJs28SHLf5HUi6BbYVnsZibrFF3BZP5aNnBg2TIgo3+dbX6EPHM904By3Z9XTBxsQfH6p1VoyUf0EGKZnUnJFezFtN9m2tyKbV/Z/5vCh9kVp6Yn/BE/AwGAH7kqqjPtHTnZiq+Xy1UwV4/eHkxGAvSAR3Z6wTQCt/rwqGrQi2eGpIcyjxTwlPYaVjfx3L+1tnBR966lGLnhwX0I6b6whXAm3hRb1AhYFnuyF/CoG/PEmQsMU5GfkroQkb5LL+UeCYKbjvMvgCe2hFxfh3dcGJ9E3dad9W0rSKrPd5t/dR1kDtItHau36+G9PSVyqRD1yt0MS2vLLUQu7t2RhiIPrl20fkbnum9JAfmLlgliHIiQPHASL32CXB6EzsgqRX6w8TmWNOSvlR7LU8JZtd4Gmiw9wGBh7JEGodkaH6lc5ndQluykC18RUXtuLft+S4dnQCApHX6FoIGZjug==",
      "username": "AgCgBQc/fhGqB0YBGfXzhybC6YXJeLkOZyi7Z7Y+HjfnYSg4Q/Zh8Kn7UEbq9CwEl+CtagARjKmLfhIcAqFWS8+h8j4A2xNq7gzLnv+eCo0vFDPTddDVvdb6ixmRvF5rzD1gZ2vxWzlWVqk7x0wt8wCE90S0yu40j+JOaqH35Ir3kb4NgTMXk6Yqlidw06r3P2cqbZ0jBleOFf5eRfiu0ZquU5PJ/J7t9Pecx9S8mlitTtFPlvpVprNPB+XPSz2uwcwNW9i5OBUgR3PXsOjILLog8SiWYyk7bHaWnJtZ+JVEi9isy4EiwyrDY5kHRK2kB9Nnf6a9zz2krP7W+w9a3qXJkv8GP9D2+FN9Pj+2WP4r0hz7JL0i5q9bcc5HgBKP946u87z2lEjv2ioUAghaG/zwol3q+tKv0i6pPe0guGRCdpMlXa1Z1deOBJvxJXanTrIwi7dVc/bCsRGMRyYwD6hWhe1JjxgBjc/YbbBj8JJVdHrc2tGYFBU9qG2Kv3cAZMRrMXvKUkTK8JiMVzN0/DHEtdNv1PW4U3hlAqt5b62WahyzdHNVqHycwe+Ogz0BfTdohlxftv5qQYx0SEynXaIY+WltRnCnYrY1Kg1/DmsWYCGy++TO+6cEEwISPe/FM1peidsXVf5S3DCUQWE6aMK/6XDzukZoTjor/8JPkHc56Pk1Paty0yrP+YdL5R5m3IERzHoD"
    }
  }
}
```

可以看到内容已经得到了加密，接下来就需要创建 `SealedSecret` 了：
```
$ kubectl -n test apply -f test-seal-secret.yaml
```

接着查看`SealedSecret` 和 `secret`：

```
$ kubectl -n test get SealedSecret,secret
sealedsecret.bitnami.com/seal-test-secret   4s
secret/seal-test-secret      Opaque                                3      3s
```
在如下的 `Deployment` 中以环境变量的形式引用上述生成的 `secret`：

```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: devops
  name: devops
spec:
  replicas: 1
  selector:
    matchLabels:
      app: devops
  template:
    metadata:
      labels:
        app: devops
    spec:
      restartPolicy: Always
      containers:
      - name: devops
        image: dllhb/devopsday:v0.6
        imagePullPolicy: Always
        envFrom:
          - secretRef:
              name: test-secret
        ports:
        - containerPort: 9999
          name: http
          protocol: TCP
```

使用下面命令部署：
```
$ kubectl -n test apply -f test-deploy.yaml
deployment.apps/devops created
```

查看 pod 状态，并查看环境变量（`secret` 是以环境变量的形式注入 pod 内的）
```
$ kubectl -n test get pods
devops-b48df6659-gmjtr    1/1     Running   0          21s
$ kubectl -n test exec -it devops-b48df6659-gmjtr sh
env | grep -E "username|password|token"
username=asdf
token=1e4dgr5fgrh3rffgrhtonnhrhr
password=passw0rd
```
说明 `secret` 注入成功。最后就可以包含 `secret` 信息且经过加密的文件 test-seal-secret.yaml 推送至版本管理系统，比如 GitHub。

其他的 `secret` 类型，比如 tls、dockerconfigjson 等都可以用上面的方式进行使用。

## Helm Secrets

Helm Secrets 是 Helm 的一个插件，用来对于Helm Chart 中的敏感信息进行加密。

### 安装

前提条件：

* helm
* sops


## Kamus

Kamus 是一款开源的采用零信任的 secret 加解密方式为 Kubernetes 应用程序安全处理 secrets 的工具。Kamus 提供两种方式来对 Kubernetes secrets 进行加密，即

* **使用 `init container`**：将 secrets 加密后存储为 Kubernetes configmap，然后在应用程序的部署中添加一个 `init container`，通过 `init container` 将 `configmap` 中的加密数据解密至指定文件，应用程序再从此文件读取解密了的 `secret` 信息。
* **使用 `KamusSecret`**：创建一个 `KamusSecret` 对象（充分利用了 Kubernetes 的扩展机制），此对象包含了加密之后的数据，在此对象创建的过程中，会创建一个同名的 `secret` 对象，`secret` 中的数据是经过 base64 编码之后的数据，可以被 Kubernetes 其他对象引用。

下面会分别讲述这两种方法的具体使用。首先需要完成 `kamus` 的安装。

### 安装

`kamus` 的安装包括 `controller` 和 客户端工具 `kamus-cli`。

#### 安装 `controller`
```
$ helm repo add soluto https://charts.soluto.io
$ helm install kamus --namespace kamus soluto/kamus
```
#### 检查 pod 状态
```
$ kubectl -n kamus get pods
NAME                                READY   STATUS    RESTARTS   AGE
kamus-controller-55d959895d-hdklf   1/1     Running   0          9m30s
kamus-decryptor-5974b6ff47-5pkbr    1/1     Running   0          7m34s
kamus-decryptor-5974b6ff47-c4jt4    1/1     Running   0          7m34s
kamus-encryptor-f75dd457-fwp8r      1/1     Running   0          9m28s
kamus-encryptor-f75dd457-p9rnx      1/1     Running   0          9m29s
```

####  安装客户端
```
$ npm install -g @soluto-asurion/kamus-cli
```

检查安装是否成功
```
$ kamus-cli -V
0.3.0
```

### 使用

下面分别以`init container`和`KamusSecret`的方式来演示使用方式。

#### 以`init container`的方式

首先需要加密 `secret`。kamus 加密 secret 时需要和一个 `service account` 关联起来，此 `service account` 会在后续的应用部署中使用，所以先创建一个 `service account`（本文所有 demo 均在 test ns 下）：

```
$ kubectl -n test create sa xiaomage
```

接着使用客户端工具 `kamus-cli` 来加密 `secret`（username 的值为 xiaomge，password 的值为 passw0rd）：

```
$ kamus-cli encrypt --secret xiaomage --service-account xiaomage --namespace test --kamus-url http://localhost:9999 --allow-insecure-url
[info  kamus-cli]: Encryption started...
[info  kamus-cli]: service account: xiaomage
[info  kamus-cli]: namespace: test
[warn  kamus-cli]: Auth options were not provided, will try to encrypt without authentication to kamus
[info  kamus-cli]: Successfully encrypted data to xiaomage service account in test namespace
[info  kamus-cli]: Encrypted data:
CxujUBK4jgdjt+wP5mXyDA==:+CoXRDJVtW3EZ2FpterVTA==
```
返回的`CxujUBK4jgdjt+wP5mXyDA==:+CoXRDJVtW3EZ2FpterVTA==`就是 xiaomage 经过加密后的值，用同样的方法，可以将 passw0rd 进行加密：
```
$ kamus-cli encrypt --secret passw0rd --service-account xiaomage --namespace test --kamus-url http://localhost:9999 --allow-insecure-url
[info  kamus-cli]: Encryption started...
[info  kamus-cli]: service account: xiaomage
[info  kamus-cli]: namespace: test
[warn  kamus-cli]: Auth options were not provided, will try to encrypt without authentication to kamus
[info  kamus-cli]: Successfully encrypted data to xiaomage service account in test namespace
[info  kamus-cli]: Encrypted data:
ChmNEPM8Nj7Huh1YwO5xOA==:r9MHhEyTIEaQ4hw837lA9w==
```
然后，将加密后的值放在 `configmaap` 中：
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: kamus-encrypted-secrets-cm
  namespace: test
data:
  username: CxujUBK4jgdjt+wP5mXyDA==:+CoXRDJVtW3EZ2FpterVTA==
  password: ChmNEPM8Nj7Huh1YwO5xOA==:r9MHhEyTIEaQ4hw837lA9w==
```

接下来，将上述 `configmap` 以 `volume` 的方式挂在到 `pod` 中，随后使用 `init container` 来解密数据，且将数据存放在一个 `config.json` 文件中：

```
apiVersion: v1
kind: Pod
metadata:
  namespace: test
  name: kamus-pod
spec:
   serviceAccountName: xiaomage
   automountServiceAccountToken: true
   initContainers:
   - name: "kamus-init"
     image: "soluto/kamus-init-container:latest"
     imagePullPolicy: IfNotPresent
     env:
     - name: KAMUS_URL
       value: http://kamus-decryptor.kamus.svc.cluster.local/
     volumeMounts:
     - name: encrypted-secrets
       mountPath: /encrypted-secrets
     - name: decrypted-secrets
       mountPath: /decrypted-secrets
     args: ["-e","/encrypted-secrets","-d","/decrypted-secrets", "-n", "config.json"]
   containers:
   - name: kamus-test
     image: dllhb/devopsday:v0.6
     imagePullPolicy: IfNotPresent
     volumeMounts:
     - name: decrypted-secrets
       mountPath: /secrets
   volumes:
   - name: encrypted-secrets
     configMap:
       name: kamus-encrypted-secrets-cm
   - name: decrypted-secrets
     emptyDir:
       medium: Memory
```

> 需要注意的是，需要指定 `kamus` 的地址，即 `decryptor` 的地址，可根据自己的安装情况自行指定。

接下来，部署 `configmap` 和 `pod` 并查看：
```
$ kubectl -n test apply -f configmap.yaml
$ kubectl -n test apply -f kamus-deploy.yaml
$ kubectl -n test get pods,cm
NAME                          READY   STATUS    RESTARTS   AGE
pod/kamus-pods              1/1     Running   0          4h3m

NAME                                   DATA   AGE
configmap/kamus-encrypted-secrets-cm   2      30s
```

进入`pod` 查看解密后的数据：

```
$kubectl -n test exec -it kamus-deploy sh
$ cat /secrets/config.json
{
    "password":"passw0rd",
    "username":"username"
}
```

可以看到 `secet` 已经被解密到了`config.json`文件中，应用程序只需要读取此文件即可获得`secret`的相关数据。

#### 以`KamusSecret`的方式

kamus 对 Kubernetes 进行了扩展，有了自己支持的`KamusSecret` 对象，将上述加密后的数据存放在 `KamusSecret` 中：
```
apiVersion: "soluto.com/v1alpha2"
kind: KamusSecret
metadata:
  name: kamus-test
  namespace: test
type: Opaque
stringData:
  username: CxujUBK4jgdjt+wP5mXyDA==:+CoXRDJVtW3EZ2FpterVTA==
  password: ChmNEPM8Nj7Huh1YwO5xOA==:r9MHhEyTIEaQ4hw837lA9w==
serviceAccount: xiaomage
```
创建 `KamusSecret` 对象：
```
$ kubectl -n test apply -f kamus-secrets.yaml
```
查看生成的`KamusSecret`和`Secret`：
```
$ kubectl -n test get KamusSecret,secret
NAME                                AGE
kamussecret.soluto.com/kamus-test   60s

NAME                                TYPE                                  DATA   AGE
secret/kamus-test                   Opaque                                2      59s
```
可以看到`KamusSecret`生成了一个和自己同名的`secret`，接着查看`secret`的内容：
```
apiVersion: v1
data:
  password: cGFzc3cwcmQ=
  username: eGlhb21hZ2U=
```
解码后为：
```
password: passw0rd
username: xiaomage
```
最后，可以像正常方式在`pod`中引用此`secret`。
