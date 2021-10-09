简介：Helm 是一个Kubernetes包管理工具。最新版本支持OCI协议。AWS ECR最新版本也支持OCI。本文快速介绍了如何结合ECR使用Helm。

前置知识：Helm, ECR, Kubernetes, AWS CLI

# 教程

## 准备工作

### 安装 AWS CLI

参考 [Installing, updating, and uninstalling the AWS CLI version 2
](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)

```
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws configure
# enter the AK / SK
```

### 安装 helm

参考 [Installing Helm](https://helm.sh/docs/intro/install/)

```
wget https://get.helm.sh/helm-v3.7.0-linux-amd64.tar.gz
tar -zxvf helm-v3.7.0-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
```

### 创建 AWS ECR repository

```
aws ecr create-repository --repository-name charts-test

# 记录下这里输出的repository 地址
```

### Helm 登陆 ECR

现在 `heml` OCI 支持还是试验阶段。需要通过环境变量启用。 

```
export HELM_EXPERIMENTAL_OCI=1

aws ecr get-login-password | helm registry login --username AWS --password-stdin 123456789101.dkr.ecr.ap-northeast-1.amazonaws.com
```

## 创建 Helm charts

```
mkdir charts-test
helm create charts-test
cd charts-test/templates/

# 创建一个Hello World configmap
cat << EOF > configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
    name: charts-test-cm
data:
    hello: world
EOF

cd ../../

# 打包
helm package charts-test/

# 默认的版本号是 0.1.0，所以会生成 charts-test-0.1.0.tgz
```

## 上传 Helm Charts 到 ECR

注意，我们采用OCI协议上传到ecr
```
heml push charts-test-0.1.0.tgz oci://123456789101.dkr.ecr.ap-northeast-1.amazonaws.com

# 检查上传结果
aws ecr describe-images --repository-name charts-test
```

## 下载&安装 Helm Charts 

```
# 从ECR repository charts-test 下载 tag 0.1.0的版本。 
helm pull oci://953228137468.dkr.ecr.ap-northeast-1.amazonaws.com/charts-test --version 0.1.0

# 将下载的 charts-test-0.1.0.tgz 安装
helm install charts-test charts-test-0.1.0.tgz

# 检查安装结果
helm status charts-test

# 清理
helm uninstall charts-test 
```

# References

- [AWS ECR OCI support](https://docs.aws.amazon.com/AmazonECR/latest/userguide/push-oci-artifact.html)
- [OCI Artifact Support In Amazon ECR
](https://aws.amazon.com/blogs/containers/oci-artifact-support-in-amazon-ecr/)
- [Helm Registry Docs](https://helm.sh/docs/topics/registries/)
- [Helm OCI Support Roadmap](https://github.com/helm/community/blob/main/hips/hip-0006.md)
- [Helm Release Notes](https://github.com/helm/helm/releases)

