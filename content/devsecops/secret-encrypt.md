## Sealed Secrets

Sealed Secrets 是加密 kubernetes secret 的一种方式，充分利用 kuberntes 的高扩展性，通过 CRD 来创建一个 `SealedSecret` 对象，然后将 secret 加密变成一个 `SealedSecret` 对象，而 `SealedSecret` 只能够被运行于目标集群上的 controller 解密。其他人员和方式都无法正确解密原始数据。可以将加密后的文件直接推送至版本控制系统，而不用担心敏感信息被泄漏。

### 安装

Sealed Secrets 有两部分组成：

* 安装与集群侧的 controller/operator
* 客户端工具：`kubeseal`

所以安装也分两部分走，安装`controller`和`kubeseal`。安装的方法在[这儿](https://github.com/bitnami-labs/sealed-secrets/releases)可以找到，可以先安装`controller`，执行如下命令即可：

```
$ $ kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.14.1/controller.yaml
```

随后查看`kube-system` ns 下面的 controller pod
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

### 使用方式

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

可以看到内容已经得到了加密，接下来就需要创建 SealedSecret 了：
```
$ kubectl -n test apply -f test-seal-secret.yaml
```

接着查看 SealedSecret 和 secret：

```
$ kubectl -n test get SealedSecret,secret
sealedsecret.bitnami.com/seal-test-secret   4s
secret/seal-test-secret      Opaque                                3      3s
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

查看 pod 状态，并查看环境变量（secret 是以环境变量的形式注入 pod 内的）
```
$ kubectl -n test get pods
devops-b48df6659-gmjtr    1/1     Running   0          21s
$ kubectl -n test exec -it devops-b48df6659-gmjtr sh
env | grep -E "username|password|token"
username=asdf
token=1e4dgr5fgrh3rffgrhtonnhrhr
password=passw0rd
```
说明 secret 注入成功。最后就可以包含 secret 信息且经过加密的文件 test-seal-secret.yaml 推送至版本管理系统，比如 GitHub。

其他的 secret 类型，比如 tls、dockerconfigjson 等都可以用上面的方式进行使用。

## Helm Secrets

Helm Secrets 是 Helm 的一个插件，用来对于Helm Chart 中的敏感信息进行加密。

### 安装

前提条件：

* helm
* sops


## Kamus

### 安装

```
$ helm repo add soluto https://charts.soluto.io
$ helm install kamus --namespace kamus soluto/kamus
```
### 检查 pod 状态
```
$ kubectl -n kamus get pods
NAME                                READY   STATUS    RESTARTS   AGE
kamus-controller-55d959895d-hdklf   1/1     Running   0          9m30s
kamus-decryptor-5974b6ff47-5pkbr    1/1     Running   0          7m34s
kamus-decryptor-5974b6ff47-c4jt4    1/1     Running   0          7m34s
kamus-encryptor-f75dd457-fwp8r      1/1     Running   0          9m28s
kamus-encryptor-f75dd457-p9rnx      1/1     Running   0          9m29s
```

### 安装客户端
```
$ npm install -g @soluto-asurion/kamus-cli
```

检查安装是否成功
```
$ kamus-cli -V
0.3.0
```

