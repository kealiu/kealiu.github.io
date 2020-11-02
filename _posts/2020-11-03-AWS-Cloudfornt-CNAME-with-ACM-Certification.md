简介： AWS Cloudfront 是AWS的CDN服务，覆盖全球大部分区域。不管你的服务是否部署在AWS上，都能使用AWS Cloudfront 对服务进行加速。本文简单介绍Cloudfront，然后通过AWS Certification Manager来申请域名证书，并配置到Cloudfront进行自定义域名配置。

# Cloudfront 简介

> Amazon CloudFront 是一项快速内容分发网络 (CDN) 服务，可以安全地以低延迟和高传输速度向全球客户分发数据、视频、应用程序和 API，全部都在开发人员友好的环境中完成。CloudFront 已与 AWS 集成 – 同时包括直接连接到 AWS 全球基础设施的物理位置以及其他 AWS 服务。CloudFront 可与多种服务无缝集成，包括用于 DDoS 缓解的 AWS Shield、Amazon S3、用作您的应用程序来源的 Elastic Load Balancing 或 Amazon EC2 以及用于在靠近浏览者客户用户的位置运行自定义代码和自定义用户体验的 Lambda@Edge。最后，如果您使用的是 Amazon S3、Amazon EC2 或 Elastic Load Balancing 等 AWS 源服务，那么您也无需为这些服务与 CloudFront 之间传输的任何数据支付费用。

Reference: [AWS Cloudfront 产品](https://aws.amazon.com/cn/cloudfront/)

# 创建 Cloudfront

关于创建cloudfront的流程，参见 [Resize Image On The Fly Using Lambda At Edge And Cloudfront With S3
](https://kealiu.github.io/Resize-Image-On-The-Fly-Using-Lambda-at-Edge-and-Cloudfront-With-S3/)，不再累述。

# 设置自定义域名

进入Cloudfront Distribution的设置菜单，选择`General` 标签页，可以看到对应的`Alternate Domain Names (CNAMEs)` 选项。

![CloudfrontCNAME](https://github.com/kealiu/kealiu.github.io/raw/master/images/2020-11-02-Cloudfront-With-Custom-Domain/CloudfrontCNAME.png)
如果直接填写域名，并保存，将直接失败，原因在于Cloudfront强制要求HTTPS，在设置自定义域名的时候必须提供SSL。 而默认的`*.cloudfront.net`证书并不包含用户自定义域名。我们需要点击`Request or Import a Certification with ACM` 来生成对应的证书。

## 添加ACM证书

点击申请ACM证书后，进入ACM证书申请环境。请务必注意右上角的Region显示的是 `N.Virginia`，因为Cloudfront是一个全球性的服务，起大部分配置都必须要在`us-east-1`，也就是`N.Virginia`进行。

### Step 1

填写自己的自定义域名，告诉AWS我们想要申请证书的域名。AWS支持泛域名，可以申请 `*.exmaple.com` 这样的域名证书。

![ACMCreateNewRequest](https://github.com/kealiu/kealiu.github.io/raw/master/images/2020-11-02-Cloudfront-With-Custom-Domain/ACMCreateNewRequest.png)

### Step 2

为了验证你是否对这个域名有管理权限，AWS会对域名进行验证。 你可以选择通过DNS或者Email进行验证。这里我们选择DNS进行验证。

![ACMRequestValidationMethod](https://github.com/kealiu/kealiu.github.io/raw/master/images/2020-11-02-Cloudfront-With-Custom-Domain/ACMRequestValidationMethod.png)

### Step 3

添加tag，按照需求添加

![ACMRequestTags](https://github.com/kealiu/kealiu.github.io/raw/master/images/2020-11-02-Cloudfront-With-Custom-Domain/ACMRequestTags.png)

### Step 4 

对上面的选项以及值进行Review，确定都准确无误后，可以开始验证流程。AWS会给出一个域名，以及对应的CNAME值，需要你按AWS给出的值设置到你的DNS中，用以证明你对该域名的控制权。

![ACMRequestValidated](https://github.com/kealiu/kealiu.github.io/raw/master/images/2020-11-02-Cloudfront-With-Custom-Domain/ACMRequestValidated.png)

### Step 5

在你的域名提供商处，设置域名的CNAME。一般来说，添加一个CNAME记录，域名是AWS提供的域名，值是AWS提供的CNAME值。保存。由于DNS有一定时间的生效期，需要等待一定时间待AWS验证成功。

![ACMDomainValidation](https://github.com/kealiu/kealiu.github.io/raw/master/images/2020-11-02-Cloudfront-With-Custom-Domain/ACMDomainValidation.png)

### Step 6

等待验证通过后，该域名将获得对应的证书。

## Cloudfornt启用该证书

回到Cloudfront证书配置页面，`Alternate Domain Names (CNAMEs)` 填上期望的自定义域名，同时选择`Custome SSL Certification` 并选择ACM中刚申请的对应域名的证书。然后保存。 

Cloudfront需要在全球所有节点进行配置更新，需要一定时间。大概5分钟后，确认Distribution状态已经为Deployed，即可用自定义域名进行访问了。



