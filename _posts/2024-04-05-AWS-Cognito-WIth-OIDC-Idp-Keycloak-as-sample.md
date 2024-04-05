# AWS Cognito 使用 keycloak 作为SAML IdP

## 安装 keycloak

参考keycloak文档

## 添加 keycloak client

添加新的client， protocl 选择 oidc

client-id 自行填写

![](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-05-AWS-Cognito-WIth-OIDC-Idp-Keycloak-as-sample/kc-create-client.png?raw=true)

注意，下图中的值，需要等aws cognito 域名配置后才有。 这里先填上。 创建好后也可以更改。其中的域名格式为：

`https://mydomain.us-east-1.amazoncognito.com/oauth2/idpresponse`

![](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-05-AWS-Cognito-WIth-OIDC-Idp-Keycloak-as-sample/kc-valid-redirect.png?raw=true)


## 配置 keycloak 属性

为了安全，推荐使用 credentials，要求访问者提供密钥。

![](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-05-AWS-Cognito-WIth-OIDC-Idp-Keycloak-as-sample/kc-enable-auth.png?raw=true)


在下图位置找到密码，后面aws console要用到。

![](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-05-AWS-Cognito-WIth-OIDC-Idp-Keycloak-as-sample/kc-cliet-key.png?raw=true)

## 配置 aws

### 创建 cognito user pool

创建userpool，一般钩上 username 登录。

![](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-05-AWS-Cognito-WIth-OIDC-Idp-Keycloak-as-sample/aws-add-user-pool.png?raw=true)

使用默认值即可。

![](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-05-AWS-Cognito-WIth-OIDC-Idp-Keycloak-as-sample/aws-create-user-pool-policy.png?raw=true)

配置IdP信息，信息比较多，我们单独配置，这里先跳过。

![](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-05-AWS-Cognito-WIth-OIDC-Idp-Keycloak-as-sample/aws-user-pool-identity.png?raw=true)

配置userpool的名字以及自定义domain。 这个domain与前面 keycloak中使用的应该一致。

![](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-05-AWS-Cognito-WIth-OIDC-Idp-Keycloak-as-sample/aws-create-user-pool.png?raw=true)

最后，点击创建。完成后，我们进一步配置cognito user pool。

在下图中，点击 add identity provider

![](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-05-AWS-Cognito-WIth-OIDC-Idp-Keycloak-as-sample/aws-add-idp.png?raw=true)

进入下图界面，选择 openid connect

![](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-05-AWS-Cognito-WIth-OIDC-Idp-Keycloak-as-sample/aws-update-identity-provider.png?raw=true)


更新一下信息，其中provider name 自行定义， client-id 同 keycloak client-id， secret 在 keycload client credentials里面获取。

![](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-05-AWS-Cognito-WIth-OIDC-Idp-Keycloak-as-sample/aws-oidc-connect-info.png?raw=true)


最后是issuer相关的信息，如下图。 

aws-oidc-issuer

具体issuer的值， 从 keycloak中，找到 配置文件的link

![](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-05-AWS-Cognito-WIth-OIDC-Idp-Keycloak-as-sample/kc-oidc-meta.png?raw=true)

点开后，找到 issuer 字段。即可

![](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-05-AWS-Cognito-WIth-OIDC-Idp-Keycloak-as-sample/kc-oidc-meta.png?raw=true)

## 登录测试


通过 下图中， 找到 hosted ui 的URL， 然后点开edit

![](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-05-AWS-Cognito-WIth-OIDC-Idp-Keycloak-as-sample/aws-hosted-ui.png?raw=true)


需要编辑 callback 的URL，也就是我们的用户应用。 然后，支持的userpool，需要钩上我们刚添加的IdP，也就是 keycloak
![](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-05-AWS-Cognito-WIth-OIDC-Idp-Keycloak-as-sample/aws-final-app-cb.png?raw=true)



在hosted ui 界面，点击 view hosted ui，出现下图界面，点击我们创建的 oidc provider 的按钮
![](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-05-AWS-Cognito-WIth-OIDC-Idp-Keycloak-as-sample/aws-cognito-hostui.png?raw=true)

登录keycloak后，可以看到转到了我们定义的 app （ jwt.io），并附带有对应的验证code

![](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-05-AWS-Cognito-WIth-OIDC-Idp-Keycloak-as-sample/jwt-code.png?raw=true)

[refer: AWS Cognito: Adding OIDC identity providers to a user pool](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-oidc-idp.html#cognito-user-pools-oidc-idp-step-1)
