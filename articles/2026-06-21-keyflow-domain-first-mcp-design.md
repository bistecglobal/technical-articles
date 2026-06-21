---
title: "Domain-First MCP Design: Building OKR Tools That AI Agents Actually Use"
project: keyflow
tags: [AI, MCP, OKR, Claude, Copilot Studio, TypeScript, Architecture]
status: audited
date: 2026-06-21
---

# Domain-First MCP Design: Building OKR Tools That AI Agents Actually Use

When we first wired Claude into Keyflow — our OKR and performance management platform — we made the classic mistake: we designed MCP tools around what was easy to expose, not around what agents actually need to do. The result was a brittle layer that required agents to make four sequential calls to accomplish what a user does in one click.

This article documents how we redesigned Keyflow's MCP surface around domain intent, the specific patterns that made the tools reliable across multiple AI clients, and the service-layer refactor that kept REST and MCP from drifting apart.

## The Problem with Tool-First Thinking

Early Keyflow MCP tools mirrored REST endpoints directly: `GET /objectives` became `list_objectives`, `POST /key-results` became `create_key_result`. That seems reasonable until you watch an agent try to recognize a teammate.

Peer recognition was entirely absent from the tool surface — the original server exposed only OKR data operations (`get_active_cycle`, `list_objectives`, `create_key_result`). Agents had no way to celebrate a teammate, send feedback, or query analytics, because those domains weren't part of the database tables the tools were built around.

The deeper problem: our tool boundaries matched our database schema, not our domain. An agent doesn't think in tables — it thinks in tasks.

## Domain-First Design: What It Means

Domain-first MCP design asks a different question: *What does an agent need to accomplish?* rather than *What data does a route return?*

For Keyflow, the domains are clear: recognition, feedback, OKR templates, analytics, objectives, key results, cycles, users. Each became a single MCP tool with an `action` parameter dispatching across operations — `recognition(action='create')`, `recognition(action='list')`. This pattern, visible in `src/app/api/mcp/tools/recognition.ts` (commit `f7161bc`), collapses a scattered endpoint surface into 10 composable tools registered in `src/app/api/mcp/tools/server.ts`.

```typescript
export const RecognitionSchema = z.object({
  action: z
    .enum(["create", "list"])
    .describe(
      "Action to perform: 'create' to post a peer recognition, 'list' to view recognitions received by a user",
    ),
  receiverId: z.string().optional().describe(
    "(create, required) ID of the user to recognize. (list, optional) Filter by recipient"
  ),
  message: z.string().min(1).optional().describe("(create, required) Recognition message text"),
  badge: z.string().optional().describe("(create, optional) Emoji badge or label"),
  // ...
});
```

One tool. One schema. The agent learns the domain, not the API surface.

## Flat Schema: The Copilot Studio Constraint

We discovered early that Zod's `discriminatedUnion` — the obvious choice for action-dispatched tools — breaks under Copilot Studio's schema parser. Copilot Studio expects a flat `z.object` where all fields are optional, with action semantics expressed in field descriptions.

This constraint drove a deliberate choice: mark fields as optional at the schema level, validate them conditionally in the handler. The `recognition.ts` tool is typical — `receiverId` is optional in the Zod schema but required for `create` and validated there:

```typescript
if (!input.receiverId || !input.message) {
  return err(McpErrorCode.VALIDATION, "receiverId and message are required for create action");
}
```

The tradeoff is that Zod loses the ability to statically type "create requires receiverId." We accepted this — the alternative was two separate tools that agents must choose between, which produces different failure modes (choosing the wrong tool vs. missing a field).

## Agent-Readable Responses: The `next_steps` Contract

Every successful response in Keyflow MCP includes a `next_steps` array. This is not documentation — it is active guidance embedded in the response JSON that the calling agent reads directly.

From `src/app/api/mcp/tools/shared.ts`:

```typescript
export interface McpSuccess<T = unknown> {
  ok: true;
  data: T;
  next_steps?: string[];
  meta?: { count?: number; page?: number; total?: number };
}
```

The recognition tool uses this to guide the agent forward:

```typescript
return ok(dto, {
  next_steps: [
    `Recognition posted to ${receiver.name ?? receiver.email}`,
    "Use recognition(action='list') to see recent recognitions",
  ],
});
```

Error responses mirror this pattern — `next_steps` on errors tell the agent how to recover:

