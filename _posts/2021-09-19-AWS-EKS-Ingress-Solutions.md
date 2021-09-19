简介：AWS EKS有两种主流的Ingress实现方式. 1) 基于ALB的Ingress 2) 社区的nginx ingress 加 NLB。本文主要介绍如何落地

# 基础EKS环境

## 安装

本文不详细说明如何启动EKS，简单使用 `eksctl` 通过下面配置文件生成。具体eksctl 怎么使用，请参考 [官网](https://eksctl.io)

在配置好 `aws-cli` 的命令行环境执行:

`eksctl create -f esk.yaml`

esk.yaml
```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: ekstest
  region: ap-northeast-1

managedNodeGroups:
  - name: ekstest
    instanceTypes: ["m5.xlarge", "c5.xlarge", "r5.xlarge", "m5a.xlarge", "c5a.xlarge", "r5a.xlarge"]
    desiredCapacity: 1
    volumeSize: 50
    privateNetworking: true
```

## whomai 服务

[whoami](https://hub.docker.com/r/containous/whoami) 提供了方便简单的功能验证调试功能，方便我们通过浏览器快速查看访问信息。

```bash
kubectl apply -f whoami.yaml
```

whoami.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: containous/whoami
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
  labels:
    svc: nginx-svc
  annotations:
    alb.ingress.kubernetes.io/healthcheck-path: /health
spec:
  ports:
  - port: 8080
    targetPort: 80 
    protocol: TCP
  selector:
    app: nginx
  type: ClusterIP
```

# ALB Ingress

## 安装

参考 [官方安装文档](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/deploy/installation/#iam-permissions)

### AWS IAM role 及权限准备

```bash
eksctl utils associate-iam-oidc-provider \
    --region <region-code> \
    --cluster <your-cluster-name> \
    --approve

curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.1/docs/install/iam_policy.json

aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam-policy.json

# 记下上条命令返回的 policy ARN， 然后更新下面命令对应的 policy ARN

eksctl create iamserviceaccount \
--cluster=<cluster-name> \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
--override-existing-serviceaccounts \
--approve

```

### 安装 AWS Load Balancer Controller

```bash
helm repo add eks https://aws.github.io/eks-charts

kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"

helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=<cluster-name> --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
```

### 创建ingress

```bash
kubectl apply -f alb-ingress-example.yaml
```

alb-ingress-example.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: "alb-ingress"
  annotations:
    kubernetes.io/ingress.class: alb
  labels:
    ingress: alb-nginx
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: "nginx-svc"
                port:
                  number: 8080

```

## 测试

获取`ingress-nginx` DNS 地址：

```bash
kubectl get ing alb-ingress | awk '/alb-ingress/ {print $4}'
```

浏览器访问上面地址，验证信息是否正确。

## 注意事项

- 一个ALB 对应一个 ingress
- 一个Target Group对应一个backend services
- ALB 不支持 `url rewrite`, `cors`
- ALB controller 暂时不能垮`namespaces`引用`services`

# Nginx Ingress

## 安装

`Nginx Ingress` 在AWS上如果要跟ELB交互，依赖 [`AWS Load Balancer Controller`](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/guide/ingress/annotations/)。
前面已经安装了该`controller`，这里不再重复。

### 安装Nginx Ingress 
[使用 helm 安装Nginx Ingress](https://kubernetes.github.io/ingress-nginx/deploy/#using-helm)

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx
```

### 修改Ningx Ingress，使用NLB

```bash
kubectl edit svc ingress-nginx-controller 
```

添加如下 annotations
```yaml
service.beta.kubernetes.io/aws-load-balancer-backend-protocol: tcp
service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
service.beta.kubernetes.io/aws-load-balancer-type: nlb
service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: '*'
service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
service.beta.kubernetes.io/aws-load-balancer-type: external
```

### 启用Nginx Proxy 协议

[Proxy Protocol](https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/configmap/#use-proxy-protocol)是Load Balancer在TCP层用来保留源IP的方式。Nginx可以通过 ConfgiMap 修改nignx 配置:

```bash
kubectl apply -f ingress-nginx-cm.yaml
```

ingress-nginx-cm.yaml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  labels:
    helm.sh/chart: ingress-nginx-4.0.1
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/version: 1.0.0
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/component: controller
  name: ingress-nginx-controller
data:
  use-proxy-protocol: "true"
```

### 创建ingress

```bash
kubectl apply -f nginx-ingress-example.yaml
```

nginx-ingress-example.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: "ngx-ingress"
  annotations:
    kubernetes.io/ingress.class: nginx
  labels:
    ingress: ingress-nginx
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: "nginx-svc"
                port:
                  number: 8080

```

## 测试

获取`ingress-nginx` DNS 地址：

```bash
kubectl get svc ingress-nginx-controller | awk '/ingress-nginx-controller/ {print $4}'
```

浏览器访问上面地址，验证信息是否正确。

## 注意事项

- 每次`kubectl edit svc ingress-nginx-controller` 后，会生成新的ELB，并且老的不会删除。 这个要特别注意
- NLB的安全组由目标IP决定。因为EKS 默认的安全组没有开放 `0.0.0.0`，需要自行添加

# 总结

`AWS ALB ingress` 与 `Nginx ingress` 都是很优秀、成熟、免费的ingress方案，可以根据实践情况进行选择。由于ALB ingress流量可以直达pod，可以提供更优秀的性能，建议默认使用。如果有很复杂的场景或者历史原因，可以考虑 `nginx ingress`

# References
- [eksctl.io](https://eksctl.io/)
- [whoami docker image](https://hub.docker.com/r/containous/whoami)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/)
- [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)

