| name | description |
|------|-------------|
| cloud-infrastructure-security | Use this skill when deploying to cloud platforms, configuring infrastructure, managing IAM policies, setting up logging/monitoring, or implementing CI/CD pipelines. Provides cloud security checklist aligned with best practices. |

# 云端与基础设施安全技能

此技能确保云端基础设施、CI/CD 管线和部署配置遵循安全最佳实践并符合业界标准。

## 何时启用

- 部署应用程序到云端平台（AWS、Vercel、Railway、Cloudflare）
- 配置 IAM 角色和权限
- 设置 CI/CD 管线
- 实现基础设施即代码（Terraform、CloudFormation）
- 配置日志和监控
- 在云端环境管理密钥
- 设置 CDN 和边缘安全
- 实现灾难恢复和备份策略

## 云端安全检查清单

### 1. IAM 与访问控制

#### 最小权限原则

```yaml
# 正确：最小权限
iam_role:
  permissions:
    - s3:GetObject  # 只有读取访问
    - s3:ListBucket
  resources:
    - arn:aws:s3:::my-bucket/*  # 只有特定 bucket

# 错误：过于广泛的权限
iam_role:
  permissions:
    - s3:*  # 所有 S3 动作
  resources:
    - "*"  # 所有资源
```

#### 多因素认证（MFA）

```bash
# 总是为 root/admin 帐户启用 MFA
aws iam enable-mfa-device \
  --user-name admin \
  --serial-number arn:aws:iam::123456789:mfa/admin \
  --authentication-code1 123456 \
  --authentication-code2 789012
```

#### 验证步骤

- [ ] 生产环境不使用 root 帐户
- [ ] 所有特权帐户启用 MFA
- [ ] 服务帐户使用角色，非长期凭证
- [ ] IAM 策略遵循最小权限
- [ ] 定期进行访问审查
- [ ] 未使用凭证已轮换或移除

### 2. 密钥管理

#### 云端密钥管理器

```typescript
// 正确：使用云端密钥管理器
import { SecretsManager } from '@aws-sdk/client-secrets-manager';

const client = new SecretsManager({ region: 'us-east-1' });
const secret = await client.getSecretValue({ SecretId: 'prod/api-key' });
const apiKey = JSON.parse(secret.SecretString).key;

// 错误：写死或只在环境变量
const apiKey = process.env.API_KEY; // 未轮换、未审计
```

#### 密钥轮换

```bash
# 为数据库凭证设置自动轮换
aws secretsmanager rotate-secret \
  --secret-id prod/db-password \
  --rotation-lambda-arn arn:aws:lambda:region:account:function:rotate \
  --rotation-rules AutomaticallyAfterDays=30
```

#### 验证步骤

- [ ] 所有密钥存储在云端密钥管理器（AWS Secrets Manager、Vercel Secrets）
- [ ] 数据库凭证启用自动轮换
- [ ] API 密钥至少每季轮换
- [ ] 代码、日志或错误消息中无密钥
- [ ] 密钥访问启用审计日志

### 3. 网络安全

#### VPC 和防火墙配置

```terraform
# 正确：限制的安全组
resource "aws_security_group" "app" {
  name = "app-sg"

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]  # 只有内部 VPC
  }

  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # 只有 HTTPS 输出
  }
}

# 错误：对互联网开放
resource "aws_security_group" "bad" {
  ingress {
    from_port   = 0
    to_port     = 65535
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # 所有端口、所有 IP！
  }
}
```

#### 验证步骤

- [ ] 数据库不可公开访问
- [ ] SSH/RDP 端口限制为 VPN/堡垒机
- [ ] 安全组遵循最小权限
- [ ] 网络 ACL 已配置
- [ ] VPC 流量日志已启用

### 4. 日志与监控

#### CloudWatch/日志配置

```typescript
// 正确：全面日志记录
import { CloudWatchLogsClient, CreateLogStreamCommand } from '@aws-sdk/client-cloudwatch-logs';

const logSecurityEvent = async (event: SecurityEvent) => {
  await cloudwatch.putLogEvents({
    logGroupName: '/aws/security/events',
    logStreamName: 'authentication',
    logEvents: [{
      timestamp: Date.now(),
      message: JSON.stringify({
        type: event.type,
        userId: event.userId,
        ip: event.ip,
        result: event.result,
        // 永远不要记录敏感数据
      })
    }]
  });
};
```

#### 验证步骤

- [ ] 所有服务启用 CloudWatch/日志记录
- [ ] 失败的认证尝试被记录
- [ ] 管理员操作被审计
- [ ] 日志保留已配置（合规需 90+ 天）
- [ ] 可疑活动设置警报
- [ ] 日志集中化且防篡改

### 5. CI/CD 管线安全

#### 安全管线配置

```yaml
# 正确：安全的 GitHub Actions 工作流程
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read  # 最小权限

    steps:
      - uses: actions/checkout@v4

      # 扫描密钥
      - name: Secret scanning
        uses: trufflesecurity/trufflehog@main

      # 依赖审计
      - name: Audit dependencies
        run: npm audit --audit-level=high

      # 使用 OIDC，非长期 tokens
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/GitHubActionsRole
          aws-region: us-east-1
```

