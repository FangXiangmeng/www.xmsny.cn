---
layout: post
title: 服务注入nginx sidecat
subtitle: ""
description: "注入nginx sidecar容器到服务中"
date: 2019-06-10T11:25:13+08:00
author: "FangXiangMeng"
published: true
tags:
  - Kubernetes
categories: [ Docs ]
---


### 自动注入nginx sidecar半生容器

**开源代码：** https://github.com/morvencao/kube-mutating-webhook-tutorial \
**参考文章：** https://blog.hdls.me/15564491070483.html

###### 需求：
> 访问请求： nodeport--nginx-podip:port---localhost---tomcat 

想让pod内部的访问请求都走nginx再到服务。也就是访问服务得时候，要走pod内部nginx，然后再通过localhost转发到tomcat服务。pod中两个容器，1个nginx 1个tomcat，这样会有一个问题，就是每次创建服务的时候都要手动把nginx相关yaml写入到服务得yaml中。会很麻烦。这时候可以通过自动注入的方式。动态注入nginx pod到服务中。和istio中的sidecar做法一样。


### 实现方法

###### 编写 Sidecar 注入配置
现在我们来创建一个 Kubernetes ConfigMap，包含需要注入到目标 pod 中的容器和 volume 信息：
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: sidecar-injector-webhook-configmap
data:
  sidecarconfig.yaml: |
    containers:
      - name: sidecar-nginx
        image: nginx:1.12.2
        imagePullPolicy: IfNotPresent
        ports:
          - containerPort: 80
        volumeMounts:
          - name: nginx-conf
            mountPath: /etc/nginx
    volumes:
      - name: nginx-conf
        configMap:
          name: nginx-configmap
```

> 可以添加一些环境变量以及其他参数

###### 创建
```
$ git clone https://github.com/morvencao/kube-mutating-webhook-tutorial.git
$ kubectl create -f ./deployment/nginxconfigmap.yaml
configmap "nginx-configmap" created
$ kubectl create -f ./deployment/configmap.yaml
configmap "sidecar-injector-webhook-configmap" created
```
> nginxconfigmap中的是nginx的配置文件，可以进行更改

###### 创建包含秘钥对的 Secret
由于准入控制是一个高安全性操作，所以对外在的 webhook server 提供 TLS 是必须的。作为流程的一部分，我们需要创建由 Kubernetes CA 签名的 TLS 证书，以确保 webhook server 和 apiserver 之间通信的安全性。对于 CSR 创建和批准的完整步骤，请参考 [这里](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/) 。

简单起见，我们参考了 Istio 的脚本并创建了一个类似的名为 webhook-create-signed-cert.sh 的脚本，来自动生成证书及秘钥对并将其加入到 secret 中。
```
#!/bin/bash
while [[ $# -gt 0 ]]; do
    case ${1} in
        --service)
            service="$2"
            shift
            ;;
        --secret)
            secret="$2"
            shift
            ;;
        --namespace)
            namespace="$2"
            shift
            ;;
    esac
    shift
done

[ -z ${service} ] && service=sidecar-injector-webhook-svc
[ -z ${secret} ] && secret=sidecar-injector-webhook-certs
[ -z ${namespace} ] && namespace=default

csrName=${service}.${namespace}
tmpdir=$(mktemp -d)
echo "creating certs in tmpdir ${tmpdir} "

cat <<EOF >> ${tmpdir}/csr.conf
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names
[alt_names]
DNS.1 = ${service}
DNS.2 = ${service}.${namespace}
DNS.3 = ${service}.${namespace}.svc
EOF

openssl genrsa -out ${tmpdir}/server-key.pem 2048
openssl req -new -key ${tmpdir}/server-key.pem -subj "/CN=${service}.${namespace}.svc" -out ${tmpdir}/server.csr -config ${tmpdir}/csr.conf

# clean-up any previously created CSR for our service. Ignore errors if not present.
kubectl delete csr ${csrName} 2>/dev/null || true

# create  server cert/key CSR and  send to k8s API
cat <<EOF | kubectl create -f -
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: ${csrName}
spec:
  groups:
  - system:authenticated
  request: $(cat ${tmpdir}/server.csr | base64 | tr -d '\n')
  usages:
  - digital signature
  - key encipherment
  - server auth
EOF

# verify CSR has been created
while true; do
    kubectl get csr ${csrName}
    if [ "$?" -eq 0 ]; then
        break
    fi
done

