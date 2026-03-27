---
name: Stack preference - hybrid Node.js + Python
description: User rejected pure Python stack. Use Node.js/TS for server/infra layer, Python only for AI/ML workers
type: feedback
---

Use a hybrid stack: Node.js/TypeScript for server-side (API, data collectors, execution engine, real-time WebSocket handling) and Python only for AI/ML workers and backtesting.

**Why:** User works with Node.js professionally and finds Python slow and not great for scaling on the server side. Node.js event loop is better suited for high-concurrency I/O (WebSocket feeds, API calls, order execution).

**How to apply:** When designing services, default to TypeScript/Node.js. Only use Python for: ML model training, inference servers, backtesting engine, and data science notebooks.
