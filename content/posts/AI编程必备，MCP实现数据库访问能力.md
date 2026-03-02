---
title: "AI 编程必备：MCP 实现数据库访问能力（PostgreSQL / MySQL / Oracle）"
date: 2026-02-27
lastmod: 2026-02-27
slug: "mcp-database-access"
description: "基于 mcp-servers 目录实战，讲清如何用 MCP Server 给 AI 提供数据库访问能力，并附可复用源码（已脱敏）。"
categories: ["AI 编程"]
tags: ["MCP", "数据库", "PostgreSQL", "MySQL", "Oracle", "Node.js"]
keywords: ["MCP 数据库访问", "AI 调用数据库", "MCP Server 教程", "Postgres MCP", "MySQL MCP", "Oracle MCP"]
ShowToc: true
faq:
  - q: "MCP 访问数据库时最重要的安全点是什么？"
    a: "优先保证只读能力，限制 SQL 类型并对连接信息做脱敏与权限隔离，避免写操作和敏感数据泄露。"
  - q: "一个 MCP Server 可以同时支持多种数据库吗？"
    a: "可以，常见做法是按数据库类型拆分 server 文件并统一暴露 query 工具接口。"
draft: false
---

## 为什么 AI 编程需要 MCP + 数据库

在 AI 编程场景里，模型如果只能看代码，无法直接访问数据库，很多任务都做不完整，例如：

- 快速验证线上表结构是否和 ORM 一致
- 让 AI 直接查询配置表、字典表做排查
- 在不暴露写权限的前提下完成“读库分析”

MCP（Model Context Protocol）正好解决这件事：给模型一个标准化的工具层，把数据库能力以工具形式暴露出来。

本文基于 `mcp-servers` 目录里的实现，提炼成一套可复用方案，并把文中所有连接串、账号等信息统一脱敏。

---

## 先讲原理：MCP 在这件事里到底做了什么

可以把 MCP 理解成 **“AI 的标准工具总线”**：

- **大模型**：负责理解问题和决策“要不要调用工具”
- **MCP Client（如 IDE/Agent）**：负责把工具调用请求转发给 MCP Server
- **MCP Server**：真正执行能力（这里是查数据库），再把结果返回给模型

在数据库场景里，MCP 的价值不只是“能查 SQL”，而是把查询能力放进一个可治理边界中：

- 能限制只读
- 能按工具协议定义输入输出
- 能审计每次调用
- 能把多种数据库统一成一套调用方式

---

## MCP 规范要点（本文实现对应关系）

这篇代码里用到的主要是 MCP 的三类接口能力：

1. **ListTools**：告诉模型“我有哪些工具可用”  
   - 本文统一暴露 `query` 工具  
2. **CallTool**：执行工具调用  
   - 执行 SQL，并返回结构化文本结果  
3. **ListResources / ReadResource**：资源发现与读取  
   - 列出数据表、读取表结构（列名/类型）

### 为什么要同时提供 Tools + Resources？

- 只有 `query`：模型能查，但容易“盲查”
- 加上 `ListResources/ReadResource`：模型可先理解表结构再查询，成功率明显更高

这也是“数据库 MCP”实战里很关键的一点：  
**先让模型看 schema，再让模型写 SQL。**

---

## 实战规范：写数据库 MCP Server 的 8 条约束

结合本文代码，建议固定以下规则：

1. **只读 SQL 白名单**：只允许 `SELECT/WITH/SHOW/EXPLAIN...`
2. **连接串脱敏**：对外暴露资源 URI 时清空密码
3. **最小权限账号**：MCP 独立只读账号，不复用 DBA 账号
4. **事务保护（可选）**：如 Postgres 用 `BEGIN READ ONLY + ROLLBACK`
5. **统一错误语义**：非法 SQL、参数缺失、URI 非法都要明确报错
6. **统一输出格式**：尽量 JSON 文本，便于模型稳定解析
7. **资源命名稳定**：`schema` 路径约定固定，减少模型混淆
8. **可观测性**：记录调用日志（至少 SQL 摘要 + 耗时 + 调用来源）

如果只记一条：  
**MCP 不是数据库直连脚本，它是“可控的能力接口层”。**

---

## 目录里有什么