# approve and fetch the signed certificate
kubectl certificate approve ${csrName}
# verify certificate has been signed
for x in $(seq 10); do
    serverCert=$(kubectl get csr ${csrName} -o jsonpath='{.status.certificate}')
    if [[ ${serverCert} != '' ]]; then
        break
    fi
    sleep 1
done
if [[ ${serverCert} == '' ]]; then
    echo "ERROR: After approving csr ${csrName}, the signed certificate did not appear on the resource. Giving up after 10 attempts." >&2
    exit 1
fi
echo ${serverCert} | openssl base64 -d -A -out ${tmpdir}/server-cert.pem


# create the secret with CA cert and server cert/key
kubectl create secret generic ${secret} \
        --from-file=key.pem=${tmpdir}/server-key.pem \
        --from-file=cert.pem=${tmpdir}/server-cert.pem \
        --dry-run -o yaml |
    kubectl -n ${namespace} apply -f -
```
运行脚本后，包含证书和秘钥对的 secret 就被创建出来了：
```
[root@mstnode kube-mutating-webhook-tutorial]# ./deployment/webhook-create-signed-cert.sh
creating certs in tmpdir /tmp/tmp.wXZywp0wAF
Generating RSA private key, 2048 bit long modulus
...........................................+++
..........+++
e is 65537 (0x10001)
certificatesigningrequest "sidecar-injector-webhook-svc.default" created
NAME                                   AGE       REQUESTOR                                           CONDITION
sidecar-injector-webhook-svc.default   0s        https://mycluster.icp:9443/oidc/endpoint/OP#admin   Pending
certificatesigningrequest "sidecar-injector-webhook-svc.default" approved
secret "sidecar-injector-webhook-certs" created
```

###### 创建 Sidecar 注入器的 Deployment 和 Service
deployment 带有一个 pod，其中运行的就是 sidecar-injector 容器。该容器以特殊参数运行：

- sidecarCfgFile 指的是 sidecar 注入器的配置文件，挂载自上面创建的 ConfigMap sidecar-injector-webhook-configmap。
- tlsCertFile 和 tlsKeyFile 是秘钥对，挂载自 Secret injector-webhook-certs。
- alsologtostderr、v=4 和 2>&1 是日志参数。
```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: sidecar-injector-webhook-deployment
  labels:
    app: sidecar-injector
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: sidecar-injector
    spec:
      containers:
        - name: sidecar-injector
          image: morvencao/sidecar-injector:v1
          imagePullPolicy: IfNotPresent
          args:
            - -sidecarCfgFile=/etc/webhook/config/sidecarconfig.yaml
            - -tlsCertFile=/etc/webhook/certs/cert.pem
            - -tlsKeyFile=/etc/webhook/certs/key.pem
            - -alsologtostderr
            - -v=4
            - 2>&1
          volumeMounts:
            - name: webhook-certs
              mountPath: /etc/webhook/certs
              readOnly: true
            - name: webhook-config
              mountPath: /etc/webhook/config
      volumes:
        - name: webhook-certs
          secret:
            secretName: sidecar-injector-webhook-certs
        - name: webhook-config
          configMap:
            name: sidecar-injector-webhook-configmap
```
Service 暴露带有 app=sidecar-injector label 的 pod，使之在集群中可访问。这个 Service 会被 MutatingWebhookConfiguration 中定义的 clientConfig 部分访问，默认的端口 spec.ports.port 需要设置为 443。
```
apiVersion: v1
kind: Service
metadata:
  name: sidecar-injector-webhook-svc
  labels:
    app: sidecar-injector
spec:
  ports:
  - port: 443
    targetPort: 443
  selector:
    app: sidecar-injector
```

然后将上述 Deployment 和 Service 部署到集群中，并且验证 sidecar 注入器的 webhook server 是否 running：
```
[root@mstnode kube-mutating-webhook-tutorial]# kubectl create -f ./deployment/deployment.yaml
deployment "sidecar-injector-webhook-deployment" created
[root@mstnode kube-mutating-webhook-tutorial]# kubectl create -f ./deployment/service.yaml
service "sidecar-injector-webhook-svc" created
[root@mstnode kube-mutating-webhook-tutorial]# kubectl get deployment
NAME                                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
sidecar-injector-webhook-deployment   1         1         1            1           2m
[root@mstnode kube-mutating-webhook-tutorial]# kubectl get pod
NAME                                                  READY     STATUS    RESTARTS   AGE
sidecar-injector-webhook-deployment-bbb689d69-fdbgj   1/1       Running   0          3m
```

动态配置 webhook 准入控制器
MutatingWebhookConfiguration 中具体说明了哪个 webhook admission server 是被使用的并且哪些资源受准入服务器的控制。建议你在创建 MutatingWebhookConfiguration 之前先部署 webhook admission server，并确保其正常工作。否则，请求会被无条件接收或根据失败规则被拒。

现在，我们根据下面的内容创建 MutatingWebhookConfiguration：
```
apiVersion: admissionregistration.k8s.io/v1beta1
kind: MutatingWebhookConfiguration
metadata:
  name: sidecar-injector-webhook-cfg
  labels:
    app: sidecar-injector
