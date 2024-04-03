# 如何使用SAML连接第三方IdP登陆AWS Console
以keycloak为例

## 安装 keycloak

参考keyloak文档

## 配置 keycloak

### 创建realm

realm是keycloak的一个用户域。 具体创建参考 keycloak 文档

### 创建 client

client 使用 SAML/OpenID/OAuth 等第三方应用，使用import功能，如下图

AWS SAML metadata可以从 [AWS Saml Metadata](https://signin.aws.amazon.com/static/saml-metadata.xml) 下载。

![Add Provider in AWS](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-03-AWS-Console-WIth-SAML-Idp-Keycloak-as-sample/kc-import-client.png)

### 调整 client 属性

设置 `IdP initiated SSO URL name`，最终的登陆地址与之有关，为：`{server-root}/realms/{realm}/protocol/saml/clients/{client-url-name}`

![Set init URL](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-03-AWS-Console-WIth-SAML-Idp-Keycloak-as-sample/kc-config-client-init-url.png)

## 配置AWS

现在 对应realm 的 `saml-metadata.xml`:
![download keycloak realm saml meta](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-03-AWS-Console-WIth-SAML-Idp-Keycloak-as-sample/kc-download-saml-meta.png)

然后，使用下载的`saml-metadata.xml`，在AWS中添加 IdP
![Add Provider in AWS](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-03-AWS-Console-WIth-SAML-Idp-Keycloak-as-sample/aws-add-provider.png)

## 创建Role

在AWS为 IdP 创建 Role，Role trust IdP选择刚导入的keycloak。

需要记住：
1. 刚导入的 `IdP ARN`
2. 刚新建的 `Role ARN`

keycloak中，role格式为 **`Role ARN`,`IdP ARN`**， 如图。

然后， 在 keycloak 的client中，创建对应的Role：
![Add Role in keycloak](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-03-AWS-Console-WIth-SAML-Idp-Keycloak-as-sample/kc-create-role.png)

## 创建keycloak用户并绑定role

创建用户
![Add User in keycloak](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-03-AWS-Console-WIth-SAML-Idp-Keycloak-as-sample/kc-create-user.png)

![Add Role for user in keycloak](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-03-AWS-Console-WIth-SAML-Idp-Keycloak-as-sample/kc-user-role-mapper.png)

分配在keycloak client中创建的role
![Add Role for user in keycloak](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-03-AWS-Console-WIth-SAML-Idp-Keycloak-as-sample/kc-user-role-mapping.png)

## （optional）修改属性mapping

如果不幸，出现aws报错 ，如 [Error: Your request included an invalid SAML response. To logout, click here.](https://docs.aws.amazon.com/IAM/latest/UserGuide/troubleshoot_saml.html)，需要进行如下属性修改

在 clientscope tab，编辑 aws 属性
![enter client scope](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-03-AWS-Console-WIth-SAML-Idp-Keycloak-as-sample/kc-attr-session-name.png)

删除 Role, RoseSessionName 属性，然后添加如图的mapping：

![Add Role Mapper](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-03-AWS-Console-WIth-SAML-Idp-Keycloak-as-sample/kc-attr-role.png)

![Add Session Name mapper](https://github.com/kealiu/kealiu.github.io/blob/master/images/2024-04-03-AWS-Console-WIth-SAML-Idp-Keycloak-as-sample/kc-attr-session-name.png)
