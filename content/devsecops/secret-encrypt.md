---
title: "Kubernetes secrets 加密的几种方式"
description: "处理好 Kubernetes secretes 是实现 GitOps 的关键"
author: 马景贺（小马哥）
categories: ["Security"]
tags: ["cloud security","security"]
date: 2021-02-05T13:05:42+08:00
type: "post"
---

## 前言

`Kubernetes` 已经毫无争议的成为了云原生时代的事实标准，在 `Kubernetes` 上部署应用程序也变得简单起来（无论是采用 `kustomize` 还是 `helm`），虽然对于敏感信息（比如用户名、密码、token 和证书等）的处理，`Kubernetes` 自己提供了 `secret` 这种方式，但是其是一种编码方式，而非加密方式，如果需要用版本控制系统（比如 git）来对所有的文件、内容等进行版本控制时，这种用编码来处理敏感信息的方式就显得很不安全了（即使是采用私有库），这一点在实现 `GitOps` 时，是一个痛点。基于此，本文就介绍三种可以加密 `Kubernetes secret` 的几种方式：`Sealed Secrets`、`Helm Secrets` 和 `Kamus`。


## Sealed Secrets

`Sealed Secrets` 是加密 `kubernetes secret` 的一种方式，充分利用 `kubernte`s 的高扩展性，通过 `CRD` 来创建一个 `SealedSecret` 对象，然后将 `secret` 加密变成一个 `SealedSecret` 对象，而 `SealedSecret` 只能够被运行于目标集群上的 controller 解密。其他人员和方式都无法正确解密原始数据。可以将加密后的文件直接推送至版本控制系统，而不用担心敏感信息被泄漏。

### 安装

`Sealed Secrets` 有两部分组成：

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

