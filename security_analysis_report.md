# Nacos 3.0.3-SNAPSHOT 安全漏洞分析报告

## 项目概览
- **项目名称**: Alibaba Nacos
- **版本**: 3.0.3-SNAPSHOT 
- **技术栈**: Java 17, Spring Boot 3.4.4, Maven
- **项目类型**: 微服务注册中心和配置管理平台

## 🔴 高危安全漏洞

### 1. 默认Token密钥为空（CVE类风险）
**风险等级**: 🔴 高危
**位置**: `bootstrap/src/main/resources/application.properties:258`
```properties
nacos.core.auth.plugin.nacos.token.secret.key=
```
**问题描述**: 
- Token密钥默认为空字符串，攻击者可以利用空密钥伪造JWT token
- 对应已知漏洞：QVD-2023-6271身份登录绕过
- 可能导致完全绕过身份认证机制

**修复建议**:
```properties
nacos.core.auth.plugin.nacos.token.secret.key=<强随机密钥>
```

### 2. 默认弱口令
**风险等级**: 🔴 高危  
**位置**: 多个测试文件中使用了默认密码 `nacos/nacos`
```java
// test/core-test/src/test/java/com/alibaba/nacos/test/core/auth/RoleCoreITCase.java:95
Params.newParams().appendParam("username", "nacos").appendParam("password", "nacos").done()
```
**问题描述**:
- 默认管理员账号使用弱口令
- 生产环境可能延续使用默认密码

**修复建议**: 强制要求首次启动时修改默认密码

### 3. SQL注入漏洞风险（Standalone模式）
**风险等级**: 🔴 高危
**相关CVE**: 类似2024年7月公开的Nacos SQLi to RCE漏洞
**问题描述**:
- Standalone模式下使用Derby数据库
- 存在SQL执行接口，可能被滥用导致SQL注入
- 可能升级为远程代码执行

**修复建议**: 升级到最新版本并禁用 `derbyOpsEnabled`

## 🟡 中危安全问题

### 4. XSS防护不完整
**风险等级**: 🟡 中危
**位置**: `console/src/main/java/com/alibaba/nacos/console/filter/XssFilter.java`
```java
private static final String CONTENT_SECURITY_POLICY = "script-src 'self'";
```
**问题描述**:
- CSP策略过于宽松，仅限制script-src
- 缺少其他XSS防护措施如object-src, style-src等
- 未对输入进行充分的XSS过滤

**修复建议**: 
```java
private static final String CONTENT_SECURITY_POLICY = 
    "default-src 'self'; script-src 'self'; object-src 'none'; style-src 'self' 'unsafe-inline'";
```

### 5. CSRF保护被禁用
**风险等级**: 🟡 中危
**位置**: 
- `console/src/main/java/com/alibaba/nacos/console/config/ConsoleWebConfig.java:139`
- `plugin-default-impl/nacos-default-auth-plugin/src/main/java/com/alibaba/nacos/plugin/auth/impl/configuration/web/NacosAuthPluginWebConfig.java:70`

```java
http.csrf(AbstractHttpConfigurer::disable);
```
**问题描述**: CSRF保护被完全禁用，可能导致跨站请求伪造攻击

**修复建议**: 启用CSRF保护或使用token验证机制

## 🟢 低危安全问题

### 6. 依赖组件版本
**风险等级**: 🟢 低危
**当前版本**:
- Spring Boot: 3.4.4 ✅ (较新版本)
- JJWT: 0.11.2 ✅ (稳定版本)
- MySQL Connector: 8.2.0 ✅ (较新版本)

**问题描述**: 大部分依赖版本较新，安全风险相对较低

### 7. 密码加密方式
**风险等级**: 🟢 低危
**位置**: `plugin-default-impl/nacos-default-auth-plugin/src/main/java/com/alibaba/nacos/plugin/auth/impl/utils/PasswordEncoderUtil.java`
```java
return new SafeBcryptPasswordEncoder().encode(raw);
```
**问题描述**: 使用BCrypt加密，安全性较好，但实现细节需要审查

## 🔧 修复优先级建议

### 立即修复（高危）
1. **修改默认Token密钥** - 设置强随机密钥
2. **禁用或修改默认账号密码** - 强制首次登录修改
3. **升级到最新版本** - 修复已知SQL注入漏洞

### 近期修复（中危）
1. **完善XSS防护** - 加强CSP策略和输入过滤
2. **启用CSRF保护** - 添加适当的CSRF保护机制

### 长期改进（低危）
1. **定期更新依赖** - 保持依赖组件的最新版本
2. **安全审计** - 定期进行安全代码审查

## 📋 安全配置检查清单

- [ ] 修改默认Token密钥
- [ ] 修改默认管理员密码
- [ ] 启用认证机制
- [ ] 配置强CSP策略
- [ ] 启用HTTPS
- [ ] 定期更新依赖
- [ ] 配置防火墙规则
- [ ] 监控异常访问

## 🔗 参考链接

1. [Nacos官方安全文档](https://nacos.io/docs/latest/manual/admin/auth/)
2. [QVD-2023-6271漏洞详情](https://github.com/NET-Flowers/Nacos_check)
3. [CVE-2024-38816 Spring框架漏洞](https://nvd.nist.gov/vuln/detail/cve-2024-38816)
4. [Nacos SQL注入漏洞分析](https://dev.to/sharon_42e16b8da44dabde6d/nacos-admin-interface-rce-sqli-to-full-system-compromise-1b37)

---
**报告生成时间**: 2025年1月24日  
**建议复查周期**: 每季度