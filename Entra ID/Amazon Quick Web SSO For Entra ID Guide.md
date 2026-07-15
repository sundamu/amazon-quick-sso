# Tutorial: Amazon Quick 与 Microsoft Entra ID (Azure AD) IAM 联合身份认证

> **注意**：IAM 身份联合不支持将身份提供商的组同步到 Amazon Quick。

---

## 概述

本教程将演示如何设置 **Microsoft Entra ID（原 Azure AD）** 作为 Amazon Quick 的 SAML 2.0 联合身份认证服务。通过 SAML 联合，用户可以使用企业 Entra ID 凭证通过单点登录 (SSO) 访问 Amazon Quick。

整体流程：

```
用户 → Entra ID 登录 → SAML 断言 → AWS STS (AssumeRoleWithSAML) → 自动创建账户 → 进入 Amazon Quick Website

```

---

## 步骤一：在 Microsoft Entra ID 中创建企业应用

### 1.1 注册应用

1. 进入 **Microsoft Entra ID** → **Enterprise applications** → **New application**
2. 点击 **Create your own application**
3. 输入名称：`Amazon Quick Web`
4. 选择 **Integrate any other application you don't find in the gallery (Non-gallery)**
5. 点击 **Create**

### 1.2 配置 SAML 单点登录

1. 进入刚创建的应用 → **Single sign-on** → 选择 **SAML**
2. 在 **Basic SAML Configuration** 中，点击 **Edit**，填写：

| 字段 | 值 |
| --- | --- |
| **Identifier (Entity ID)** | `urn:amazon:webservices` |
| **Reply URL (Assertion Consumer Service URL)** | `https://signin.aws.amazon.com/saml` |
| **Sign on URL** | 留空 |
| **Relay State** | `https://quicksight.aws.amazon.com` |
| **Logout URL** | 留空 |

点击 **Save**

### 1.3 配置 SAML Attributes & Claims

点击 **Attributes & Claims** → **Edit**，配置以下 Claims：

必需 Claim 1：Role

| 设置 | 值 |
| --- | --- |
| **Name** | `https://aws.amazon.com/SAML/Attributes/Role` |
| **Namespace** | 留空 |
| **Source** | Attribute |
| **Source attribute** | 见下方说明 |

**Source attribute 的值**（格式）：

```
arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>,arn:aws:iam::<ACCOUNT_ID>:saml-provider/QuickEntraID

```

> ⚠️ **注意**：此处的 `QuickEntraID` 是后续步骤二中创建的 Identity Provider 的名字（可自定义），这里需要保持一致。 `<ROLE_NAME>` 是步骤 3.2 中创建的 IAM Role 的名字，也要保持一致。

示例：

```
arn:aws:iam::831762732388:role/QuickEntraIDFederatedRole,arn:aws:iam::831762732388:saml-provider/QuickEntraID

```

必需 Claim 2：RoleSessionName

| 设置 | 值 |
| --- | --- |
| **Name** | `https://aws.amazon.com/SAML/Attributes/RoleSessionName` |
| **Namespace** | 留空 |
| **Source** | Attribute |
| **Source attribute** | `user.userprincipalname`（从下拉框选择，不要手动输入） |

Claim 3：SessionDuration

| 设置 | 值 |
| --- | --- |
| **Name** | `https://aws.amazon.com/SAML/Attributes/SessionDuration` |
| **Namespace** | 留空 |
| **Source** | Attribute |
| **Source attribute** | `3600`（秒，即 1 小时） |

### 1.4 下载 Federation Metadata XML

1. 在 **SAML Signing Certificate** 区域
2. 找到 **Federation Metadata XML** → 点击 **Download**
3. 保存文件为 `AmazonQuickWeb-metadata.xml`

> 📌 **保存好这个文件，后续步骤需要上传到 AWS IAM。**

---

## 步骤二：在 AWS IAM 中创建 SAML Identity Provider

