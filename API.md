# V2EX HODL API

所有接口默认返回 JSON：`{ ok: boolean, ... }`。除上传接口外，均为 HTTP GET。

通用说明：
- 时间参数 `from`/`to` 支持三种输入：epoch 秒、epoch 毫秒或 ISO 时间（如 `2025-01-01T00:00:00Z`）。
- 分页参数：`limit`（默认 200，最大 5000），`offset`（默认 0）。
- 排序参数：`order=asc|desc`，各接口对应的排序列见各自小节。
- CORS：所有接口均设置 `Access-Control-Allow-Origin: *`；支持预检 `OPTIONS`。

错误响应（通用）：
```
{ ok: false, error: string }
```

---

## 1) 统计快照：原始表 v2ex_hodl

- 路径：`GET /api/stats`
- 排序列：`created_at`（默认 `order=desc`）
- 查询参数：
	- `from`：过滤 `created_at >= from`
	- `to`：过滤 `created_at <= to`
	- `order`：`asc|desc`（默认 `desc`）
	- `span`：按时间跨度做抽样，仅返回每个时间桶（跨度为 `span`）的一条记录。
	  - 支持格式：纯数字（秒）或带单位：`ms`/`s`/`m`/`h`（如 `500ms`、`2s`、`3m`、`1h`）。
	  - 当存在 `span` 时：会按 `created_at` 将记录分桶（以 `strftime('%s', created_at)/span` 取整），每桶仅保留一条。
	  - 哪条被保留由 `order` 决定：`order=desc` 选每桶中时间最晚的一条；`order=asc` 选每桶中时间最早的一条。
	  - 注意：`ms` 会被下取整到秒，最小 1 秒。
	- `limit`、`offset`
- 成功响应：
```
{
	ok: true,
	count: number,
	order: "asc" | "desc",
	limit: number,
	offset: number,
	span: number | null, // 秒；仅当传入 span 时返回其秒值
	results: Array<{
		id: number,
		hodl_10k_addresses_count: number,
		new_accounts_via_solana: number,
		total_solana_addresses_linked: number,
		sol_tip_operations_count: number,
		member_tips_sent: number,
		member_tips_received: number,
		total_sol_tip_amount: number,
		v2ex_token_tip_count: number,
		holders: number,
		price: number,
		price_change_24h: number,
		current_online_users: number,
		btc_price: number,
		sol_price: number,
		pump_price: number,
		total_v2ex_token_tip_amount: number,
	created_at: string  // UTC 文本 "YYYY-MM-DD HH:MM:SS"
	}>
}
```

示例：
- `/api/stats?from=2025-01-01&limit=100`
- `/api/stats?from=2025-01-01T10:31:00Z&to=2025-01-01T10:32:00Z&span=2s` —— 在该 60 秒窗口内按 2 秒取样，约返回 30 条（因每 2 秒选一个点）。

---

## 2) 统计历史：压缩表 v2ex_hodl_history

两个入口都查询同一张历史表，但校验规则略有不同：
- `GET /api/stats/history`：要求至少提供 `from` 或 `to`（否则 400）。
- `GET /api/history`：`from`/`to` 可选；不提供时返回最新的若干区间片段。

- 排序列：`statistics_date`（默认 `order=desc`）
- 查询参数：
	- `from`：日期范围过滤，按 `statistics_date >= from`
	- `to`：日期范围过滤，按 `statistics_date <= to`
	- `order`、`limit`（两个路径均不支持 `offset`，与实现一致）
- 成功响应：
```
{
	ok: true,
	count: number,
	from: number | null,   // 仅当传入时返回
	to: number | null,     // 仅当传入时返回
	order: "asc" | "desc",
	limit: number,
	results: Array<{
		id: number,
		hodl_10k_addresses_count: number,
		new_accounts_via_solana: number,
		total_solana_addresses_linked: number,
		sol_tip_operations_count: number,
		member_tips_sent: number,
		member_tips_received: number,
		total_sol_tip_amount: number,
		v2ex_token_tip_count: number,
		total_v2ex_token_tip_amount: number,
		current_online_users: number,
		peak_online_users: number,
		holders: number,
		price: number,
		price_change_24h: number,
		btc_price: number,
		sol_price: number,
		pump_price: number,
	statistics_date: string  // UTC 文本 YYYY-MM-DD
	}>
}
```

