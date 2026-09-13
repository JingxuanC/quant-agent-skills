# MCP 接入配置

技能依赖 [mcphub](https://causal-memory.com) 提供的量化工具集（35 个工具，streamable-http + Bearer 认证）。

> 把下面的 `YOUR_MCPHUB_KEY` 换成你自己的 key（形如 `mcphub_...`）。

---

## Claude Code / Claude Desktop

`~/Library/Application Support/Claude/claude_desktop_config.json`（macOS）：

```jsonc
{
  "mcpServers": {
    "mcphub": {
      "type": "streamable-http",
      "url": "https://causal-memory.com/hub/mcp/alpha",
      "headers": { "Authorization": "Bearer YOUR_MCPHUB_KEY" }
    }
  }
}
```

## Cursor

`.cursor/mcp.json`（项目级）或 `~/.cursor/mcp.json`（全局）：

```jsonc
{
  "mcpServers": {
    "mcphub": {
      "url": "https://causal-memory.com/hub/mcp/alpha",
      "headers": { "Authorization": "Bearer YOUR_MCPHUB_KEY" }
    }
  }
}
```

## Cline / Roo Code（VS Code）

`.vscode/cline_mcp_settings.json`：结构同上（`type` + `url` + `headers`）。

## DSH（DeepSeek Harness）

在 `~/.dsh/profiles/web/cordis.patch.yml` 的 `insert` 列表里加：

```yaml
- insert:
    - id: mcp-mcphub
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        serverName: mcphub
        transport: streamable-http
        url: https://causal-memory.com/hub/mcp/alpha
        headers:
          Authorization: Bearer YOUR_MCPHUB_KEY
        toolCallTimeoutMs: 300000   # 因子回测/训练是长任务，务必放大
```

工具会以 `mcp__mcphub__<tool>` 出现（如 `mcp__mcphub__factor-miner-ml_predict`）。

## 其他客户端

任何支持 **MCP streamable-http** 的客户端，通用三要素：

| 项 | 值 |
|---|---|
| transport | `streamable-http` |
| url | `https://causal-memory.com/hub/mcp/alpha` |
| header | `Authorization: Bearer YOUR_MCPHUB_KEY` |

---

## 验证接入

接入后让 Agent 做一次最小调用（消耗极低）：

```
列出 mcphub 提供的工具，并告诉我 regime_detect 需要什么参数
```

或直接问一个轻量问题：

```
用 regime_detect 判断一下沪深300最近 60 天是牛是熊
```

---

## 常见问题

| 现象 | 原因 / 解决 |
|---|---|
| 401 Unauthorized | key 缺失/错误/过期 —— 检查 `Authorization: Bearer ` 前缀和空格 |
| 工具调用超时 | `factor_backtest` / `ml_train_rolling` 是长任务：放大 `toolCallTimeoutMs`，或改用异步 job_id + `job_status` 轮询 |
| 提示缺 klines 参数 | 这些工具是**纯计算**，不代为取行情 —— 先提供 K 线数据，或让 Agent 从你的数据源取 |
| 工具名带前缀看不懂 | 前缀是命名空间：`factor-miner-*` 因子挖掘、`causal-*` 因果推断、`kronos-*` K 线预测 |

## 速率限制

服务端默认 **480 请求/分钟**。批量场景（`batch-screening`）注意：

- 用批量工具（`event_study_batch` / `kronos-forecast_batch`）而不是循环单次调用
- 重任务提交后轮询 `job_status`，不要高频重试