#### 供应链安全

```json
// package.json - 使用 lock 文件和完整性检查
{
  "scripts": {
    "install": "npm ci",  // 使用 ci 以获得可重现构建
    "audit": "npm audit --audit-level=moderate",
    "check": "npm outdated"
  }
}
```

#### 验证步骤

- [ ] 使用 OIDC 而非长期凭证
- [ ] 管线中的密钥扫描
- [ ] 依赖漏洞扫描
- [ ] 容器镜像扫描（如适用）
- [ ] 强制执行分支保护规则
- [ ] 合并前需要代码审查
- [ ] 强制执行签名 commits

### 6. Cloudflare 与 CDN 安全

#### Cloudflare 安全配置

```typescript
// 正确：带安全标头的 Cloudflare Workers
export default {
  async fetch(request: Request): Promise<Response> {
    const response = await fetch(request);

    // 新增安全标头
    const headers = new Headers(response.headers);
    headers.set('X-Frame-Options', 'DENY');
    headers.set('X-Content-Type-Options', 'nosniff');
    headers.set('Referrer-Policy', 'strict-origin-when-cross-origin');
    headers.set('Permissions-Policy', 'geolocation=(), microphone=()');

    return new Response(response.body, {
      status: response.status,
      headers
    });
  }
};
```

#### WAF 规则

```bash
# 启用 Cloudflare WAF 管理规则
# - OWASP 核心规则集
# - Cloudflare 管理规则集
# - 速率限制规则
# - Bot 保护
```

#### 验证步骤

- [ ] WAF 启用 OWASP 规则
- [ ] 速率限制已配置
- [ ] Bot 保护启用
- [ ] DDoS 保护启用
- [ ] 安全标头已配置
- [ ] SSL/TLS 严格模式启用

### 7. 备份与灾难恢复

#### 自动备份

```terraform
# 正确：自动 RDS 备份
resource "aws_db_instance" "main" {
  allocated_storage     = 20
  engine               = "postgres"

  backup_retention_period = 30  # 30 天保留
  backup_window          = "03:00-04:00"
  maintenance_window     = "mon:04:00-mon:05:00"

  enabled_cloudwatch_logs_exports = ["postgresql"]

  deletion_protection = true  # 防止意外删除
}
```

#### 验证步骤

- [ ] 已设置自动每日备份
- [ ] 备份保留符合合规要求
- [ ] 已启用时间点恢复
- [ ] 每季执行备份测试
- [ ] 灾难恢复计划已记录
- [ ] RPO 和 RTO 已定义并测试

## 部署前云端安全检查清单

任何生产云端部署前：

- [ ] **IAM**：不使用 root 帐户、启用 MFA、最小权限策略
- [ ] **密钥**：所有密钥在云端密钥管理器并有轮换
- [ ] **网络**：安全组受限、无公开数据库
- [ ] **日志**：CloudWatch/日志启用并有保留
- [ ] **监控**：异常设置警报
- [ ] **CI/CD**：OIDC 认证、密钥扫描、依赖审计
- [ ] **CDN/WAF**：Cloudflare WAF 启用 OWASP 规则
- [ ] **加密**：数据静态和传输中加密
- [ ] **备份**：自动备份并测试恢复
- [ ] **合规**：符合 GDPR/HIPAA 要求（如适用）
- [ ] **文档**：基础设施已记录、建立操作手册
- [ ] **事件响应**：安全事件计划就位

## 常见云端安全错误配置

### S3 Bucket 暴露

```bash
# 错误：公开 bucket
aws s3api put-bucket-acl --bucket my-bucket --acl public-read

# 正确：私有 bucket 并有特定访问
aws s3api put-bucket-acl --bucket my-bucket --acl private
aws s3api put-bucket-policy --bucket my-bucket --policy file://policy.json
```

### RDS 公开访问

```terraform
# 错误
resource "aws_db_instance" "bad" {
  publicly_accessible = true  # 绝不这样做！
}

# 正确
resource "aws_db_instance" "good" {
  publicly_accessible = false
  vpc_security_group_ids = [aws_security_group.db.id]
}
```

## 资源

- [AWS Security Best Practices](https://aws.amazon.com/security/best-practices/)
- [CIS AWS Foundations Benchmark](https://www.cisecurity.org/benchmark/amazon_web_services)
- [Cloudflare Security Documentation](https://developers.cloudflare.com/security/)
- [OWASP Cloud Security](https://owasp.org/www-project-cloud-security/)
- [Terraform Security Best Practices](https://www.terraform.io/docs/cloud/guides/recommended-practices/)

**记住**：云端错误配置是数据泄露的主要原因。单一暴露的 S3 bucket 或过于宽泛的 IAM 策略可能危及你的整个基础设施。总是遵循最小权限原则和深度防御。
