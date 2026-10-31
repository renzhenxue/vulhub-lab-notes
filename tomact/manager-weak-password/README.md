# Tomcat Manager 后台弱口令导致 WAR 包部署 GetShell

## 1. 漏洞标识
- **环境**：Vulhub `tomcat/tomcat8`
- **漏洞类型**：后台弱口令 + 管理接口滥用（配置不当）
- **影响组件**：Tomcat 7+ Manager 应用
- **利用前提**：`tomcat-users.xml` 中存在弱口令账号，且 Manager 允许远程访问

## 2. 环境与复现
- 启动：`cd vulhub-master/tomcat/tomcat8 && docker-compose up -d` #你要输入自己放置的靶场文件路径，这里只是一个例子。
- 访问 `http://your-ip:8080/manager/html`，使用弱口令登录[images/01-login-success.png]
- 制作包含 JSP WebShell 的 WAR 包并上传部署[tomact/manager-weak-password/shell.jsp]
- 访问部署后的路径，使用客户端连接验证[02-success-shell.png]

## 3. 根因分析
- **直接原因**：管理员在 `conf/tomcat-users.xml` 中创建了 `tomcat/tomcat` 等弱口令账号，并授予了 `manager-gui` 等管理角色。
- **条件原因**：Manager 应用默认仅限 localhost 访问，但被修改为允许远程 IP 连接，使得外部攻击者可以触达登录入口。
- **本质**：运维便利功能（Web 端部署 WAR）在认证强度不足时，从“管理工具”变成了“攻击入口”。

## 4. 安全运维视角：修复与验证
### 4.1 修复方案
1.  **删除或禁用 Manager**（首选）：如果生产环境不需要通过 Web 界面部署应用，直接移除 `$CATALINA_HOME/webapps/manager` 和 `host-manager` 目录，这是最彻底的攻击面收敛。
2.  **强制强口令**：必须保留时，在 `tomcat-users.xml` 中为管理员设置高强度密码，并仅授予完成工作所必需的最小角色（如仅 `manager-script`，避免 `manager-gui` 暴露）。
3.  **限制访问来源**：在 `context.xml` 或网络层，将 `/manager/*` 限制在管理 VPN 或可信 IP 范围内，不对公网开放。

### 4.2 修复验证
- **复测弱口令**：使用原弱口令 `tomcat/tomcat` 登录，预期返回 401/403。
- **检查 Manager 可达性**：从非授权 IP 访问 `/manager/html`，预期连接被拒绝或超时。
- **确认角色最小化**：若保留账号，通过 `manager-script` 接口尝试 HTML 页面操作，预期无权限。

## 5. 检测与加固检查清单
-  扫描所有 Tomcat 实例的 `/manager/html` 是否对非管理网段开放
-  检查 `tomcat-users.xml` 中是否存在 `tomcat/tomcat`、`admin/admin` 等默认或常见弱口令
-  确认 `webapps` 下无残留的 `manager`、`host-manager` 目录
-  日志监控：关注 `/manager/deploy` 接口的异常 WAR 上传请求