关于 Helm 的介绍与使用可以查看[这篇公众号文章](https://mp.weixin.qq.com/s?__biz=Mzg3NjIzODc5NA==&mid=2247483702&idx=1&sn=aba5c37d0570dfaf74053ff55ab8155a&chksm=cf340393f8438a8558b627d466951b252990656d60826c91f8c10c8aa44e4cd329ebdc2f69d8&mpshare=1&scene=1&srcid=0206qXfqgb0wJITJVqxgqJ7Z&sharer_sharetime=1612610316961&sharer_shareid=69a671b032908bc53da173d06860fd16&exportkey=ATUCeld2T%2Bto3xSroQqGeNA%3D&pass_ticket=iUA3ldsRFzyThNhlk2ZEzJC9YRhNhoY8aCOqi5pTahkuTRrc5uTQqf4n1zganuN1&wx_header=0#rd)。下面我们简单介绍一下 sops。

#### sops

`sops`是一个加密文件的编辑器，支持 YAML、JSON、ENV、INI 和二进制格式，并使用 AWS KMS、GCP KMS、Azure Key Vault 和PGP 进行加密。本文将使用 PGP 来进行加密。

PGP（Pretty Good Privacy）是一种常用的加密方式。在 1990s（1991 年）由 Phil Zimmermann 所开发，现在归属于 Symantec 公司，它是商业软件，需要付费才能使用。而 GPG（GNU Privacy Guard）是一种基于 Open PGP 标准的加密方式。它是开源且免费的。所以本文的演示将使用 GPG 的方式。

由于`sops`采用非对称加密，所以需要先生成一对`key`。使用`gpg --full-generate-key`并输入必要的参数即可生成`key`，如下：
```
$ gpg --full-generate-key
gpg (GnuPG) 2.2.12; Copyright (C) 2018 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Please select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
Your selection? 1
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (3072) 4096
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 1y
Key expires at Sat Jan  8 12:12:10 2022 UTC
Is this correct? (y/N) y

GnuPG needs to construct a user ID to identify your key.

Real name: xiaomage
Email address: devops@xiaomage.com
Comment: gpg key generation
You selected this USER-ID:
    "xiaomage (gpg key generation) <devops@xiaomage.com>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.
gpg: key 8BA2C5716B5C007F marked as ultimately trusted

gpg: revocation certificate stored as '/root/.gnupg/openpgp-revocs.d/BCEB5797691E6C95E33A465D8BA2C5716B5C007F.rev'
public and secret key created and signed.

pub   rsa4096 2021-01-08 [SC] [expires: 2022-01-08]
      BCEB5797691E6C95E33A465D8BA2C5716B5C007F
uid                      xiaomage (gpg key generation) <devops@xiaomage.com>
sub   rsa4096 2021-01-08 [E] [expires: 2022-01-08]
```
可以查看生成的`private key`和`public key`：

* private key 查看
```
gpg -K(gpg --list-secret-keys)

/root/.gnupg/pubring.kbx
------------------------
sec   rsa4096 2021-01-08 [SC] [expires: 2022-01-08]
      BCEB5797691E6C95E33A465D8BA2C5716B5C007F
uid           [ultimate] xiaomage (gpg key generation) <devops@xiaomage.com>
ssb   rsa4096 2021-01-08 [E] [expires: 2022-01-08]
```
* public key 查看

```
gpg -k(gpg --list-keys)
gpg: checking the trustdb
gpg: marginals needed: 3  completes needed: 1  trust model: pgp
gpg: depth: 0  valid:   2  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 2u
gpg: next trustdb check due at 2022-01-08
/root/.gnupg/pubring.kbx
------------------------
pub   rsa4096 2021-01-08 [SC] [expires: 2022-01-08]
      BCEB5797691E6C95E33A465D8BA2C5716B5C007F
uid           [ultimate] xiaomage (gpg key generation) <devops@xiaomage.com>
sub   rsa4096 2021-01-08 [E] [expires: 2022-01-08]
```

在用 sops 来加密数据之前，先创建一个`.sops.yaml`文件，写一个加密的规则，比如只加密`key`是`username`和`password`的敏感信息，而且需要指定 gpg 的 fingerprint。
```
cat << EOF > .sops.yaml
creation_rules:
  - encrypted_regex: '^(username|password)$'
    pgp: 'B1C77B2CCF5575FAF0DA6B882CA51446C98C9D85'
EOF
```

接着，将下面的敏感信息写入一个yaml文件
```
cat << EOF > secrets.yaml
username: xiaomage
password: passw0rd
EOF
```
现在就可以用 sops 来加密数据了：
```
$ sops -e secrets.yaml
username: ENC[AES256_GCM,data:s6pInMY3eGM=,iv:5Q7JsntVoKjseD3ApWcgmYeedmGXj2A1/PyGCNFHGdE=,tag:vInq3NBLxvVWXsoVUD46Rw==,type:str]
password: ENC[AES256_GCM,data:Ua7de2w6Jgw=,iv:qYIjTW1D0dh20NA8FGu4XEGI16kvYGAWIk4iu3r/Gdg=,tag:b33tpsP1vCgqlpyCEDP88Q==,type:str]
sops:
    kms: []
    gcp_kms: []
    azure_kv: []
    hc_vault: []
    lastmodified: '2021-02-06T12:08:57Z'
    mac: ENC[AES256_GCM,data:QHHDRSO2PyJt0/OA67ex0R39gEjWEnwg0MSnBac8QtLNh3ncY+9D8IZw/WqVnbcaiPta2Pem96yJZTZP4pum9ZX446iRKldsAXNqS4+tmlfowpMWI+1DgOa1QCkhSDH9U/2URA1dzyn3cZLPFzb5Ai6YUEQ93sRjlPI+kHXl16c=,iv:jhFM/uJSeChikUv777qgYVDFCHQhQeXlUSjiHx5X8Ow=,tag:6QTo5CsXQoqr0fK1B947ug==,type:str]
    pgp:
    -   created_at: '2021-02-06T12:08:51Z'
        enc: |
            -----BEGIN PGP MESSAGE-----

            hQIMA/AYjF0OZ4PLAQ/+LRc5vgpRhOez8q9up8t+3OVM5QdnMwSYiuwLvjfInqqk
            K19jUfUhwXDGGtSMlTotYlTWqWCiSm7sYeqFB0/Lx9lCZY5BhCrVnK7u7m8azpWU
            osCQNmJehflqqnBmn82nblOGnDjM/FYkcnz4+NHUPNyYV5tWjzw9s8i/WhDeuNrf
            IPnGKRCGJunWlHDP3yWMo7bnCNU/TmuRiSpf7lQLsp/U71M5t1X8RajatO7DPecq
            caq3VZ+Ynx0Qcgyt+aHugZw5Sw9oFOT4WVqwLlC/NKvrjtY8pCQ1HtY5/agLHrDw
            Hn2Phz1aQ+l4EAarphXCiYAFw/LHD2tisbQoApXe5tud9CjiyMu/14qhQalLgQxA
            yGcMmhEH7Ke4bubaA0ZPo8hBXAkxfdeicSzB/e1IkUP4LtlQiwPldDcDShB6MROH
            sK3RpELhSaNdfQZxqDVN0CgjRS0/AjboWejjrLQHD1hVcUDAU2WTyfvIaSxKpHIx
            ONo5sTvzYOjU/BRTLn0EujRP414xadOtt+4gEQDrGacYAokuiK2ev0dinHo32EWY
            j/vsb0o3whNRpBEGMZTUrl9HSkt58FQZsmu5JnL3ZYKiujHFoQS/aOcxD0slUxhC
            PoCnce6PgmB78RHOLHaXkTrORc+6oMpCGN8/K1hjXE+eH/kk4jv8yVLwmbg9XjLS
            XgFTcQYs6nVTSoWVea62kRN4qlC/XTJ6D91HXRX5UyB3qrZ3k+w9TOlM9quYYI/B
            E0FqbFVSKT3ekPQqF91a7tV01FIxpfr4Mvzy2+8xsXiAQtDm52PSlk9eovkAMqU=
            =nafU
            -----END PGP MESSAGE-----
        fp: B1C77B2CCF5575FAF0DA6B882CA51446C98C9D85
    encrypted_regex: ^(username|password)$
    version: 3.6.1
```

可以看到`username`和`password`的值都被加密了。

#### 安装 Helm Secrets plugin

执行一下命令安装`helm-secrets` plugin
```
$ helm plugin install https://github.com/zendesk/helm-secrets 
```

### Helm Secrets 的使用

首先，创建一个演示用的`helm chart`，并在根目录下创建一个名为`helm_vars`的目录，用来存放用来加密的`secret`，如下所示：
```
$ tree
.
└── devsecops
    ├── Chart.yaml
    ├── charts
    ├── helm_vars
    │   ├── secrets.yaml
    │   ├── tls.crt
    │   └── tls.key
    ├── templates
    │   ├── NOTES.txt
    │   ├── _helpers.tpl
    │   ├── deployment.yaml
    │   ├── hpa.yaml
    │   ├── ingress.yaml
    │   ├── secrets.yaml
    │   ├── service.yaml
    │   ├── serviceaccount.yaml
    │   └── tests
    │       └── test-connection.yaml
    └── values.yaml
```

将需要加密的信息写入`secrets.yaml`文件里面，比如
```
cat << EOF > secrets.yaml
secret_data:
  username: xiaomage
  password: passw0rd
EOF
```

使用`helm secrets`来加密上述文件
```
$ helm-3 secrets enc secrets.test.yaml
Encrypting secrets.test.yaml
Encrypted secrets.test.yaml
```
查看加密后的文件内容
```
secret_data:
    username: ENC[AES256_GCM,data:O/1pyNsL3Gc=,iv:HZ0MrGWaBxM37cIkp/JdsA5gRzw6aJFfBR19rno3h5I=,tag:2SiMs46lonnwECc8RHfT/Q==,type:str]
    password: ENC[AES256_GCM,data:l15XlhZ4CsM=,iv:TMbV6+Rh2wGpMlHi7zJsHWM6IxMK2hBuMKsD82p8LiY=,tag:N4Kbftl//B1U2R9Khsduzg==,type:str]
sops:
    kms: []
    gcp_kms: []
    azure_kv: []
    hc_vault: []
    lastmodified: '2021-02-07T06:19:15Z'
    mac: ENC[AES256_GCM,data:dSXjEbKyBXVtqqSqshGXKUwDJcMVZrDf2GxFj0Oor3FDnNeS+bTY4Yubv1J0XlzU6yxO0Y87NzVN84unkF/Ph95JJV2opk6a0VTtaxKYOFUVneyY5WQ2glHEntX+aEq1lJkW1Sd34i/tvWeSABemIX4M2xcIOdIaCHgzk//vi9w=,iv:febius/ashzpdfKStJnQYVG/3FrVaYw102q87P9+egQ=,tag:/MUXrxhhOk6F8MS5wi7cLQ==,type:str]
    pgp:
    -   created_at: '2021-02-07T06:19:08Z'
        enc: |
            -----BEGIN PGP MESSAGE-----

            hQIMA7Oc9Dk1ccccARAAk7l23omTBRThnP7YC5AHdqzEO8Lapxc8ycWg5tsbM8eE
            JaRFn4u3/+dQdpL6xlHv1wu0kmrZUgG8P41WmNDIKb2GtAlHQk+bjjV2IU0lCEj7
            9UZXuAyhxHtVjHMBnzjppFh+6L0nH2K5AGaJWATwhO9M6CqmdCFnWJx7vAPfVQZF
            Li9zqHK/YsbwgEWKs0bVvJ1btB7u4J5olKagYaZhaFaLzwjbtXmEqDUpfmPkooNr
            7kPSVe8IMv/+MUaJY6uYNTBGWGrije4bY4A+hA/dUj4yN0gqqd796oc9GuN1MJSO
            cAAoiTW2Vrw3OdyP7PIJVuxlS9gXnxtBOjo+p/Ij91ELq+DnC+6bGS9UIeF+Y1RD
            h4siwx7I7hzk9tp+tXmsfdJit+usK6raPzYkcBgZVF8woKZsp2/qxloYyIFJ0sbK
            MO67+dcAg+AX0M0/u33t1BAMTt/LJ1V2ZQUl+yzjRSKfZ2bCmd/skkE3VZx2ls44
            LMngWZG7EzE39Onw9PB3ukXD7W+X+BThc2AJzVotrpDWbSI2/anoM9TMJjYfBjyU
            xBuTuoviT5ENdm14bGomww9G+Ean3dyC2vWoHhY2KfuPlSxZ6mDIDm5zAPkZZl5A
            QHjtaPT5qymPCpqy2X3yvK76zyJhfWYFIHguOy3JlDxiONC9DH1M6OVWoC69pPzS
            XgEtII9fTeLXFU5Jy9gJa5nNKEQY87OkSXl3TFAiQ9OmgDbuUHZuvQzlecsKwR2s
            mS7P7Z3Bb+eRakQ41Gzw4B7wmOrm2w0t4guVJDNIP/gQB0XBO1XZj4RsbMKn070=
            =yQ2R
            -----END PGP MESSAGE-----
        fp: B1C77B2CCF5575FAF0DA6B882CA51446C98C9D85
    encrypted_regex: ^(data|username|password|.dockerconfigjson|token|token1|key|crt)$
    version: 3.6.1
```

需要注意的是，此时只是加密了需要加密的内容，但是这些内容改怎么用呢？其实也比较简单，就是：正常用。举例来说，在`helm chart`中，正常用`secret`的方式如下
```
apiVersion: v1
kind: Secret
metadata:
  name: test
  labels:
    app: devsecops
type: Opaque
data:
  {{- range $key, $value := .Values.data}}
  {{ $key }} : {{ $value | b64enc | quote}}
  {{- end}}
```
而上面循环中引用的值`.Values.data`就是来自于上面`helm_vars`目录下的加密文件？怎么做到的呢？简单点说，就是执行`helm`的相关命令时，会先将`helm_vars`目录下的加密内容解密，并且“放在”`values.yaml`文件中，接下来的就和正常的`helm chart`使用是一样的了。在`chart`中的`deployment.yaml`文件中引用`secret`
```
......
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          envFrom:
            - secretRef:
                name: test
......
```

接下来可以使用下面命令将上面生成的`chart`进行安装
```
helm-3 secrets install test . --namespace test -f helm_vars/secrets.yaml -f values.yaml
```
接着查看生成的`pod`、`secret`
```
$ kubectl -n test get pods,secret
pod/test-devsecops-7876ffc8b7-967xr   1/1     Running   0          6s
secret/test                         Opaque                                2      8s
```
由于上面的`secret`是以环境变量的形式注入到`pod`里面的，可以查看进行验证
```
$ kubectl -n test exec -it test-devsecops-7876ffc8b7-967xr sh
$ env | grep -E 'username|password'
username=xiaomage
password=passw0rd
```
可以看到`secret`解密成功，并成功注入`pod`。

最后就可以将加密后的文件上传至源码管理系统了（比如`git push`至`GitHub`)。

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





## 参考
1. https://github.com/Soluto/kamus
2. https://kamus.soluto.io/
3. https://blog.solutotlv.com/can-kubernetes-keep-a-secret/
4. https://en.sokube.ch/post/lightweight-kubernetes-gitops-secrets
5. https://github.com/mozilla/sops#showing-diffs-in-cleartext-in-git
