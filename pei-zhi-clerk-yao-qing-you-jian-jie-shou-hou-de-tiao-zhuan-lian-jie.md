# 配置 Clerk 邀请邮件接受后的跳转链接

### 方法 1: 通过代码设置

在您的代码中，当创建邀请时，您已经通过 `redirect_url` 参数指定了跳转链接：

```python
invitation = await clerk_service.create_invitation(
    email=email,
    organization_id=organization.id,
    role="admin",
    redirect_url=f"/onboarding/admin?org_id={organization.id}"
)
```

这个参数告诉 Clerk 在用户完成身份验证后将他们重定向到哪里。这是最灵活的方法，因为您可以动态设置不同类型用户的不同跳转路径。

### 方法 2: 在 Clerk 仪表板中配置

1. 在 Clerk 仪表板中，导航到 **Developers** → **Paths**
2. 找到 **After sign up URL** 和/或 **After sign in URL** 设置
3. 您也可以在 **Customization** → **Emails** 部分找到邀请邮件模板设置

如果您在上述位置找不到这些设置，还可以尝试：

1. 转到 **User & Authentication** 部分
2. 查找 **Email, phone, username** 设置
3. 在认证设置中查找 "After sign up" 或 "After sign in" 重定向选项

### 确保跳转链接正常工作的关键步骤

1. **允许的重定向域**：确保您的应用域名已在 Clerk 中配置为允许的重定向域
2. **路径配置**：确保您的重定向路径（如 `/onboarding/admin` 和 `/onboarding/general`）已在允许列表中
3. **验证邀请流程**：测试整个流程，确保邀请邮件中的链接正确跳转到您期望的 URL

### 查看当前邀请邮件配置

您可以通过以下方式查看当前的邀请邮件设置：

1. 在 Clerk 仪表板中转到 **Customization** → **Emails**
2. 找到 "Organization Invitation" 邮件模板
3. 查看模板中的链接设置

如果您的代码中已经指定了 `redirect_url` 参数（如您所示），那么该参数会覆盖 Clerk 仪表板中的默认设置，但域名仍需在 Clerk 中配置为允许的重定向域。