webhooks:
  - name: sidecar-injector.morven.me
    clientConfig:
      service:
        name: sidecar-injector-webhook-svc
        namespace: default
        path: "/mutate"
      caBundle: ${CA_BUNDLE}
    rules:
      - operations: [ "CREATE" ]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    namespaceSelector:
      matchLabels:
        sidecar-injector: enabled
```

第 8 行：name - webhook 的名字，必须指定。多个 webhook 会以提供的顺序排序；
第 9 行：clientConfig - 描述了如何连接到 webhook admission server 以及 TLS 证书；
第 15 行：rules - 描述了 webhook server 处理的资源和操作。在我们的例子中，只拦截创建 pods 的请求；
第 20 行：namespaceSelector - namespaceSelector 根据资源对象是否匹配 selector 决定了是否针对该资源向 webhook server 发送准入请求。

在部署 MutatingWebhookConfiguration 前，我们需要将 ${CA_BUNDLE} 替换成 apiserver 的默认 caBundle。我们写个脚本来自动匹配：
```
#!/bin/bash
set -o errexit
set -o nounset
set -o pipefail

ROOT=$(cd $(dirname $0)/../../; pwd)

export CA_BUNDLE=$(kubectl get configmap -n kube-system extension-apiserver-authentication -o=jsonpath='{.data.client-ca-file}' | base64 | tr -d '\n')

if command -v envsubst >/dev/null 2>&1; then
    envsubst
else
    sed -e "s|\${CA_BUNDLE}|${CA_BUNDLE}|g"
fi
```
然后执行：
```
[root@mstnode kube-mutating-webhook-tutorial]# cat ./deployment/mutatingwebhook.yaml |\
>   ./deployment/webhook-patch-ca-bundle.sh >\
>   ./deployment/mutatingwebhook-ca-bundle.yaml
```

我们看不到任何日志描述 webhook server 接收到准入请求，似乎该请求并没有发送到 webhook server 一样。所以有一种可能性是这是被 MutatingWebhookConfiguration 中的配置触发的。再确认一下 MutatingWebhookConfiguration 我们会发现下面的内容：
```
namespaceSelector:
      matchLabels:
        sidecar-injector: enabled
```
通过 namespaceSelector 控制 sidecar 注入器
我们在 MutatingWebhookConfiguration 中配置了 namespaceSelector，也就意味着只有在满足条件的 namespace 下的资源能够被发送到  webhook server。于是我们将 default 这个 namespace 打上标签 sidecar-injector=enabled：
```
[root@mstnode kube-mutating-webhook-tutorial]# kubectl label namespace default sidecar-injector=enabled
namespace "default" labeled
[root@mstnode kube-mutating-webhook-tutorial]# kubectl get namespace -L sidecar-injector
NAME          STATUS    AGE       sidecar-injector
default       Active    1d        enabled
kube-public   Active    1d
kube-system   Active    1d
```
现在我们配置的 MutatingWebhookConfiguration 会在 pod 创建的时候就注入 sidecar 容器。将运行中的 pod 删除并确认是否创建了新的带 sidecar 容器的 pod：
```
[root@mstnode kube-mutating-webhook-tutorial]# kubectl delete pod sleep-6d79d8dc54-r66vz
pod "sleep-6d79d8dc54-r66vz" deleted
[root@mstnode kube-mutating-webhook-tutorial]# kubectl get pods
NAME                                                  READY     STATUS              RESTARTS   AGE
sidecar-injector-webhook-deployment-bbb689d69-fdbgj   1/1       Running             0          29m
sleep-6d79d8dc54-b8ztx                                0/2       ContainerCreating   0          3s
sleep-6d79d8dc54-r66vz                                1/1       Terminating         0          11m
[root@mstnode kube-mutating-webhook-tutorial]# kubectl get pod sleep-6d79d8dc54-b8ztx -o yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubernetes.io/psp: default
    sidecar-injector-webhook.morven.me/inject: "true"
    sidecar-injector-webhook.morven.me/status: injected
  labels:
    app: sleep
    pod-template-hash: "2835848710"
  name: sleep-6d79d8dc54-b8ztx
  namespace: default
