# 腾讯云服务器部署方案 (Ubuntu 24.04 LTS x86_64)

本文档提供在腾讯云服务器（CVM 或轻量应用服务器 Lighthouse，系统版本 Ubuntu 24.04.4 LTS，架构 x86_64 / amd64）上搭建 `antigravity-claude-proxy` 的标准化生产部署方案。

- **项目仓库**：[https://github.com/xuzepei/agy-cc-proxy.git](https://github.com/xuzepei/agy-cc-proxy.git)
- **服务器公网 IP**：`43.173.115.129`（腾讯云美西/硅谷节点，原生直连 Google，无需前置代理）
- **目标环境**：Ubuntu 24.04.4 LTS (x86_64 / amd64)
- **推荐端口**：`8088` (Node.js 后端服务) / `80` & `443` (Nginx 反向代理)

---

## 1. 端口选型规范与架构策略

服务默认使用作者预设的 `12345` 端口，生产部署推荐将其规范指定为 **`8088`**。

### 1.1 为什么推荐指定为 8088 端口？
- **避开系统特权端口**：Linux 下 1024 以下端口（如 `80`、`443`）需要 root 特权才能直接监听，使用非特权高位端口 `8088` 可安全以普通运维用户权限运行。
- **避开项目自动化测试冲突**：本项目内部的自动化测试套件（`tests/*.cjs` 及 `tests/run-all.cjs`）固定监听在 `:8080` 端口。若生产服务占用 8080，将导致服务器本地无法执行测试并报 `EADDRINUSE` 错误。
- **避开内置 OAuth 回调与中间件冲突**：
  - 本项目 OAuth 回调监听在 `51121`~`51126`；
  - 本项目内部认证服务监听在 `9092`；
  - 常见开发与数据库端口：`3000` (前端/Node)、`3306` (MySQL)、`6379` (Redis)、`8888` (宝塔/Jupyter)。
- **规范标准**：`8088` 是行业公认的 HTTP-Alt 服务端口，符合网络工程规范且便于记忆。

### 1.2 地域网络要求
- **方案 A：海外地域（强烈推荐）**：腾讯云中国香港、新加坡、东京、硅谷等节点，直接直连 Google API 端点，低延迟且无需前置网络代理。
- **方案 B：中国大陆境内地域（必须配置前置代理）**：广州、上海、北京等节点无法直连 Google API，必须在服务器部署出站代理并通过 `HTTPS_PROXY` 环境变量路由流量。

---

## 2. 基础运行环境准备

在腾讯云 Ubuntu 24.04 服务器上执行以下命令安装 Node.js 20 LTS 运行环境、编译工具链及 PM2 进程管理器：

```bash
# 1. 更新系统包索引并安装原生模块编译工具（better-sqlite3 必需）
sudo apt-get update && sudo apt-get install -y curl git build-essential python3 ufw lsof

# 2. 安装 Node.js 20.x LTS (NodeSource 官方仓库)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs

# 3. 校验环境版本
node -v   # 要求 >= 18.0.0 (推荐 v20.x)
npm -v

# 4. 全局安装 PM2 进程管理器
sudo npm install -g pm2
```

---

## 3. 代码部署与构建

将代码部署至服务器应用目录并完成编译：

```bash
# 1. 创建应用目录并设置权限
sudo mkdir -p /opt/agy-cc-proxy
sudo chown -R $USER:$USER /opt/agy-cc-proxy
cd /opt/agy-cc-proxy

# 2. 同步项目代码至服务器
# 方式 A：通过 Git 直接克隆仓库（推荐）
git clone https://github.com/xuzepei/agy-cc-proxy.git .
# （若已配置 SSH Key 也可使用：git clone git@github.com:xuzepei/agy-cc-proxy.git .）

# 方式 B：从本地通过 rsync 同步（排除本地 node_modules 与测试日志）
# rsync -avz --exclude 'node_modules' --exclude '.git' --exclude 'logs' ./ ubuntu@43.173.115.129:/opt/agy-cc-proxy/

# 3. 安装依赖（自动触发 Tailwind CSS 构建与 better-sqlite3 本地编译）
npm install
```

---

## 4. 进程与网络配置 (指定 8088 端口)

修改项目根目录的 `ecosystem.config.cjs`，显式指定生产端口为 **`8088`**（如需前置代理亦在此注入）：

```javascript
module.exports = {
  apps: [{
    name: 'agy-cc-proxy',
    script: 'src/index.js',
    args: '--strategy=hybrid',
    interpreter: 'node',
    restart_delay: 3000,
    max_restarts: 10,
    env: {
      NODE_ENV: 'production',
      PORT: '8088', // 显式指定生产端口为 8088
      // 大陆服务器按需解开以下出站代理配置：
      // HTTP_PROXY: 'http://127.0.0.1:7890',
      // HTTPS_PROXY: 'http://127.0.0.1:7890'
    }
  }]
};
```

---

## 5. 账号认证与凭证配置

### 方式 1：从开发机同步已有凭证（推荐，最省时）
该方式先在有浏览器与界面的**本地个人电脑（Mac/Windows 开发机）**上完成 Google OAuth 授权，再将自动生成的凭证传输给云服务器。

#### 步骤 1.1：在本地电脑获取并生成凭证
在本地开发机打开终端，进入当前项目目录并运行：

```bash
# 1. 确保本地依赖已安装
npm install

# 2. 启动本地 OAuth 授权（自动调起默认浏览器）
npm run accounts:add
```
- 命令会自动在本地启动临时回调监听（51121 端口）并自动弹出浏览器跳转至 Google 登录页面。
- 选择并登录你的 Google 账号，点击授权。
- 授权成功后，项目会在本地电脑的家目录下自动生成持久化凭证文件：
  - `~/.config/antigravity-proxy/accounts.json`（保存 OAuth `refreshToken`、邮箱与项目配额信息）
  - `~/.config/antigravity-proxy/config.json`（保存 API Key 与系统默认配置）

#### 步骤 1.2：将本地凭证一键同步至腾讯云服务器
在本地开发机终端中执行以下命令，将刚生成的凭证拷贝至云服务器对应目录：

```bash
# 1. 在云服务器上预先创建好配置目录
ssh ubuntu@43.173.115.129 "mkdir -p ~/.config/antigravity-proxy"

# 2. 从本地开发机传输凭证（替换 ubuntu 为实际登录用户）
scp ~/.config/antigravity-proxy/accounts.json ubuntu@43.173.115.129:~/.config/antigravity-proxy/
scp ~/.config/antigravity-proxy/config.json ubuntu@43.173.115.129:~/.config/antigravity-proxy/
```

### 方式 2：服务器无浏览器（Headless）OAuth 授权
若在云服务器上全新添加账号：

```bash
cd /opt/agy-cc-proxy
npm run accounts:add -- --no-browser
```
1. 终端会输出形如 `https://accounts.google.com/o/oauth2/v2/auth?...` 的授权链接。
2. 在任意可访问的外网本地浏览器中打开该链接，登录 Google 账号完成授权。
3. 授权后浏览器重定向至 `http://localhost:51121/...`，复制该完整重定向 URL 或其携带的 `code` 贴回服务器终端完成绑定。

检查账号凭证有效性：
```bash
npm run accounts:list
npm run accounts:verify
```

---

## 6. 服务安全与核心参数加固

编辑云服务器配置文件 `~/.config/antigravity-proxy/config.json`，配置固定的服务端口、访问密钥与管理密码：

```json
{
  "port": 8088,
  "apiKey": "sk-agy-YOUR_SECURE_API_KEY_STRING",
  "webuiPassword": "YOUR_STRONG_DASHBOARD_PASSWORD",
  "debug": false,
  "devMode": false,
  "maxRetries": 5,
  "thinkingAsText": true,
  "accountSelection": {
    "strategy": "hybrid"
  }
}
```

> **安全注意**：
> - 严禁在公网暴露无 `apiKey` 的服务；若 `apiKey` 为空，任何扫描到端口的实体均可消耗账号配额。
> - `webuiPassword` 设置后，首次使用 WebUI 会自动转换为安全 scrypt 哈希存储。

---

## 7. 启动服务与开机自启配置

使用自带的生产管理脚本启动（脚本支持通过 `PORT` 环境变量自适应）：

```bash
cd /opt/agy-cc-proxy

# 赋予执行权限并以 8088 端口启动（内置端口防冲突与健康检查）
chmod +x start.sh stop.sh
PORT=8088 ./start.sh

# 配置开机自启
pm2 startup
# 根据 pm2 startup 的提示执行生成的 sudo env PATH=... 命令
pm2 save

# 常用运维命令
pm2 status                  # 查看运行状态
pm2 logs agy-cc-proxy       # 查看实时日志
pm2 restart agy-cc-proxy    # 重启服务
```

---

## 8. Nginx 反向代理与 SSL 配置（生产推荐）

避免直接将 Node.js 8088 端口裸露给公网，建议使用 Nginx 反代并关闭缓冲保证流式（SSE）响应流畅。

### 8.1 安装 Nginx
```bash
sudo apt-get install -y nginx
```

### 8.2 创建 Nginx 虚拟主机配置
新建 `/etc/nginx/sites-available/agy-proxy`：

```nginx
server {
    listen 80;
    server_name 43.173.115.129; # 填写服务器公网 IP，或替换为你的真实解析域名

    client_max_body_size 50M;

    location / {
        proxy_pass http://127.0.0.1:8088; # 反代至本地 8088 端口
        proxy_http_version 1.1;

        # 核心：必须禁用代理缓冲以支持 Server-Sent Events (SSE) 流式传输
        proxy_buffering off;
        proxy_cache off;
        proxy_set_header X-Accel-Buffering no;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # 长连接超时时间控制（Thinking 模式思考时间较长）
        proxy_read_timeout 600s;
        proxy_send_timeout 600s;
    }
}
```

启用站点并重载：
```bash
sudo ln -sf /etc/nginx/sites-available/agy-proxy /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

### 8.3 配置 HTTPS (Certbot)
```bash
sudo apt-get install -y certbot python3-certbot-nginx
sudo certbot --nginx -d proxy.yourdomain.com
```

---

## 9. 腾讯云安全组规则

前往 **腾讯云控制台 -> 云服务器/轻量服务器 -> 实例详情 -> 安全组**：

| 方向 | 协议 | 端口 | 来源 | 说明 |
| :--- | :--- | :--- | :--- | :--- |
| 入站 | TCP | `22` | 限制指定 IP 或全局 | SSH 管理运维 |
| 入站 | TCP | `80` | `0.0.0.0/0` | HTTP（Nginx 反代） |
| 入站 | TCP | `443` | `0.0.0.0/0` | HTTPS（SSL 加密） |
| 入站 | TCP | `8088` | 建议仅限 127.0.0.1 或内网 | 若直接裸跑不用 Nginx，需开放此端口 |

---

## 10. 客户端对接验证

### 10.1 Claude Code CLI 接入
在客户端电脑终端配置环境变量并测试：

```bash
# 方式 A：直接通过后端 8088 端口直连（需安全组开放 8088）
export ANTHROPIC_BASE_URL="http://43.173.115.129:8088"
export ANTHROPIC_AUTH_TOKEN="sk-agy-YOUR_SECURE_API_KEY_STRING"

# 方式 B：通过 Nginx 80 端口代理访问（或绑定解析后的域名）
# export ANTHROPIC_BASE_URL="http://43.173.115.129"
# export ANTHROPIC_AUTH_TOKEN="sk-agy-YOUR_SECURE_API_KEY_STRING"

# 启动 Claude Code CLI
claude
```

### 10.2 连通性测试命令
```bash
# 1. 服务健康检查测试（在客户端电脑或服务器本地均可测试）
curl http://43.173.115.129:8088/health

# 2. 消息生成测试（验证 Claude 转译链路）
curl -X POST http://43.173.115.129:8088/v1/messages \
  -H "x-api-key: sk-agy-YOUR_SECURE_API_KEY_STRING" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{
    "model": "claude-3-5-sonnet-20241022",
    "max_tokens": 100,
    "messages": [{"role": "user", "content": "ping"}]
  }'
```

---

## 11. 常见故障与维护

1. **`better-sqlite3` 原生模块编译报错**
   - 原因：缺少 C++ 编译环境或 Node.js 版本变化。
   - 解决：运行 `sudo apt-get install -y build-essential python3` 后执行 `npm rebuild better-sqlite3`。
2. **请求返回 403 / 无法连接 Google 认证**
   - 原因：服务器处于大陆境内无法触达 Google，或账号授权过期。
   - 解决：检查 `HTTPS_PROXY` 环境变量连通性，或运行 `npm run accounts:verify` 刷新凭证。
3. **SSE 流式输出卡顿（整段输出而非打字机效果）**
   - 原因：Nginx 开启了响应缓冲（`proxy_buffering on`）。
   - 解决：检查 Nginx 配置中的 `proxy_buffering off;` 与 `proxy_set_header X-Accel-Buffering no;`。