1. 登录 [AWS IAM 控制台](https://console.aws.amazon.com/iam/)
2. 左侧导航栏选择 **Identity providers** → **Add provider**
3. 填写以下信息：

| 字段 | 值 |
| --- | --- |
| **Provider type** | SAML |
| **Provider name** | `自定义名字，如 QuickEntraID` |
| **Metadata document** | 上传步骤 1.4 下载的 `AmazonQuickWeb-metadata.xml` |

1. 点击 **Add provider**

---

## 步骤三：创建 IAM Role（SAML 2.0 联合身份信任实体）

### 3.1 创建策略

先创建策略，再创建 Role 时直接附加。

点击 IAM 控制台 → **Policies** → **Create policy** → **JSON**，粘贴以下内容：

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "quicksight:CreateReader"
            ],
            "Effect": "Allow",
            "Resource": "arn:aws:quicksight::<ACCOUNT_ID>:user/${aws:userid}"
        }
    ]
}

```

- **Policy name**：`QuickCreateReader`
- 点击 **Create policy**

> 💡 **角色说明**：
> - `quicksight:CreateReader` → 用户首次登录自动创建为 **Reader**
> - `quicksight:CreateUser` → 用户首次登录自动创建为 **Author**
> - `quicksight:CreateAdmin` → 用户首次登录自动创建为 **Admin** 

⚠️ **特别注意**：由于历史原因，以上角色都是对应之前 QuickSight 的角色。使用 Quick AI 功能，需要用户首次登录后，后台管理员将用户角色升级为以下三种，以获取相应的 Quick GenAI 能力：

> - **Reader Pro (Professional)**
> - **Author Pro (Enterprise)**
> - **Admin Pro (Enterprise)**

### 3.2 创建 Role 并附加策略

1. IAM 控制台 → **Roles** → **Create role**
2. **Trusted entity type**：选择 **SAML 2.0 federation**
3. **SAML provider**：选择步骤二创建的 SAML identity provider
4. **Condition** 保持默认（无需添加额外条件）
5. 点击 **Next** 进入 Add permissions 步骤
6. 搜索并勾选 `QuickCreateReader`
7. 点击 **Next: Tags** → **Next: Review**
8. **Role name**：`QuickEntraIDFederatedRole`
9. 点击 **Create role**

### 3.3 更新 Role 的 Trust Policy

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::<ACCOUNT_ID>:saml-provider/QuickEntraID"
            },
            "Action": "sts:AssumeRoleWithSAML",
            "Condition": {
                "StringEquals": {
                    "SAML:aud": "https://signin.aws.amazon.com/saml"
                }
            }
        }
    ]
}

```

> ⚠️ **注意**：此处 `"Federated": "arn:aws:iam::<ACCOUNT_ID>:saml-provider/QuickEntraID"` 中的 `QuickEntraID` 是之前创建的 SAML Identity Provider Name，需要保持一致。

---

## 步骤四：在 Entra ID 中分配用户和组

1. 回到 Entra ID Portal → Enterprise applications → `Amazon Quick Web`
2. 左侧选择 **Users and groups** → **Add user/group**
3. 选择需要访问 Quick 的 **用户** 或 **组**
4. 点击 **Assign**

> 只有被分配的用户/组才能通过 SSO 登录到 Quick。

## 步骤五：（可选但强烈推荐）配置 Service Provider Initiated SSO（SP 发起的联合登录）

