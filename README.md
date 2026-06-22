# APIs 本地代理服务器

从 Cloudflare Worker 适配而来的本地 API 代理服务器。支持多账号池、负载均衡、余量追踪和使用统计。

## 功能特性

### 核心能力
- **多账号池管理**：添加多个上游 API 账号，自动故障转移
- **智能负载均衡**：基于优先级和权重的路由分配
- **余量追踪**：按账号监控配额使用（次数/余额/积分）
- **使用统计**：追踪 API 调用次数、Token 消耗和成本
- **连接池复用**：复用 TCP/TLS 连接，提升性能
- **管理后台**：Web 界面管理账号和查看统计

### 兼容性
- 兼容 OpenAI API 格式（`/v1/chat/completions`）
- 支持流式响应（SSE）
- 支持图片生成 API（`/v1/images/generations`）
- 支持 Responses API 兼容层

## 快速开始

### 方式一：npm 安装（推荐）

```bash
# 安装
npm install apis-local-proxy

# 运行
npx apis-local-proxy
```

### 方式二：克隆源码

```bash
# 克隆仓库
git clone https://github.com/lijinzhao8/apis-local-proxy.git
cd apis-local-proxy

# 安装依赖（可选，无第三方依赖）
npm install

# 运行
npm start
```

### 方式三：直接运行

```bash
# 需要 Node.js >= 18
node APIs.js
```

启动后访问：
- 代理服务：`http://localhost:3000`
- 管理后台：`http://localhost:3000/admin`

## 使用方法

### 1. 添加账号

启动服务后，访问管理后台 `http://localhost:3000/admin`，点击「新增账号」：

| 字段 | 说明 | 示例 |
|------|------|------|
| 名称 | 账号备注名 | 阿里云百炼 |
| 接口地址 | 上游 API Base URL | `https://dashscope.aliyuncs.com/compatible-mode/v1` |
| API Key | 上游的 API 密钥 | `sk-xxx` |
| 支持的模型 | 每行一个模型名 | `qwen-max` |
| 优先级 | 数字越小越优先 | `1` |
| 权重 | 1-10，越大分配越多 | `5` |

### 2. 调用 API

将客户端的 API 地址改为 `http://localhost:3000/v1`，其他保持不变：

```bash
# 使用 curl 测试
curl http://localhost:3000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen-max",
    "messages": [{"role": "user", "content": "你好"}]
  }'

# 使用 Python
import openai
client = openai.OpenAI(base_url="http://localhost:3000/v1", api_key="any")
response = client.chat.completions.create(
    model="qwen-max",
    messages=[{"role": "user", "content": "你好"}]
)
```

### 3. 查看统计

管理后台提供三个统计页面：

| 页面 | 说明 |
|------|------|
| 账号管理 | 查看/编辑/测试账号，监控实时负载 |
| 使用记录 | 按时间查看每次 API 调用详情 |
| 累计统计 | 按天/月/总计查看 Token 消耗和成本 |

## 工作原理

```
客户端请求
    ↓
APIs 代理服务器（localhost:3000）
    ↓ 路由选择（优先级 + 权重）
    ↓
上游账号 A / B / C ...
    ↓
返回响应给客户端
```

### 路由策略

1. **优先级**：数字越小越优先（1 > 2 > 3）
2. **权重**：同一优先级内，按权重比例分配（权重 6 比权重 3 多分配 2 倍）
3. **故障转移**：账号失败自动尝试下一个
4. **并发控制**：可设置单账号最大并发数

### 余量系统

支持三种配额模式：

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| 按次数 | 限制总调用次数 | 免费额度 |
| 按余额 | 按金额扣减 | 付费账号 |
| 按积分 | 按积分消耗 | 积分制平台 |

## 配置说明

账号数据存储在 `apis-data/kv.json`，可通过管理后台编辑，也可直接修改文件：

```json
{
  "accounts": [
    {
      "id": "acc_123",
      "name": "阿里云百炼",
      "base_url": "https://dashscope.aliyuncs.com/compatible-mode/v1",
      "api_key": "sk-xxx",
      "models": ["qwen-max", "qwen-plus"],
      "enabled": true,
      "priority": 1,
      "weight": 5,
      "model_map": {}
    }
  ]
}
```

## 环境变量

| 变量 | 默认值 | 说明 |
|------|--------|------|
| `PORT` | `3000` | 服务监听端口 |

## 项目结构

```
apis-local-proxy/
├── APIs.js          # 主程序（单文件，无依赖）
├── package.json     # npm 配置
├── README.md        # 说明文档
├── .gitignore       # 忽略规则
└── apis-data/       # 运行时数据（已 gitignore）
    └── kv.json      # 账号和统计数据
```

## 常见问题

### Q: 需要安装依赖吗？
A: 不需要。这是一个单文件 Node.js 应用，只需要 Node.js >= 18 即可运行。

### Q: 支持哪些上游 API？
A: 支持所有兼容 OpenAI API 格式的服务，包括：
- OpenAI / Azure OpenAI
- 阿里云百炼
- DeepSeek
- MiniMax
- Moonshot
- 其他兼容 OpenAI 格式的 API

### Q: 可以同时添加多个相同平台的账号吗？
A: 可以。通过池模式（Pool Mode）可以将多个账号组合成一个逻辑账号，自动轮询和故障转移。

### Q: 如何部署到服务器？
A: 直接将 `APIs.js` 复制到服务器，用 `node APIs.js` 运行即可。建议配合 PM2 使用：

```bash
npm install -g pm2
pm2 start APIs.js --name apis-proxy
pm2 save
pm2 startup
```

## 许可证

MIT License
