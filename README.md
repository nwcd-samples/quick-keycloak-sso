# Amazon Quick SSO with Keycloak

基于 Keycloak 为 Amazon Quick 提供 SAML / OIDC 单点登录。

## 一、部署指南

### 0. 前置准备

开通 Amazon Quick 账户，选择支持 SSO 的身份验证方法。

- [ ] 注册 AWS 账户
- [ ] 开通 Amazon Quick
  - [ ] 填写账户名称，如公司名称 `example`
  - [ ] 选择默认区域 US East（N. Virginia）
  - [ ] 选择身份验证方法："密码或单点登录"（推荐）

### 1. 部署 Keycloak 基础设施

使用 CloudFormation 模板一键部署 Keycloak 所需的基础设施，包括 EC2、ALB、PostgreSQL、IAM Roles、Lambda 等。

- [ ] 确定 Keycloak 域名，如 `sso.example.com`
- [ ] 使用 CloudFormation 模板部署
  - [ ] 部署过程中在 ACM 控制台完成证书 DNS 验证
- [ ] 配置 DNS 解析，将域名 CNAME 指向 ALB DNS Name
- [ ] 等待部署完成，确认 Keycloak 可访问：`https://sso.example.com`

### 2. 配置 Keycloak 管理员

模板部署时会创建一个临时管理员（Bootstrap Admin），此步骤将其替换为正式管理员。

- [ ] 使用临时管理员登录 Keycloak 管理控制台：`https://sso.example.com`
- [ ] 创建正式管理员账户，设置用户名、邮箱、密码，分配 admin 角色
- [ ] 使用正式管理员登录，禁用或删除 Bootstrap Admin（临时管理员）

### 3. 运行 Bootstrap

通过脚本自动配置 Keycloak 的 Realm、Client、Roles、Groups、用户等，无需手动逐项操作。

- [ ] SSH 登录 EC2，进入 `/opt/keycloak/bootstrap/`
- [ ] 编辑 `.env`，填写：
  - Keycloak 管理员凭据（Step 2 创建的正式管理员）
  - Quick 管理员信息（用户名、邮箱）
  - Quick 账户名（Step 0 填写的账户名称）
  - AWS 账户 ID
- [ ] 运行 `./bootstrap.sh`
- [ ] 确认脚本输出无报错

### 4. 配置 AWS IAM

创建 SAML Identity Provider，建立 AWS 与 Keycloak 之间的信任关系。

- [ ] 下载 SAML Metadata：`https://sso.example.com/realms/quick/protocol/saml/descriptor`
- [ ] 在 IAM 控制台创建 SAML Identity Provider
  - 名称：`keycloak`（与模板中 IAM Role 信任策略一致）
  - 上传 Metadata XML 文件

### 5. 配置 Amazon Quick SSO

在 Quick 管理控制台启用 SAML 单点登录，将 Keycloak 作为身份提供商。

- [ ] IdP URL：`https://sso.example.com/realms/quick/protocol/saml/clients/aws`
- [ ] IdP 重定向 URL 参数：`RelayState`
- [ ] 开启服务提供商启动的 SSO
- [ ] 开启联合身份用户的电子邮件同步

### 6. 配置 Quick Desktop 扩展访问

通过 OIDC 协议为 Quick Desktop 客户端提供 SSO 登录能力。

- [ ] 添加扩展访问，配置以下参数：
  - Issuer URL：`https://sso.example.com/realms/quick`
  - Authorization Endpoint：`https://sso.example.com/realms/quick/protocol/openid-connect/auth`
  - Token Endpoint：`https://sso.example.com/realms/quick/protocol/openid-connect/token`
  - JWKS URI：`https://sso.example.com/realms/quick/protocol/openid-connect/certs`
  - Client ID：`amazon-quick-desktop`
- [ ] 添加扩展

---

## 二、登录方式

### 1. Web 端

通过 IAM 联合用户名称识别（角色/邮箱），对应 Quick 用户管理中的用户名，如 `QuickAdminProRole/admin@example.com`。

#### 1.1 IAM 用户（Quick 默认注册用户）

使用 AWS 账号密码登录：

`https://quicksight.aws.amazon.com/sn/account/example/start?enable-sso=0`

#### 1.2 Keycloak 用户

使用 Keycloak 用户密码登录（SSO）：

`https://quicksight.aws.amazon.com/sn/account/example/start`

#### 1.3 Keycloak 用户中心

用户可自行管理个人信息、修改密码、查看登录设备：

`https://sso.example.com/realms/quick/account`

### 2. Desktop 端

通过邮件地址识别，对应 Quick 用户管理中的电子邮件，如 `admin@example.com`。

#### 2.1 IAM 用户（Quick 默认注册用户）

- 需要在 Keycloak 中创建一个和 IAM 用户邮箱相同的用户（用户名建议相同）
- 该用户不加入任何组（禁止 Web 端权限，因为本方案基于组绑定 Role 实现授权）

#### 2.2 Keycloak 用户

- 首次登录（包括 Web 登录）会自动在 Quick 创建用户，并停留在 Web 端
- 再次点击 Desktop 的 SSO 登录及后续登录，跳转正常
