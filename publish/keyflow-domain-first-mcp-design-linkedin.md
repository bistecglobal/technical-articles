Design MCP tools around what agents want to *accomplish* — not around what your REST layer exposes.

We learned this building Keyflow's AI integration. Our first MCP layer mirrored REST endpoints: `list_objectives`, `create_key_result`. Peer recognition, feedback, analytics — entire domains were invisible because they weren't database tables.

The redesign: one tool per business domain, `action` parameter dispatching operations. 10 composite tools now cover everything Claude or Copilot Studio needs to drive OKR workflows.

Patterns that made it work:

• **Flat Zod schemas** — `discriminatedUnion` breaks Copilot Studio's parser. Flat `z.object` + handler-level validation keeps tools working across all MCP clients.

• **`next_steps` in every response** — each result embeds actionable hints. Agents don't need to ask what comes next.

• **Shared service layer** — extracted `src/lib/services/` so business logic runs identically for humans and agents. Surfaced a real bug where MCP and REST had silently diverged.

• **RBAC at the tool boundary** — machine tokens can read; writes that impersonate users require an OAuth user-scoped token.

Domain boundaries are more stable than database boundaries.

Full article → [link placeholder — operator fills in after Medium publish]

#MCP #AIAgents #OKR #TypeScript #EnterpriseAI
