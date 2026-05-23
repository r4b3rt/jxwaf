# JXWAF安全模型服务

## 目录

- [JXWAF 安全模型服务](#jxwaf安全模型服务)
  - [介绍](#介绍)
  - [快速上手](#快速上手)
  - [私有化部署](#私有化部署)
  - [Web 攻击检测 API](#web-攻击检测-api)
- [JXWAF 标准版](#jxwaf)
  - [介绍](#介绍-1)
  - [功能](#功能)
  - [部署](#部署)
  - [配置说明](#配置说明-1)
  - [防护效果对比](#防护效果对比)
  - [微信公众号](#微信公众号)
  - [贡献者](#贡献者)
  - [BUG & 需求](#bug--需求)

## 介绍

基于多维稀疏注意力机制的大模型语义缓存系统，通过在线蒸馏获得大模型的安全检测能力，实现**高并发、低成本、低幻觉**的安全模型服务。目前已上线 **Web 安全检测模型**，可精准识别 SQL 注入、XSS、命令执行等 Web 攻击，后续将陆续推出更多安全模型。

## 快速上手

### 1. 访问控制台
浏览器打开 `https://model.jxwaf.com`。首次使用需注册账号，**强烈建议开启 OTP 双因素认证**以增强账户安全。

### 2. 配置 AI 模型
进入「AI 模型配置」页面：
- 选择模型（如 `deepseek-v4-flash`）
- 填入 DeepSeek API Key，支持一键验证有效性
- 按需开关「思考模式」：开启时 AI 执行深度推理链，准确率更高；关闭时速度更快、成本更低

### 3. 生成 API Key
进入「API 密钥管理」页面创建 Key，输入标识名后系统生成唯一密钥。

### 4. 第一次调用
```bash
curl -X POST https://model.jxwaf.com/jxwaf_model/web_attack_check \
  -H "Content-Type: application/json" \
  -H "jxwaf-model-auth: <你的API Key>" \
  -d "id=1' OR '1'='1"
```
首次提交会返回 `unknown`，等待大约 5 秒后重试即可获取分析结果（开启思考模式则 1 分钟后能有结果）。

---

## 私有化部署

### 环境要求
- **操作系统**：Debian 12.x
- **端口**：确保 80、443、3306 端口未被占用

### 部署步骤

**1. 安装 Docker**
```bash
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
```

**2. 获取部署文件**
```bash
git clone https://github.com/jx-sec/jxwaf.git
```

**3. 启动服务**
```bash
cd jxwaf/jxwaf_model_platform
docker compose up -d
```

### 配置说明

项目仓库已包含 `docker-compose.yml`，按需修改以下核心配置：

| 配置项 | 说明 |
|--------|------|
| `AUTH_CODE` | 授权码，测试用授权码过期时间 2026-12-31 |
| `MYSQL_PASSWORD` | MySQL 密码，`jxwaf_model_platform` 和 `mysql` 服务**两处必须一致**，建议修改为强密码 |
| `PLATFORMS` → `deepseek.api_key` | DeepSeek API Key，用于调用 AI 分析，**仅对演示页面生效** |
| `PLATFORMS` → `deepseek.thinking_type` | `enabled` 开启深度推理（更准）；`disabled` 关闭（更快、成本更低），**仅对演示页面生效** |
| `IP_RATE_LIMIT` | 单 IP 24 小时内最大请求次数，超过则临时封禁，默认 `1000`，**仅对演示页面生效** |
| `ANALYSIS_COUNT` | 多轮投票次数，默认 `3` 次独立分析 |
| `ENABLE_TLS` | 是否启用 HTTPS，开启后需挂载 SSL 证书 |

> 修改配置后执行 `docker compose down && docker compose up -d` 重启生效。

---

## Web 攻击检测 API

### 基本信息
- **方法**：`POST`
- **地址**：`/jxwaf_model/web_attack_check`
- **认证**：请求头 `jxwaf-model-auth` 传入 API Key

### 请求示例
```bash
curl -X POST https://model.jxwaf.com/jxwaf_model/web_attack_check \
  -H "Content-Type: application/json" \
  -H "jxwaf-model-auth: <你的API Key>" \
  -d "id=1' OR '1'='1"
```

### 响应示例

**判定为攻击：**
```json
{
  "status": "ok",
  "result": "attack",
  "token": "a1b2c3d4e5f6...",
  "ai_model": "deepseek-v4-flash",
  "attack_type": ["sql", "rce"]
}
```

**判定为安全：**
```json
{
  "status": "ok",
  "result": "safe",
  "token": "a1b2c3d4e5f6...",
  "ai_model": ""
}
```

**待分析（首次提交）：**
```json
{
  "status": "ok",
  "result": "unknown",
  "token": "a1b2c3d4e5f6...",
  "ai_model": ""
}
```

**认证失败：**
```json
{
  "status": "error",
  "result": "auth_fail"
}
```

### 响应字段说明

| 字段 | 类型 | 描述 |
|------|------|------|
| `status` | string | `ok` – 请求成功；`error` – 请求失败 |
| `result` | string | `safe` – 安全；`attack` – 攻击；`unknown` – 待分析（后台异步处理中）；`auth_fail` – 认证失败 |
| `token` | string | 请求内容的唯一哈希标识，可用于日志追踪或二次查询 |
| `ai_model` | string | 返回 `attack`/`safe` 时，展示所用 AI 模型名称；否则为空 |
| `attack_type` | array | 仅在 `result=attack` 时返回，列出命中的攻击类型 |

### 攻击类型

平台当前支持 7 类 Web 攻击：

| 标识 | 名称 | 典型特征 |
|------|------|----------|
| `sql` | SQL 注入 | `' OR '1'='1`、`UNION SELECT` |
| `xss` | 跨站脚本 | `<script>alert(1)</script>` |
| `rce` | 远程命令执行 | `; ls`、`$(whoami)` |
| `code_exec` | 任意代码执行 | SSTI、OGNL/SpEL 表达式 |
| `path_traversal` | 路径遍历 | `../../etc/passwd` |
| `xxe` | XML 外部实体注入 | 利用 XML 解析器读取文件或发起 SSRF |
| `other` | 其他攻击 | SSRF、NoSQL 注入、CRLF 注入等 |

> **最佳实践**：收到 `unknown` 后，建议客户端间隔 5 秒重试 1 次。重试不产生额外 AI 调用费用。

---

# JXWAF

## 介绍

JXWAF6 标准版是一款基于 AI 大模型的 Web 应用防火墙。

## 功能

- 数据统计
- 攻击事件
- 攻击日志
- 网站防护
  - 网站接入
  - 证书管理
- AI 防护配置
  - Web 安全防护
  - AI 分析记录
- 防护配置
  - Web 防护规则
  - 流量防护规则
  - IP 区域封禁
  - 白名单规则
- 防护组件
- 节点状态

## 部署

### 环境要求

- 服务器系统：Debian 12.x
- 服务器最低配置：4 核 8G

### 一键部署

服务器 IP 地址：
- 公网地址：47.120.63.196
- 内网地址：172.29.198.241

```bash
# 1. 安装 Docker
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun

# 2. 克隆仓库
git clone https://github.com/jx-sec/jxwaf.git

# 3. 启动容器
cd jxwaf/standard
docker compose up -d
```

WAF 控制台地址：http://47.120.63.196:8000

## 配置说明

### Docker Compose 文件配置

- **JXWAF_MODEL_QUERY**  
  是否开启 JXWAF 大模型语义缓存服务并加入群体免疫网络，值为 `true` 或 `false`。  
  - **大模型语义缓存服务**：遇到未知请求时，会先进行缓存查询，若命中则无需通过大模型进行查询，可极大节省大模型的使用成本。
  - **群体免疫网络**：当其他 WAF 检测到全新的攻击 POC 时，将通过群体免疫网络将模型参数同步到本地 WAF，实时获取最新的检测能力。

- **AI_BACKUP_WAF_URL**  
  当大模型服务因各种原因不可用时，可通过该配置获取其他 WAF 的检测能力。由于需要转发请求到目标 WAF，可能存在数据泄露风险，请谨慎配置。

### 控制台 AI 防护配置

#### 防护模式说明

与传统 WAF 非黑即白的检测模式不同，AI WAF 采用 **非白即黑** 的检测模式。

- **在线学习**  
  根据线上业务流量训练本地模型，不进行任何处置。
  
- **在线防护 - 业务优先**  
  遇到已知攻击流量将进行拦截；遇到未知流量会先放行，待 AI 分析出结果后，同步更新本地模型进行处理。

- **在线防护 - 安全优先**  
  遇到已知攻击流量将进行拦截；遇到未知流量会先拦截，待 AI 分析出结果后，同步更新本地模型进行处理。

- **离线防护**  
  对已知攻击流量和未知流量均进行拦截，本地模型不再更新。

## 防护效果对比

**测试方式**：  
```bash
docker run --rm --net=host ccr.ccs.tencentyun.com/jxwaf/blazehttp:latest /app/blazehttp -t http://172.30.42.104/xxx
```
使用长亭 blazehttp 项目提供的样本集，效果如下：

### JXWAF6 标准版 - DeepSeek
- 总样本数量：33877，成功：33877，错误：0
- 检出率：41.03%（恶意样本总数：658，正确拦截：270，漏报放行：388）
- 误报率：0.14%（正常样本总数：33219，正确放行：33172，误报拦截：47）
- 准确率：98.72%（（正确拦截 + 正确放行）/ 样本总数）
- 平均耗时：28.19 毫秒

### JXWAF5 - 语义分析引擎
- 总样本数量：33877，成功：33877，错误：0
- 检出率：26.90%（恶意样本总数：658，正确拦截：177，漏报放行：481）
- 误报率：0.20%（正常样本总数：33219，正确放行：33153，误报拦截：66）
- 准确率：98.39%
- 平均耗时：43.68 毫秒

### 某云 WAF - 默认配置
- 总样本数量：33877，成功：33877，错误：0
- 检出率：40.12%（恶意样本总数：658，正确拦截：264，漏报放行：394）
- 误报率：0.23%（正常样本总数：33219，正确放行：33143，误报拦截：76）
- 准确率：98.61%
- 平均耗时：43.36 毫秒

### 某 WAF - 官方 DEMO 配置
- 总样本数量：33877，成功：33877，错误：0
- 检出率：44.38%（恶意样本总数：658，正确拦截：292，漏报放行：366）
- 误报率：0.19%（正常样本总数：33219，正确放行：33155，误报拦截：64）
- 准确率：98.73%
- 平均耗时：33.02 毫秒

**结论**：JXWAF6 标准版相比 JXWAF5 检测效果提升明显，达到商业 WAF 的检测水平。

## 微信公众号

欢迎关注微信公众号，后续更新与技术分享将在公众号发布。

<kbd><img src="img/wx_code.jpeg"></kbd>

## 贡献者

- [chenjc](https://github.com/jx-sec)
- [jiongrizi](https://github.com/jiongrizi)
- [thankfly](https://github.com/thankfly)

## BUG & 需求

- 微信：574604532（添加请备注 jxwaf）
