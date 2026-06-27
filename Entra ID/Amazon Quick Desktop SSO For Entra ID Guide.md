# Tutorial: Amazon Quick Desktop 与 Microsoft Entra ID 企业 SSO 配置

> 📎 参考文档：[Setting up Amazon Quick on desktop for enterprise deployments](https://docs.aws.amazon.com/quick/latest/userguide/desktop-enterprise-setup.html)

---

## 概述

本教程将演示如何配置 **Microsoft Entra ID（原 Azure AD）** 作为 Amazon Quick Desktop 应用的 OIDC 身份认证提供商，实现企业单点登录（SSO）。

与 Web 版使用 SAML 协议不同，Desktop 应用使用 **OpenID Connect (OIDC)** 协议

---

## 前提条件

- 已有 AWS 账号并订阅了 Amazon Quick（Identity Region 必须为 **us-east-1**）
- 拥有 Amazon Quick 管理员权限
- 拥有 Microsoft Entra ID 管理员权限

---

## 步骤一：在 Microsoft Entra ID 中创建 OIDC 应用

### 1.1 注册应用

1. 进入 **Microsoft Entra ID** → **App registrations** → **New registration**
2. 配置以下设置：

| 设置 | 值 |
| --- | --- |
| **Name** | `Amazon Quick Desktop` |
| **Supported account types** | Accounts in this organizational directory only (Single tenant) |
| **Redirect URI platform** | Public client/native (mobile & desktop) |
| **Redirect URI** | `http://localhost:18080` |

1. 点击 **Register**

### 1.2 记录关键信息

注册完成后，在 **Overview** 页面记录以下信息（后续步骤需要）：

- **Application (client) ID**
- **Directory (tenant) ID**

### 1.3 配置 API 权限

1. 进入刚创建的 App Registration → **API permissions** -> **Application registration** -> **Add a permission**
2. 选择 **Microsoft Graph** → **Delegated permissions**
3. 添加以下权限：

| 权限 | 说明 |
| --- | --- |
| `openid` | 基础 OIDC 认证 |
| `email` | 获取用户邮箱 |
| `profile` | 获取用户基本信息 |
| `offline_access` | 获取 Refresh Token（长期会话必需） |

点击 **Add permissions**

⚠️点击 **Grant admin consent for [your organization]**

### 1.4 配置身份验证设置

1. 进入 App Registration → **Authentication**
2. 在 **Advanced settings** 下，将 **Allow public client flows** 设置为 **Yes**
3. 确认 `http://localhost:18080` 已列在 **Mobile and desktop applications** 下
4. 点击 **Save**

### 1.5 分配用户/组

1. 进入 **Microsoft Entra ID** → **Enterprise applications** → 找到 `Amazon Quick Desktop`
2. 左侧导航 **Users and groups** → **Add user/group**
3. 选择需要使用 Quick Desktop 的用户或组
4. 点击 **Assign**

### 1.6 确认 OIDC Endpoints

将 `<TENANT_ID>` 替换为你的 Directory (tenant) ID：

| 字段 | 值 |
| --- | --- |
| **Issuer URL** | `https://login.microsoftonline.com/<TENANT_ID>/v2.0` |
| **Authorization endpoint** | `https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/authorize` |
| **Token endpoint** | `https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token` |
| **JWKS URI** | `https://login.microsoftonline.com/<TENANT_ID>/discovery/v2.0/keys` |

> 📌 **保存好这些 URL 和 Client ID，步骤二需要填入 Amazon Quick 管理控制台。**

---

## 步骤二：在 Amazon Quick 管理控制台配置 Extension Access

### 2.1 添加 Extension Access

1. 用管理员账号登录 Quick → 点击右上角 **Manage account**
2. 左侧导航 **Permissions** → **Extension access**
3. 点击 **Add extension access**
4. 选择 **Desktop application for Quick** extension → 点击 **Next**
5. 填写以下信息：

> 📌 以下 URL 中的 `<TENANT_ID>` 替换为步骤 1.2 中记录的 **Directory (tenant) ID**。

| 字段 | 值 |
| --- | --- |
| **Name** | 为此 Extension Access 命名（如 `Entra ID Desktop SSO`） |
| **Description** | （可选）描述 |
| **Issuer URL** | `https://login.microsoftonline.com/<TENANT_ID>/v2.0` |
| **Authorization Endpoint** | `https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/authorize` |
| **Token Endpoint** | `https://login.microsoftonline.com/<TENANT_ID>/oauth2/v2.0/token` |
| **JWKS URI** | `https://login.microsoftonline.com/<TENANT_ID>/discovery/v2.0/keys` |
| **Client ID** | 步骤 1.2 中记录的 Application (client) ID |

1. 点击 **Add**

> ⚠️ **重要**：Extension Access 创建后无法编辑。如果填写有误，需要删除后重新创建。

### 2.2 创建 Extension

1. 在 Quick 控制台左侧导航 **Connect apps and data** → **Extensions**
2. 点击 **Add extension**
3. 选择刚创建的 **Desktop application for Quick** Extension Access → 点击 **Next**
4. 点击 **Create**

---

## 完成 🎉

配置完成后，企业用户可以通过以下方式使用 Amazon Quick Desktop：

1. 打开 Desktop 应用 → 选择 **SSO 登录**
2. 浏览器自动跳转到 Entra ID 登录页面
3. 输入企业凭证完成认证
4. 返回 Desktop 应用，开始使用