`mcp-servers` 的核心文件非常清晰：

- `postgres-mcp-server.mjs`
- `mysql-mcp-server.mjs`
- `oracle-mcp-server.mjs`
- `package.json`（依赖：`@modelcontextprotocol/sdk`、`pg`、`mysql2`、`oracledb`）

整体设计统一：  
**每个数据库都暴露相同的 3 类能力**

1. `ListResources`：列出表资源  
2. `ReadResource`：读取某个表结构  
3. `query` 工具：执行只读 SQL

---

## 通用架构（可直接复用）

核心流程是：

1. 启动 `Server` + `StdioServerTransport`
2. 读取 CLI 参数中的数据库连接 URL（例如 `postgresql://...`）
3. 建立连接池
4. 注册 `ListResources` / `ReadResource` / `CallTool(query)`
5. 在 `query` 里做只读 SQL 限制（`SELECT/WITH/SHOW/EXPLAIN...`）

这套结构最大的好处是：  
换数据库只改驱动和元数据查询 SQL，MCP 接口保持一致。

---

## 源码示例 1：PostgreSQL MCP Server（脱敏版）

```javascript
#!/usr/bin/env node
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListResourcesRequestSchema,
  ListToolsRequestSchema,
  ReadResourceRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";
import pg from "pg";

const { Pool } = pg;
const server = new Server(
  { name: "example-servers/postgres", version: "0.1.0" },
  { capabilities: { resources: {}, tools: {} } }
);

const databaseUrl = process.argv[2]; // 例: postgresql://<USER>:<PASS>@<HOST>:5432/<DB>
if (!databaseUrl) process.exit(1);

const pool = new Pool({ connectionString: databaseUrl });
const SCHEMA_PATH = "schema";

server.setRequestHandler(ListResourcesRequestSchema, async () => {
  const client = await pool.connect();
  try {
    const result = await client.query(
      "SELECT table_schema, table_name FROM information_schema.tables WHERE table_schema NOT IN ('pg_catalog','information_schema','pg_toast') AND table_type='BASE TABLE' ORDER BY table_schema, table_name"
    );
    const resourceBaseUrl = new URL(databaseUrl);
    resourceBaseUrl.protocol = "postgres:";
    resourceBaseUrl.password = ""; // 脱敏
    return {
      resources: result.rows.map((row) => ({
        uri: new URL(`${row.table_schema}.${row.table_name}/${SCHEMA_PATH}`, resourceBaseUrl).href,
        mimeType: "application/json",
        name: `"${row.table_schema}"."${row.table_name}" schema`,
      })),
    };
  } finally {
    client.release();
  }
});

server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
  const u = new URL(request.params.uri);
  const parts = u.pathname.split("/");
  const schema = parts.pop();
  const fullTable = parts.pop();
  if (schema !== SCHEMA_PATH || !fullTable) throw new Error("Invalid resource URI");
  const [tableSchema, tableName] = fullTable.includes(".") ? fullTable.split(".") : ["public", fullTable];
  const client = await pool.connect();
  try {
    const result = await client.query(
      "SELECT column_name, data_type FROM information_schema.columns WHERE table_schema=$1 AND table_name=$2 ORDER BY ordinal_position",
      [tableSchema, tableName]
    );
    return {
      contents: [{ uri: request.params.uri, mimeType: "application/json", text: JSON.stringify(result.rows, null, 2) }],
    };
  } finally {
    client.release();
  }
});

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "query",
      description: "Run a read-only SQL query against PostgreSQL",
      inputSchema: { type: "object", properties: { sql: { type: "string" } }, required: ["sql"] },
    },
  ],
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name !== "query") throw new Error("Unknown tool");
  const sql = request.params.arguments?.sql;
  const normalized = (sql || "").trim().toLowerCase();
  if (!(normalized.startsWith("select") || normalized.startsWith("with") || normalized.startsWith("show") || normalized.startsWith("explain"))) {
    throw new Error("Only read-only SQL is allowed");
  }
  const client = await pool.connect();
  try {
    await client.query("BEGIN TRANSACTION READ ONLY");
    const result = await client.query(sql);
    return { content: [{ type: "text", text: JSON.stringify(result.rows, null, 2) }], isError: false };
  } finally {
    await client.query("ROLLBACK").catch(() => {});
    client.release();
  }
});

await server.connect(new StdioServerTransport());
```

