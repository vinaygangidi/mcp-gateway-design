# mcp-gateway-design

MCP gateway reference architecture: design doc and a Salesforce MCP tool implementation.

![Language](https://img.shields.io/badge/language-Jupyter%20Notebook-orange?style=flat-square)
![Last Commit](https://img.shields.io/github/last-commit/vinaygangidi/mcp-gateway-design?style=flat-square)
![License](https://img.shields.io/badge/license-MIT-blue?style=flat-square)

## What This Does

This repository holds design material for an enterprise MCP (Model Context Protocol)
gateway — the layer that sits between an MCP client and internal systems, deciding who
may call which tool and with what scope. It contains a written design document and one
fully worked reference tool implementation against the Salesforce REST API.

This is a **design and reference repository, not a deployable gateway.** There is no
server, no routing layer, and no packaging. The value is the design document and the
tool implementation as a pattern to copy.

## How It Works

The reference tool in `get_my_open_opportunities.ipynb` implements a single MCP tool
call end to end. Its control flow:

1. **Receive request** with a `UserContext` (identity, group membership, and a delegated
   Salesforce access token) plus caller parameters.
2. **Assign a correlation ID** (`uuid4`) and start a latency timer.
3. **Validate input bounds** — `min_amount` within 0–10,000,000, `close_within_days`
   within 0–365, `limit` within 1–200. Out-of-range values raise `BadInput`.
4. **Authorize** — `_assert_allowed()` requires the user to be in `sales`, `cs`, or
   `support`. Otherwise `Forbidden`.
5. **Force record scoping** — `OwnerId` is set server-side from `ctx.sf_user_id`. The
   caller cannot widen it to another user's records.
6. **Build an allowlisted SOQL query.** No raw SOQL is accepted from the client; the
   user ID is escaped via `_soql_escape()`.
7. **Call Salesforce** (`/services/data/v59.0/query`) with an 8-second timeout. HTTP 401
   maps to `Forbidden`, any other 4xx/5xx to `BackendError`.
8. **Allowlist output fields** — only the 7 opportunity fields and 2 account fields are
   copied into the response, so no extra Salesforce columns leak.
9. **Log the outcome** with request ID, user email, record count, and latency.

Error handling is deliberately tiered: `Forbidden` and `BadInput` are logged at warning
and re-raised as-is; `BackendError` is logged without the response payload; any
unexpected exception is caught and re-raised as a generic `BackendError` so internal
details are not returned to the caller.

`MCPDesign.pdf` (7 pages, 3 diagrams) contains the broader gateway design.
**[PLACEHOLDER]** — the PDF uses a custom font encoding that does not extract to text
programmatically, so its architecture is not summarized here. Fill in the gateway's
components and request flow from the document, or export it to Markdown.

## Quickstart

This repository contains a notebook and a document. There is no service to run.

1. Clone the repository:
   ```bash
   git clone https://github.com/vinaygangidi/mcp-gateway-design.git
   cd mcp-gateway-design
   ```

2. Install the one runtime dependency (there is no `requirements.txt`):
   ```bash
   pip install requests jupyter
   ```

3. Open the notebook:
   ```bash
   jupyter notebook get_my_open_opportunities.ipynb
   ```

4. Read `MCPDesign.pdf` for the gateway design.

The notebook defines functions but does not execute a call. To exercise it, construct a
`UserContext` with a valid Salesforce instance URL and OAuth access token and invoke
`get_my_open_opportunities(ctx, min_amount=50000)`.

## Configuration

The code reads **no environment variables.** All configuration arrives as function
arguments — a deliberate choice, since the gateway pattern assumes the calling layer
already resolved the user's identity and delegated credentials.

| Name | Required | Default | Description |
|---|---|---|---|
| `ctx.user_id` | Yes | — | Internal user identifier, carried into logs |
| `ctx.email` | Yes | — | User email, used as the log subject |
| `ctx.groups` | Yes | — | Group list; must contain `sales`, `cs`, or `support` |
| `ctx.sf_user_id` | Yes | — | Salesforce user ID; forced into `OwnerId` |
| `ctx.sf_instance_url` | Yes | — | Salesforce instance base URL |
| `ctx.sf_access_token` | Yes | — | Delegated OAuth bearer token |
| `min_amount` | Yes | — | Minimum opportunity amount; 0–10,000,000 |
| `close_within_days` | No | `30` | Close-date window; 0–365 |
| `limit` | No | `25` | Max records; 1–200 |
| `timeout_s` | No | `8.0` | Salesforce HTTP timeout in seconds |

## Limitations

- **Not a gateway.** No server, router, or transport. One tool function and a design
  document. The repository name promises more than the code delivers.
- **Not an MCP server.** The function is not registered as an MCP tool — no protocol
  handler, no manifest, no stdio/SSE transport. It is the tool *body* only.
- **No tests.** No test file, no CI, no verification that the RBAC gate or the field
  allowlist behave as intended.
- **No dependency manifest.** `requests` is imported but there is no `requirements.txt`
  or `pyproject.toml`.
- **Rate limiting and audit logging are absent from the code.** Logging goes to a
  Python `logger` with a `# In production this would go to a log pipeline` note. There is
  no persistence, no audit trail, and no rate limiter.
- **SOQL is built by f-string interpolation.** `_soql_escape()` handles backslashes and
  single quotes on the user ID, and the numeric parameters are bounds-checked, which
  closes the obvious injection paths — but this is string concatenation, not
  parameterized query binding.
- **Token lifecycle is out of scope.** The tool consumes an access token and maps HTTP
  401 to `Forbidden`. It does not refresh, cache, or revoke.
- **Salesforce API v59.0 is hardcoded** in the request URL.
- **Single tool, single object.** Only open Opportunities owned by the caller.
- **The PDF is not machine-readable** and its content is not reflected in this README.
  See the `[PLACEHOLDER]` above.

## License

MIT — see [LICENSE](LICENSE).