spec:
  containers:
  - command:
    - /bin/sleep
    - infinity
    image: tutum/curl
    imagePullPolicy: IfNotPresent
    name: sleep
    resources: {}
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-d7t2r
      readOnly: true
  - image: nginx:1.12.2
    imagePullPolicy: IfNotPresent
    name: sidecar-nginx
    ports:
    - containerPort: 80
      protocol: TCP
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /etc/nginx
      name: nginx-conf
  volumes:
  - name: default-token-d7t2r
    secret:
      defaultMode: 420
      secretName: default-token-d7t2r
  - configMap:
      defaultMode: 420
      name: nginx-configmap
    name: nginx-conf
...
```
可以看到，sidecar 容器和 volume 被成功注入到应用中。至此，我们成功创建了可运行的带 MutatingAdmissionWebhook 的 sidecar 注入器。通过 namespaceSelector，我们可以轻易的控制在特定的 namespace 中的 pods 是否需要被注入 sidecar 容器。

但这里有个问题，根据以上的配置，在 default 这个 namespace 下的所有 pods 都会被注入 sidecar 容器，无一例外。

###### 通过注解控制 sidecar 注入器
多亏了 MutatingAdmissionWebhook 的灵活性，我们可以轻易的自定义变更逻辑来筛选带有特定注解的资源。还记得上面提到的注解 ```sidecar-injector-webhook.morven.me/inject: "true"``` 吗？在 sidecar 注入器中这可以当成另一种控制方式。在 webhook server 中我写了一段逻辑来跳过那行不带这个注解的 pod。

我们来尝试一下。在这种情况下，我们创建另一个 sleep 应用，其 ```podTemplateSpec``` 中不带注解 ```sidecar-injector-webhook.morven.me/inject: "true"：```
```
[root@mstnode kube-mutating-webhook-tutorial]# kubectl delete deployment sleep
deployment "sleep" deleted
[root@mstnode kube-mutating-webhook-tutorial]# cat <<EOF | kubectl create -f -
apiVersion: extensions/v1beta1
> kind: Deployment
> metadata:
>   name: sleep
> spec:
>   replicas: 1
>   template:
>     metadata:
>       labels:
>         app: sleep
>     spec:
>       containers:
>       - name: sleep
>         image: tutum/curl
>         command: ["/bin/sleep","infinity"]
>         imagePullPolicy: IfNotPresent
> EOF
deployment "sleep" created
```
然后确认 sidecar 注入器是否跳过了这个 pod：
```
[root@mstnode kube-mutating-webhook-tutorial]# kubectl get deployment
NAME                                  DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
sidecar-injector-webhook-deployment   1         1         1            1           45m
sleep                                 1         1         1            1           17s
[root@mstnode kube-mutating-webhook-tutorial]# kubectl get pod
NAME                                                  READY     STATUS        RESTARTS   AGE
sidecar-injector-webhook-deployment-bbb689d69-fdbgj   1/1       Running       0          45m
sleep-776b7bcdcd-4bz58       
```
结果显示，这个 sleep 应用只包含一个容器，没有额外的容器和 volume 注入。然后我们将这个 deployment 增加注解，并确认其重建后是否被注入 sidecar：
```
[root@mstnode kube-mutating-webhook-tutorial]# kubectl patch deployment sleep -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar-injector-webhook.morven.me/inject": "true"}}}}}'
deployment "sleep" patched
[root@mstnode kube-mutating-webhook-tutorial]# kubectl delete pod sleep-776b7bcdcd-4bz58
pod "sleep-776b7bcdcd-4bz58" deleted
[root@mstnode kube-mutating-webhook-tutorial]# kubectl get pods
NAME                                                  READY     STATUS              RESTARTS   AGE
sidecar-injector-webhook-deployment-bbb689d69-fdbgj   1/1       Running             0          49m
sleep-3e42ff9e6c-6f87b                                0/2       ContainerCreating   0          18s
sleep-776b7bcdcd-4bz58         
```
与预期一致，pod 被注入了额外的 sidecar 容器。至此，我们就获得了一个可工作的 sidecar 注入器，可由 namespaceSelector 或更细粒度的由注解控制。