---

## 源码示例 2：MySQL MCP Server（脱敏版）

```javascript
#!/usr/bin/env node
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListResourcesRequestSchema,
  ListToolsRequestSchema,
  ReadResourceRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";
import mysql from "mysql2/promise";

const server = new Server(
  { name: "example-servers/mysql", version: "0.1.0" },
  { capabilities: { resources: {}, tools: {} } }
);

const databaseUrl = process.argv[2]; // 例: mysql://<USER>:<PASS>@<HOST>:3306/<DB>
if (!databaseUrl) process.exit(1);
const pool = mysql.createPool(databaseUrl);
const SCHEMA_PATH = "schema";

server.setRequestHandler(ListResourcesRequestSchema, async () => {
  const conn = await pool.getConnection();
  try {
    const [rows] = await conn.query(
      "SELECT table_name FROM information_schema.tables WHERE table_schema = DATABASE()"
    );
    const base = new URL(databaseUrl);
    base.password = ""; // 脱敏
    return {
      resources: rows.map((row) => ({
        uri: new URL(`${row.TABLE_NAME}/${SCHEMA_PATH}`, base).href,
        mimeType: "application/json",
        name: `"${row.TABLE_NAME}" database schema`,
      })),
    };
  } finally {
    conn.release();
  }
});

server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
  const u = new URL(request.params.uri);
  const parts = u.pathname.split("/");
  const schema = parts.pop();
  const tableName = parts.pop();
  if (schema !== SCHEMA_PATH) throw new Error("Invalid resource URI");
  const conn = await pool.getConnection();
  try {
    const [rows] = await conn.query(
      "SELECT column_name, data_type FROM information_schema.columns WHERE table_schema = DATABASE() AND table_name = ?",
      [tableName]
    );
    return {
      contents: [{ uri: request.params.uri, mimeType: "application/json", text: JSON.stringify(rows, null, 2) }],
    };
  } finally {
    conn.release();
  }
});

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "query",
      description: "Run a read-only SQL query against MySQL",
      inputSchema: { type: "object", properties: { sql: { type: "string" } }, required: ["sql"] },
    },
  ],
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name !== "query") throw new Error("Unknown tool");
  const sql = request.params.arguments?.sql;
  const normalized = (sql || "").trim().toLowerCase();
  if (!(normalized.startsWith("select") || normalized.startsWith("show") || normalized.startsWith("describe") || normalized.startsWith("explain"))) {
    throw new Error("Only read-only SQL is allowed");
  }
  const conn = await pool.getConnection();
  try {
    const [rows] = await conn.query(sql);
    return { content: [{ type: "text", text: JSON.stringify(rows, null, 2) }], isError: false };
  } finally {
    conn.release();
  }
});

await server.connect(new StdioServerTransport());
```

---

## 源码示例 3：Oracle MCP Server（脱敏版）