```typescript
return err(McpErrorCode.NOT_FOUND, `User '${input.receiverId}' not found in this tenant`, {
  field: "receiverId",
  next_steps: ["Use user(action='list') to find valid user IDs"],
});
```

The design intent: eliminate the retrieval loop where an agent must ask "what do I do next?" after every action. Each response carries its own continuation signal.

## Service Layer: Eliminating REST/MCP Drift

As the MCP surface grew, we hit a maintenance problem: business logic was being duplicated between REST route handlers and MCP tool handlers. The cycle activation logic lived in `src/app/api/cycles/[id]/status/route.ts` and was being reimplemented in `src/app/api/mcp/tools/cycle.ts`. When we fixed a bug in one, the other stayed broken.

The service layer refactor (commit `5149b32`, PR #87) extracted shared logic into `src/lib/services/`:

```
src/lib/services/
  cycles.ts       — activateCycle, cycle query helpers
  key-results.ts  — recalculateObjectiveScore, assertCanEditKr, assertCanDeleteKr
  users.ts        — listUsers, buildUserWhere
  errors.ts       — ServiceForbiddenError
```

Both the REST route and the MCP tool now call the same service function. `recalculateObjectiveScore` runs identically whether triggered by a human clicking the UI or an agent calling `key_result(action='update')`:

```typescript
export async function recalculateObjectiveScore(
  objectiveId: string,
  prisma: PrismaClient,
): Promise<void> {
  const krs = await prisma.keyResult.findMany({
    where: { objectiveId },
    select: { score: true, weight: true },
  });
  await prisma.objective.update({
    where: { id: objectiveId },
    data: { score: calculateObjectiveScore(krs) },
  });
}
```

The PR also caught a real bug: the MCP cycle tool was not setting `isClosed: false` on activation, producing silent divergence from the REST behavior. The service extraction made this visible and fixed it in one place.

## RBAC at the Tool Level

Keyflow is multi-tenant, and agents operate with either user-scoped tokens (obtained via OAuth `authorization_code` flow) or machine tokens. The distinction matters: a machine token cannot post a recognition on behalf of an unknown user.

Each tool handler receives a `TokenContext` (defined in `shared.ts`) containing `userId`, `tenantId`, and `role`. Permission checks happen at the top of the handler, before any database work:

```typescript
export async function handleAnalytics(
  input: AnalyticsInput,
  ctx: TokenContext,
): Promise<McpResult> {
  if (!can(ctx.role as Role | null, "view:team-analytics")) {
    return err(McpErrorCode.FORBIDDEN, "analytics requires MANAGER role or above");
  }
  // ...
}
```

The analytics tool (commit `051cdfc`) enforces Manager-or-above for org-wide metrics. The recognition tool requires a user-scoped token for `create` — machine tokens are rejected with an explicit `next_steps` pointing to the OAuth flow. This means agents running with service credentials can read data freely but cannot impersonate users for write operations.

## What Domain-First Design Delivered

After the redesign, Keyflow's MCP surface comprises 10 composite tools: `cycle`, `objective`, `key_result`, `user`, `report`, `feedback`, `feedback_template`, `recognition`, `okr_template`, and `analytics`. Each maps to a business domain, not a database table or HTTP route.

The OKR template tool (`src/app/api/mcp/tools/template.ts`, commit `051cdfc`) illustrates the approach at its clearest: an agent can `list` templates, `get` one to inspect its KR stubs, and `use` it to create an objective with pre-populated key results — all in a single tool family, with `next_steps` at each stage guiding the agent through the full workflow without external orchestration.

The service layer ensures that correctness fixes propagate to both REST and MCP simultaneously. RBAC enforcement at the tool boundary means agents are subject to the same access controls as human users. And flat Zod schemas keep the tools functional across Claude Desktop, Copilot Studio, and any MCP client that parses JSON Schema.

The core lesson: design MCP tools around what agents want to accomplish, not around what your REST layer already has. The mapping from domain to tool is almost always more stable than the mapping from database to endpoint — and it produces tools that agents actually use correctly on the first attempt.

---

*Keyflow is BistecGlobal's internal OKR and performance management platform. The MCP layer is live in production. Commits referenced: `f7161bc` (recognition tool), `051cdfc` (analytics + template tools), `5149b32` (service layer extraction, PR #87).*