---

## 3) Token 持仓快照：v2exer_solana_address

- 路径：`GET /api/holders`
- 排序列：`checked_at`（默认 `order=desc`）
- 查询参数：
	- `token`：按 token mint 过滤
	- `owner`：按持有者钱包地址过滤
	- `from`/`to`：对 `checked_at` 进行时间过滤（UTC）
	- `order`、`limit`、`offset`
- 成功响应：
```
{
	ok: true,
	count: number,
	order: "asc" | "desc",
	limit: number,
	offset: number,
	results: Array<{
		id: number,
		owner_address: string,
		v2ex_username: string | null,
		avatar_url: string | null,
		token_address: string,
		token_account_address: string | null,
		hold_rank: number | null,
		hold_amount: number,    // 规范化数量（amount / 10^decimals）
		decimals: number,
		hold_percentage: number,
		checked_at: string,      // UTC: YYYY-MM-DD HH:MM:SS
		rank_delta: number | null,
		amount_delta: string | null // 相对上次抓取的数量变化（规范化十进制字符串），正数增加，负数减少
	}>
}
```

---

## 4) 已移除持仓归档：v2exer_solana_address_removed

- 路径：`GET /api/holders/removed`
- 排序列：`removed_at`（默认 `order=desc`）
- 查询参数：
	- `token`：按 token mint 过滤
	- `owner`：按持有者钱包地址过滤
	- `from`/`to`：对 `removed_at` 进行时间过滤（UTC）
	- `order`、`limit`、`offset`
- 成功响应：
```
{
	ok: true,
	count: number,
	order: "asc" | "desc",
	limit: number,
	offset: number,
	results: Array<{
		id: number,
		owner_address: string,
		v2ex_username: string | null,
		avatar_url: string | null,
		token_address: string,
		token_account_address: string | null,
		hold_rank: number | null,
		hold_amount: number,
		decimals: number,
		hold_percentage: number,
		rank_delta: number | null,
		removed_at: string // UTC: YYYY-MM-DD HH:MM:SS
	}>
}
```

示例：
- `/api/holders/removed?token=...&from=2025-01-01`
- `/api/holders/removed?owner=...&order=asc&limit=100`

---

## 5) Token 持仓历史：v2exer_solana_address_history

- 路径：`GET /api/holders/history`
- 排序列：`statistics_date`（默认 `order=desc`）
- 查询参数：
	- `token`、`owner`
	- `from`：日期范围过滤，按 `statistics_date >= from`
	- `to`：日期范围过滤，按 `statistics_date <= to`
	- `order`、`limit`、`offset`
- 成功响应：
```
{
	ok: true,
	count: number,
	order: "asc" | "desc",
	limit: number,
	offset: number,
	results: Array<{
		id: number,
		owner_address: string,
		v2ex_username: string | null,
		avatar_url: string | null,
		token_address: string,
		token_account_address: string | null,
		hold_rank: number | null,
		hold_amount: number,
		decimals: number,
	hold_percentage: number,
	statistics_date: string, // UTC 文本 YYYY-MM-DD
	rank_delta: number | null
	}>
}
```

---

## 其它说明

- from/to 校验：当提供 `from`/`to` 时，若解析失败将返回 `400`。
- 时间单位：
	- v2ex_hodl 的时间列为 UTC 文本（`YYYY-MM-DD HH:MM:SS`）。
	- v2ex_hodl_history 的时间列为按日 UTC 文本（`YYYY-MM-DD`）。
	- 其余表的时间列为 UTC 文本（`YYYY-MM-DD HH:MM:SS`）。
- 返回字段：遵循数据库列名；可能会随着迁移脚本演化而扩展，但不会在同一主版本中移除已有字段。

