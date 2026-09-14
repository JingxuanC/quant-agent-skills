# MCP 接入配置

技能依赖 [mcphub](https://causal-memory.com) 的量化工具集，按**组**挂载。

> 把下面的 `YOUR_MCPHUB_KEY` 换成你自己的 key（形如 `mcphub_...`）。

## 应该挂哪些组

| 组 | 端点 | 工具数 | 建议 |
|---|---|---|---|
| **`data`** | `/hub/mcp/data` | 59 | ✅ **必挂** —— 技能用它自动取行情/资金/公告 |
| **`alpha`** | `/hub/mcp/alpha` | 31 | ✅ **必挂** —— 因子/因果/预测计算 |
| `memory` | `/hub/mcp/memory` | 17 | 可选 —— Agent 跨会话记忆 |
| `match` | `/hub/mcp/match` | 4 | 可选 —— fetch/时间工具 |

### 自建服务（不在 hub 上，需自己起）

下面两个服务是**独立仓库、独立部署**的，不通过 `causal-memory.com` 提供。
用到 `news-radar` / `event-propagation` 两个技能时必须自己跑起来：

| 服务 | 仓库 | 工具数 | 端口 | 供哪些技能使用 |
|---|---|---|---|---|
| **`news`** | [news-mcp](https://github.com/JingxuanC/news-mcp) | 8 | 50059 | `news-radar` |
| **`causal-chain`** | [causal-chain-mcp](https://github.com/JingxuanC/causal-chain-mcp) | 13 | 50058 | `event-propagation` |

两个都是零第三方依赖（纯标准库），一条命令起：

```bash
git clone https://github.com/JingxuanC/news-mcp && cd news-mcp && make serve
git clone https://github.com/JingxuanC/causal-chain-mcp && cd causal-chain-mcp && make serve
```

也可以 Docker：各自的 `docker-compose.yml` 直接 `docker compose up -d`。

> **只挂 alpha 会残缺**：alpha 组是纯计算，需要你自己提供 K 线；挂上 `data` 后 Agent 才能"用户只给代码，自己取数"。

---

## Claude Code / Claude Desktop

`~/Library/Application Support/Claude/claude_desktop_config.json`（macOS）：

```jsonc
{
  "mcpServers": {
    "mcphub-data": {
      "type": "streamable-http",
      "url": "https://causal-memory.com/hub/mcp/data",
      "headers": { "Authorization": "Bearer YOUR_MCPHUB_KEY" }
    },
    "mcphub-alpha": {
      "type": "streamable-http",
      "url": "https://causal-memory.com/hub/mcp/alpha",
      "headers": { "Authorization": "Bearer YOUR_MCPHUB_KEY" }
    },
    "mcphub-memory": {
      "type": "streamable-http",
      "url": "https://causal-memory.com/hub/mcp/memory",
      "headers": { "Authorization": "Bearer YOUR_MCPHUB_KEY" }
    }
  }
}
```

## Cursor

`.cursor/mcp.json`（项目级）或 `~/.cursor/mcp.json`（全局）：结构同上。

## Cline / Roo Code（VS Code）

`.vscode/cline_mcp_settings.json`：结构同上（`type` + `url` + `headers`）。

## DSH（DeepSeek Harness）

在 `~/.dsh/profiles/web/cordis.patch.yml` 的 `insert` 列表里加（**每个组一条**）：

```yaml
- insert:
    - id: mcp-mcphub-data
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        serverName: mcphub-data
        transport: streamable-http
        url: https://causal-memory.com/hub/mcp/data
        headers:
          Authorization: Bearer YOUR_MCPHUB_KEY
        toolCallTimeoutMs: 120000

    - id: mcp-mcphub-alpha
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        serverName: mcphub-alpha
        transport: streamable-http
        url: https://causal-memory.com/hub/mcp/alpha
        headers:
          Authorization: Bearer YOUR_MCPHUB_KEY
        toolCallTimeoutMs: 300000   # 因子回测/训练是长任务，务必放大
```

工具会以 `mcp__mcphub-data__<tool>` / `mcp__mcphub-alpha__<tool>` 出现。

### 自建服务的接入（DSH 示例）

`news` 与 `causal-chain` 是标准 MCP over HTTP，本地起好后同样按 streamable-http 挂：

```yaml
- insert:
    - id: mcp-news
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        serverName: news
        transport: streamable-http
        url: http://127.0.0.1:50059/mcp
        toolCallTimeoutMs: 120000     # run_pipeline 要现场抓多个外部源

    - id: mcp-causal-chain
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        serverName: causal-chain
        transport: streamable-http
        url: http://127.0.0.1:50058/mcp
        toolCallTimeoutMs: 120000
```

工具会以 `mcp__news__<tool>` / `mcp__causal-chain__<tool>` 出现。
（技能文档里为了简写，把它们记作 `news-*` / `causal-chain-*`。）

> 若你用的是 Claude Desktop / Cursor，把上面两条按同样的 `type` + `url` 结构写进对应配置即可 ——
> 唯一的区别是 `url` 指向 `http://127.0.0.1:50059/mcp`（本地）而不是 hub 域名。

## 其他客户端

通用三要素（每组重复一次）：

| 项 | 值 |
|---|---|
| transport | `streamable-http` |
| url | `https://causal-memory.com/hub/mcp/{data,alpha,memory,match}` |
| header | `Authorization: Bearer YOUR_MCPHUB_KEY` |

查看自己有哪些组：`GET https://causal-memory.com/hub/api/groups`（带同一个 Bearer）

---

## 验证接入

```
用 mcphub-data 的 get_a_hist 取 002371 最近 60 天日线，再用 mcphub-alpha 的 regime_detect 判断状态
```

能跑通 = data 与 alpha 都挂好了。

---

## 常见问题

| 现象 | 原因 / 解决 |
|---|---|
| 401 Unauthorized | key 缺失/错误/过期 —— 检查 `Authorization: Bearer ` 前缀与空格 |
| 工具列表为空 | 组名写错（只有 `data`/`alpha`/`memory`/`match` 四个组）或该 key 未授权此组 |
| 工具调用超时 | `factor_backtest` / `ml_train_rolling` 是长任务：放大 `toolCallTimeoutMs`，或改用异步 job_id + `job_status` 轮询 |
| 提示缺 klines 参数 | 说明只挂了 alpha（纯计算组）。挂上 `data` 组后 Agent 可用 `get_a_hist` 自动取数 |
| 工具名带前缀看不懂 | 前缀是命名空间：`astock-data-*` A股数据、`factor-miner-*` 因子、`causal-*` 因果、`kronos-*` K线预测、`news-*` 新闻链路（自建）、`causal-chain-*` 因果链推演（自建） |
| 技能提示工具不存在（`news-*` / `causal-chain-*`） | 这两个是自建服务，**不在 hub 上**。按上面的「自建服务」一节 clone + `make serve` 起本地，再挂进客户端 |

## 速率限制

服务端默认 **480 请求/分钟**。批量场景注意：

- 用批量工具（`event_study_batch` / `kronos-forecast_batch`）而不是循环单次调用
- 重任务提交后轮询 `job_status`，不要高频重试
