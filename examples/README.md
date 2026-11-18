# Examples

Messages are JSON-RPC 2.0, sent with `POST https://conta.cartaregistada.pt/mcp`, the
`Authorization: Bearer <access token>` header, `Content-Type: application/json` and
`MCP-Protocol-Version: 2025-06-18`. Responses are always JSON (never SSE).

- [`initialize.json`](initialize.json): the first request of a session. The response carries
  `serverInfo`, `capabilities` and the server `instructions`.
- [`tools-list.json`](tools-list.json): lists the nine tools with their `inputSchema` and
  annotations.
- [`orcamento-response.json`](orcamento-response.json): the shape of a `tools/call` response
  for `orcamento`. The `content` text is what the assistant repeats to the person; the
  `structuredContent` object has the same data as fields: the base price, the included
  pages and any extra ones, the urgent and digital-receipt surcharges, `total_cents` and
  the formatted `total`, and `sai_em` (the day the letter leaves for CTT if paid today).
  The real response carries the account's prices; the amounts here are placeholders
  (`0` and `<total>`).

The request for that quote looks like:

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "orcamento",
    "arguments": { "tipo": "registada_com_ar", "paginas": 2 }
  }
}
```

A business error from a tool (an unknown postal code, a missing sender, an unconfirmed
price) comes back as a result with `isError: true` and a text explaining what to do, not as
a JSON-RPC error. A missing OAuth scope is an HTTP 403 with `WWW-Authenticate`.
