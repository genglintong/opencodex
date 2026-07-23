# opencodex 管理 API 端点盘点（Menubar 挂件适用性）

> 调研日期：2026-07-24
> 目的：盘点现有管理 API 中可供 menubar 挂件直接消费的端点

## 认证方式

- **Header**: `X-OpenCodex-API-Key: <token>`
- **来源限制**: 默认仅允许 loopback（127.0.0.1）请求，可通过 `corsAllowOrigins` 配置扩展
- **GUI 行为**: 401 时弹出 prompt 让用户输入 token，存入 sessionStorage
- **挂件适配**: 挂件需要在首次配置时获取 API token，后续请求携带 header 即可

## 实时推送能力

- **现有 WebSocket**: `ws-bridge.ts` 仅用于代理 Codex CLI 的 WebSocket 连接（数据面），不用于管理 API
- **SSE**: 无
- **结论**: 管理 API 纯 REST 轮询模式，挂件需要自行实现定时轮询（建议 5-10s 间隔）

## 端点清单

### 运行状态

| 端点 | 方法 | 返回数据 | 挂件适用性 |
|------|------|----------|------------|
| `/api/settings` | GET | `{ codexAutoStart, port, hostname, startupHealth }` | ✅ 核心：代理运行状态、端口 |
| `/api/startup-health` | GET | `{ status, routingKind, autostartEnabled, shimCoverage, diagnosticStale }` | ✅ 健康状态指示 |
| `/api/config` | GET | 完整配置 DTO（脱敏） | ⚠️ 数据量大，挂件只需部分字段 |

### 请求统计 / 用量

| 端点 | 方法 | 返回数据 | 挂件适用性 |
|------|------|----------|------------|
| `/api/usage?range=<range>&surface=<surface>` | GET | `{ summary: { requests, totalTokens, inputTokens, outputTokens, estimatedCostUsd, coverageRatio, ... }, days[], models[], providers[] }` | ✅ 核心：请求数、token 用量、费用 |
| `/api/logs` | GET | `RequestLogEntry[]`（requestId, timestamp, model, provider, firstOutputMs, usage, attempts） | ✅ 最近请求列表、TTFT 延迟 |

### 活跃路由 / Combo

| 端点 | 方法 | 返回数据 | 挂件适用性 |
|------|------|----------|------------|
| `/api/combos` | GET | `{ combos: [{ id, model, ...comboConfig }] }` | ✅ 核心：当前所有 combo 及其路由配置 |
| `/api/combos` | PUT | 创建/更新 combo | ✅ 快捷操作：切换/编辑 combo |
| `/api/combos?id=<id>` | DELETE | 删除 combo | ⚠️ 危险操作，挂件需确认 |

### Provider 余额 / Quota

| 端点 | 方法 | 返回数据 | 挂件适用性 |
|------|------|----------|------------|
| `/api/provider-quotas?refresh=<0|1>` | GET | `{ generatedAt, reports: [{ provider, label, source, quota: { fiveHourPercent, weeklyPercent, monthlyPercent, customWindows[], ...ResetAt }, updatedAt }] }` | ✅ 核心：各 provider 的 quota 消耗百分比和重置时间 |
| `/api/providers` | GET | `[{ name, adapter, baseUrl, defaultModel, hasApiKey, disabled, ... }]` | ✅ provider 列表及状态 |

### 操作端点

| 端点 | 方法 | 功能 | 挂件适用性 |
|------|------|------|------------|
| `/api/combos` | PUT | 创建/更新/重命名 combo | ✅ 切换路由 |
| `/api/providers` | PATCH | 修改 provider 属性（disabled, adapter 等） | ✅ 启停 provider |
| `/api/sync` | POST | 同步模型到 Codex | ⚠️ 低频操作 |
| `/api/stop` | POST | 停止代理 | ⚠️ 危险操作，需二次确认 |
| `/api/settings` | PUT | 修改设置（codexAutoStart） | ✅ 简单开关 |
| `/api/update/check` | GET | 检查更新 | ⚠️ 可选 |
| `/api/update/run` | POST | 执行更新 | ⚠️ 低频 |

## 关键数据结构

### ProviderQuotaReport

```typescript
interface ProviderQuotaReport {
  provider: string;       // provider 名称
  label: string;          // 显示名称
  source: string;         // 数据来源
  quota: {
    fiveHourPercent?: number;   // 5 小时窗口使用百分比
    fiveHourResetAt?: number;   // 重置时间戳
    weeklyPercent?: number;     // 周使用百分比
    weeklyResetAt?: number;
    monthlyPercent?: number;    // 月使用百分比
    monthlyResetAt?: number;
    customWindows?: { label: string; percent: number; resetAt?: number }[];
    updatedAt: number;
  };
  updatedAt: number;
}
```

### RequestLogEntry（关键字段）

```typescript
interface RequestLogEntry {
  requestId: string;
  timestamp: number;
  model: string;
  provider: string;
  firstOutputMs?: number;  // TTFT（首 token 延迟）
  surface?: "claude";
  requestedModel?: string;
  usage?: { input_tokens?: number; output_tokens?: number; total_tokens?: number };
  status?: number;
}
```

### Usage Summary

```typescript
interface UsageSummary {
  requests: number;
  totalTokens: number;
  inputTokens: number;
  outputTokens: number;
  estimatedCostUsd: number;
  coverageRatio: number;
  // ... 更多字段
}
```

## 挂件所需但现有 API 缺失的数据

| 需求 | 现状 | 建议 |
|------|------|------|
| 实时 QPS | 无专用端点，需从 `/api/logs` 自行计算 | 可新增 `/api/stats/realtime` 或挂件本地计算 |
| 当前活跃连接数 | 无 | 可新增，或从 ws-bridge 暴露 |
| 聚合延迟统计（P50/P95） | `/api/logs` 有 firstOutputMs 但无聚合 | 挂件本地聚合或新增端点 |
| 代理 uptime | `/api/settings` 无 uptime 字段 | Dashboard 的 HealthData 有 uptime，需确认来源 |
| 轻量状态端点 | 无一次性获取所有挂件数据的端点 | 建议新增 `/api/widget/summary` 聚合端点 |

## 结论

现有管理 API 已覆盖挂件所需的 **大部分数据**（quota、combo、usage、logs、provider 状态）。主要缺口是：

1. **无实时推送**：需要轮询，建议 5-10s 间隔
2. **无聚合统计端点**：QPS、P50 延迟等需要挂件本地计算或新增端点
3. **无轻量聚合端点**：挂件需要调多个 API 才能拼出完整面板，建议后续新增 `/api/widget/summary`
4. **认证简单**：单个 API key header，挂件配置一次即可