> 📎 参考文档：[Setting up service provider–initiated federation with Quick Enterprise edition](https://docs.aws.amazon.com/quick/latest/userguide/setup-quicksight-to-idp.html)

默认情况下，用户需要先访问 IdP（如 Entra ID 的 MyApps 门户）才能登录 Quick（即 **IdP-initiated SSO**）。配置 **Service Provider Initiated SSO** 后，用户可以直接访问 Quick 登录页面，Quick 会自动将用户重定向到 Entra ID 进行身份验证，验证通过后再跳转回 Quick。

### 5.1 获取 Entra ID 的 IdP URL

1. 进入 **Microsoft Entra ID** → **Enterprise applications** → 选择之前创建的 `Amazon Quick Web`
2. 左侧选择 **Properties**（属性）
3. 找到 **User access URL**，格式如下：

```
https://launcher.myapps.microsoft.com/api/signin/<app_id>?tenantId=<tenant_id>

```

示例：

```
https://launcher.myapps.microsoft.com/api/signin/1cc7efd0-8ad8-4250-b42a-50263b2414c4?tenantId=3d81889e-b9ac-40b2-b565-f5f9f34ad869

```

### 5.2 在 Quick 中配置 SP Initiated SSO

1. 用管理员账号登录 Quick → 点击右上角 **Manage account**
2. 左侧导航选择 **Identity** → **SSO**
3. 找到 **Service Provider Initiated SSO** 区域
4. 在 **Configuration** 中填写：

| 字段 | 值 |
| --- | --- |
| **IdP URL** | 步骤 5.1 中获取的 URL（如 `https://launcher.myapps.microsoft.com/api/signin/<app_id>?tenantId=<tenant_id>`） |
| **IdP redirect URL parameter** | `RelayState` |

1. 确认配置无误后，将 **Status** 设置为 **ON**

---

## 步骤六：（可选）配置联合用户邮箱自动同步

默认情况下，联合用户首次登录 Quick 时会弹出页面要求手动输入邮箱地址。开启 Email Syncing 后，Quick 会自动从 SAML 断言中获取预配置的邮箱，跳过手动填写步骤。

> 📎 参考文档：[Configuring email syncing for federated users in Quick](https://docs.aws.amazon.com/quick/latest/userguide/jit-email-syncing.html)

### 6.1 更新 IAM Role 的 Trust Policy（添加 sts:TagSession）

进入 IAM → Roles → `QuickEntraIDFederatedRole` → **Trust relationships** → **Edit trust policy**，在现有 Statement **之后**添加第二个 Statement：

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::<ACCOUNT_ID>:saml-provider/QuickEntraID"
            },
            "Action": "sts:AssumeRoleWithSAML",
            "Condition": {
                "StringEquals": {
                    "SAML:aud": "https://signin.aws.amazon.com/saml"
                }
            }
        },
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::<ACCOUNT_ID>:saml-provider/QuickEntraID"
            },
            "Action": "sts:TagSession"
        }
    ]
}

```

### 6.2 在 Entra ID 中添加 PrincipalTag:Email Claim

在 Enterprise application → **Single sign-on** → **Attributes & Claims** → **Edit** → **Add new claim**：

| 设置 | 值 |
| --- | --- |
| **Name** | `https://aws.amazon.com/SAML/Attributes/PrincipalTag:Email` |
| **Namespace** | 留空 |
| **Source** | Attribute |
| **Source attribute** | `user.userprincipalname`（从下拉框选择） |

> ⚠️ 确保 `user.userprincipalname` 的值是有效邮箱格式。如果用户的 UPN 不是邮箱格式，改用 `user.mail`（需确认该字段不为空）。

### 6.3 在 Quick 中开启 Email Syncing

1. 用管理员账号登录 Quick → 点击右上角 **Manage account**
2. 左侧导航选择 **Identity** → **SSO**
3. 找到 **Email Syncing for Federated Users** → 选择 **ON**
4. 保存

开启后，新用户通过 SSO 首次登录时不再弹出填写邮箱的页面，Quick 会自动使用 SAML 断言中 PrincipalTag:Email 传递的邮箱地址。

---

## 验证

完成所有配置后，测试 SSO 登录：

### IdP-initiated 登录

1. 访问 Entra ID MyApps 门户：`https://myapps.microsoft.com`
2. 点击 `Amazon Quick Web` 图标
3. 应自动跳转到 Amazon Quick 控制台

### SP-initiated 登录（如果配置了步骤五）

1. 访问 Quick 登录页面：`https://quicksight.aws.amazon.com`
2. 系统应自动重定向到 Entra ID 登录页面
3. 输入企业凭证后，自动返回 Quick