```javascript
#!/usr/bin/env node
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListResourcesRequestSchema,
  ListToolsRequestSchema,
  ReadResourceRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

const oracledbModule = await import("oracledb");
const oracledb = oracledbModule.default ?? oracledbModule;

// 可选：ORACLE_CLIENT_LIB_DIR=/path/to/instantclient
oracledb.initOracleClient(
  process.env.ORACLE_CLIENT_LIB_DIR ? { libDir: process.env.ORACLE_CLIENT_LIB_DIR } : undefined
);

const server = new Server(
  { name: "example-servers/oracle", version: "0.1.0" },
  { capabilities: { resources: {}, tools: {} } }
);

const databaseUrl = process.argv[2]; // 例: oracle://<USER>:<PASS>@<HOST>:1521/<SERVICE_NAME>
if (!databaseUrl) process.exit(1);
const url = new URL(databaseUrl);
const user = decodeURIComponent(url.username || "");
const password = decodeURIComponent(url.password || "");
const host = url.hostname;
const port = url.port || "1521";
const serviceName = url.pathname.replace(/^\//, "");
const connectString = `${host}:${port}/${serviceName}`;

const pool = await oracledb.createPool({ user, password, connectString });
const SCHEMA_PATH = "schema";

server.setRequestHandler(ListResourcesRequestSchema, async () => {
  const conn = await pool.getConnection();
  try {
    const result = await conn.execute("SELECT table_name FROM user_tables ORDER BY table_name");
    const base = new URL(databaseUrl);
    base.password = ""; // 脱敏
    return {
      resources: result.rows.map((row) => ({
        uri: new URL(`${(row.TABLE_NAME || row[0])}/${SCHEMA_PATH}`, base).href,
        mimeType: "application/json",
        name: `"${row.TABLE_NAME || row[0]}" database schema`,
      })),
    };
  } finally {
    await conn.close();
  }
});

server.setRequestHandler(ReadResourceRequestSchema, async (request) => {
  const u = new URL(request.params.uri);
  const parts = u.pathname.split("/");
  const schema = parts.pop();
  const tableName = parts.pop();
  if (schema !== SCHEMA_PATH) throw new Error("Invalid resource URI");
  const conn = await pool.getConnection();
  try {
    const result = await conn.execute(
      "SELECT column_name, data_type FROM user_tab_columns WHERE table_name = :table",
      { table: tableName.toUpperCase() }
    );
    const rows = (result.rows || []).map((row) => ({
      column_name: row.COLUMN_NAME || row[0],
      data_type: row.DATA_TYPE || row[1],
    }));
    return {
      contents: [{ uri: request.params.uri, mimeType: "application/json", text: JSON.stringify(rows, null, 2) }],
    };
  } finally {
    await conn.close();
  }
});

server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "query",
      description: "Run a read-only SQL query against Oracle",
      inputSchema: { type: "object", properties: { sql: { type: "string" } }, required: ["sql"] },
    },
  ],
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  if (request.params.name !== "query") throw new Error("Unknown tool");
  const sql = request.params.arguments?.sql;
  const normalized = (sql || "").trim().toLowerCase();
  if (!(normalized.startsWith("select") || normalized.startsWith("with"))) {
    throw new Error("Only read-only SQL is allowed");
  }
  const conn = await pool.getConnection();
  try {
    const result = await conn.execute(sql);
    return {
      content: [{ type: "text", text: JSON.stringify(result.rows ?? [], null, 2) }],
      isError: false,
    };
  } finally {
    await conn.close();
  }
});

await server.connect(new StdioServerTransport());
```

---

## 本地运行方式（脱敏示例）

> 以下示例均使用占位符，请替换成你自己的安全配置；不要把真实密码提交到仓库。

```bash
# PostgreSQL
node mcp-servers/postgres-mcp-server.mjs "postgresql://<USER>:<PASS>@<HOST>:5432/<DB>"

# MySQL
node mcp-servers/mysql-mcp-server.mjs "mysql://<USER>:<PASS>@<HOST>:3306/<DB>"

# Oracle
node mcp-servers/oracle-mcp-server.mjs "oracle://<USER>:<PASS>@<HOST>:1521/<SERVICE_NAME>"
```

---

## 安全建议（强烈推荐）

1. **只读优先**：`query` 严格限制为只读 SQL 前缀  
2. **账号最小权限**：给 MCP 单独只读账户  
3. **连接信息不入库**：使用环境变量或密钥管理，不把明文写进仓库  
4. **审计日志**：记录 AI 发起的 SQL（用于回溯）  
5. **表级白名单**：高敏表（如账号、财务）默认不暴露资源

---

## 小结

`mcp-servers` 这套实现最有价值的点，不是“能查数据库”，而是把数据库能力变成了统一、可控、可审计的 MCP 工具层。  
对 AI 编程来说，这一步就是从“只会写代码”升级为“能理解真实数据结构并辅助排障”。

如果你已经有多种数据库环境，建议直接按本文模式统一抽象成：

- 标准化 `ListResources / ReadResource / query`
- 统一只读策略
- 统一脱敏和日志规范

这样后续不管接入哪个 AI Agent，数据库访问能力都能快速复用。

---

## 相关阅读

- [小米路由器实现科学上网](/posts/小米路由器实现科学上网/)
- [站长平台提交 SOP：Google / Bing / 百度](/posts/webmaster-platform-sop/)
