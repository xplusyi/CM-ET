# Cloudflare 多 Worker 共享 GitHub 仓库的 KV 绑定失效解决方法

**标签**：#Cloudflare #Worker #KV #自动化部署 

### 📌 问题场景
多个 Cloudflare Worker 连接到同一个 GitHub 仓库进行自动部署。每次代码更新触发重新部署后，原本在网页控制台手动绑定的 KV 空间会失效/被清空，需要重新手动绑定。

### 🔍 核心原因
Cloudflare 自动部署以代码库根目录的 `wrangler.toml` 为“事实来源”。如果该文件未区分环境并写入 KV ID，每次部署都会用默认配置覆盖掉网页端的 UI 设置。

---

### ✅ 解决方案：使用 Wrangler 环境 (`[env]`) 隔离配置

#### 第一步：修改代码库的 `wrangler.toml`
利用 `[env]` 为每个 Worker 定义独立的环境名称和对应的 KV ID（注意：是 ID，不是名称）。代码中依然可以统一使用 `env.KV` 调用。

```toml
# 基础配置 (所有环境共享)
main = "_worker.js"
compatibility_date = "2025-11-04"
keep_vars = true

# --- 环境 1: cm1 ---
[env.cm1]
name = "cm1" # 对应 Cloudflare 上的 Worker 名称
kv_namespaces = [
  { binding = "KV", id = "b1cd1e9074cd43f989b0ee3b4f913d38" }
]

# --- 环境 2: cm2 ---
[env.cm2]
name = "cm2"
kv_namespaces = [
  { binding = "KV", id = "2502cfe9908a4e40bb24845dd901e10f" }
]

# --- 环境 3: cm3 ---
[env.cm3]
name = "cm3"
kv_namespaces = [
  { binding = "KV", id = "bfe176baecfb4708a6f0ddef0941dca5" }
]

# --- 环境 4: cm4 ---
[env.cm4]
name = "cm4"
kv_namespaces = [
  { binding = "KV", id = "f7081afb04744259bbd4219803f39a44" }
]
```

#### 第二步：修改 Cloudflare 控制台的部署命令 (Deploy Command)
需要让 Cloudflare 在拉取代码后，知道当前 Worker 应该使用哪个 `[env]` 配置。

1. 登录 Cloudflare 控制台 -> **Workers & Pages** -> 点击对应的 Worker（例如 `cm1`）。
2. 进入 **Settings（设置）** -> 左侧选择 **Builds（构建）**。
3. 找到 **Build configurations（构建配置）**，点击 **Edit（编辑）**。
4. **注意区分**：保持 *Build command* 留空或默认，只修改 **Deploy command（部署命令）**。
5. 为每个 Worker 填入对应环境的部署命令：
   * cm1 修改为：`npx wrangler deploy --env cm1`
   * cm2 修改为：`npx wrangler deploy --env cm2`
   * cm3 修改为：`npx wrangler deploy --env cm3`
   * cm4 修改为：`npx wrangler deploy --env cm4`
6. 点击 **Save and Deploy** 或者是 **Update** 保存。

### 💡 总结
**核心原则**：不要在网页端手动绑定 KV。将真实的 KV ID 写进 `wrangler.toml` 的对应 `[env]` 中，并通过修改 **Deploy Command** 加上 `--env <环境名>` 参数，实现自动且精准的绑定。
