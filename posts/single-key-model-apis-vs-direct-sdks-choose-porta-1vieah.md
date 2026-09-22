# Single-Key Model APIs vs Direct SDKs: Choose Portability for Node.js Backends

A Node.js healthtech backend should not need three integrations to switch support-ticket triage among OpenAI, Claude, and Gemini. The operational constraint is plain: use a single server-side key and one API shape so classification keeps moving while the selected model remains replaceable.

**TL;DR:** Start with a unified, OpenAI-compatible chat API when portability matters more than immediate access to every provider-specific feature. Keep the selected model ID in configuration, fetch the available-model catalog for an admin control, and make the model return a small validated JSON decision. Choose direct OpenAI, Anthropic, or Google SDKs instead when a proprietary feature is central to the product. For weekly shipping, one request shape is the better default; direct SDKs are the deliberate exception.

## Should a Node.js backend use a single API key to switch models?

The job here is narrow: turn an incoming support ticket into a queue, priority, and short rationale. It is not a general clinical assistant. A request might say, "My home monitor will not sync," or, "I cannot download the invoice for last month's order." The output should route the first ticket toward device support and the second toward billing, while human support staff still own the response. That distinction matters because portability has a manageable target here: preserve one small JSON contract across providers, rather than pretending every native model feature is identical.

One route. Three vendors.

Provider portability changes the architecture because model choice becomes configuration rather than application structure. With three direct integrations, the backend owns three clients, authentication schemes, error mappings, request types, and release cycles. A unified API reduces that surface to one base URL, one bearer key, and one chat request shape. That is valuable even if the chosen model never changes: the code that validates and dispatches a triage result stays independent of the model vendor.

There is an unglamorous second constraint. A single key and single bill avoid credential sprawl across several dashboards and a month-end invoice pile. Infrai is one option that exposes one plain REST API across 295 capabilities in 20 modules, with no separate SDK required for each adjacent backend service. Its OpenAI-compatible surface keeps the triage handler stable, while its public, keyless discovery catalog is self-describing. For this workflow, the useful extra is the available-model catalog: an admin refresh can populate choices without hard-coding an imaginary list. The developer maintains one credential for the call path, and the operator reconciles one bill for it.

That breadth does not make every workload a match. Dedicated moderation is absent, so safety classification would need a chat model constrained by a JSON schema. Speech transcription is represented but currently unavailable, real-time voice sessions are pending and limited to the western region, and image upscaling supports Lanczos only. None of those limits block text ticket triage. They do block an assumption that one credential means every capability is interchangeable today.

This is the revenue-per-hour decision: outsource undifferentiated provider wiring, then spend the saved attention on routing rules, auditability, and the support experience. Ship weekly. Revisit the abstraction when a differentiated model feature earns its maintenance cost. The trade-off is explicit: I would accept a common request shape for this bounded classifier, but not if it erased a native feature that the product actually sells.

Keep that line sharp.

## The smallest working implementation

The example below keeps the model in an environment variable, uses the OpenAI TypeScript client against the compatible base URL, and validates the model's JSON instead of trusting prose. The SDK performs the explicit chat-completion operation, sends bearer authentication, and retries transient failures, including rate limits; `maxRetries` keeps the policy visible. API failures are surfaced with status and request identifiers rather than collapsed into an empty triage result.

Install `openai`, `express`, and `zod`, then set `INFRAI_API_KEY`, `INFRAI_BASE_URL`, and `TRIAGE_MODEL` in the server environment. The key must never reach browser code. `INFRAI_BASE_URL` is the service's versioned OpenAI-compatible base URL; keeping it in deployment configuration also makes the boundary visible during an infrastructure review.

```ts
import express, { type Request, type Response } from "express";
import OpenAI from "openai";
import { z } from "zod";

const apiKey = process.env.INFRAI_API_KEY;
const baseURL = process.env.INFRAI_BASE_URL;
const model = process.env.TRIAGE_MODEL;

if (!apiKey || !baseURL || !model) {
  throw new Error("INFRAI_API_KEY, INFRAI_BASE_URL, and TRIAGE_MODEL are required");
}

const client = new OpenAI({
  apiKey,
  baseURL,
  maxRetries: 4,
  timeout: 20_000,
});

const Ticket = z.object({
  id: z.string().min(1),
  subject: z.string().min(1).max(200),
  message: z.string().min(1).max(8_000),
});

const Triage = z.object({
  queue: z.enum(["device-support", "billing", "account", "general"]),
  priority: z.enum(["routine", "urgent"]),
  rationale: z.string().min(1).max(240),
});

const app = express();
app.use(express.json({ limit: "16kb" }));

app.post("/triage", async (req: Request, res: Response) => {
  const ticket = Ticket.parse(req.body);

  try {
    const completion = await client.chat.completions.create({
      model,
      temperature: 0,
      response_format: { type: "json_object" },
      messages: [
        {
          role: "system",
          content:
            "Route support tickets. Return JSON with queue, priority, and rationale. " +
            "Do not provide medical advice. Use urgent only when a person may need prompt human review.",
        },
        {
          role: "user",
          content: JSON.stringify(ticket),
        },
      ],
    });

    const content = completion.choices[0]?.message.content;
    if (!content) throw new Error("The model returned no triage result");

    const triage = Triage.parse(JSON.parse(content));
    res.status(200).json({ ticketId: ticket.id, model, triage });
  } catch (error) {
    if (error instanceof OpenAI.APIError) {
      console.error("Model API error", {
        status: error.status,
        requestId: error.request_id,
        message: error.message,
      });
      res.status(502).json({ error: "Triage provider request failed" });
      return;
    }

    console.error("Triage error", error);
    res.status(500).json({ error: "Ticket could not be triaged" });
  }
});

app.listen(3000, () => {
  console.log("Triage service listening on port 3000");
});
```

The route is intentionally boring.

Good. Swapping among supported models means changing `TRIAGE_MODEL`, not adding a new controller branch. In a production service, the allowed IDs should come from the available-only model catalog during startup or an authenticated admin refresh; the verified catalog shape includes `id`, `owned_by`, `capability`, and `available`. Do not copy a model name from a blog post and assume it is currently served.

The sample asks for a JSON object and validates it locally. A stronger deployment should use a provider-supported JSON schema where available, preserve the original ticket and triage result for review, and define a non-model fallback queue. No classification should silently disappear because an upstream call failed.

## How do the real options differ?

There are two architectural families, plus gateways with different priorities. Their interfaces overlap. Their reasons for existing do not.

| Option | Best fit | Portability cost | Important boundary |
| --- | --- | --- | --- |
| OpenAI SDK and API | A product built around OpenAI-specific platform features | Another provider requires an adapter or parallel code path | Direct access is an advantage when those features are the product |
| Anthropic TypeScript SDK | Claude-specific behavior and Anthropic's native Messages API matter | Request and response mapping remains application work | Native semantics should not be flattened merely for symmetry |
| Google Gen AI SDK | Gemini and Google's model tooling are the committed platform | Moving away means replacing native types and calls | Strong choice for a Google-centered system, weaker as a neutral boundary |
| OpenRouter | Broad model access through an OpenAI-like API is the main need | Some routing and provider metadata are gateway-specific | Evaluate privacy controls and provider selection for health-related text |
| Portkey | Gateway controls, observability, and routing policy are first-class requirements | The gateway becomes an operational component to configure | Better fit when a team needs policy infrastructure, not just a thin switch |
| Unified backend catalog | One key, one bill, a compatible chat surface, and broader backend capabilities reduce solo-operator overhead | The shared abstraction may not expose every native feature | Capability readiness varies; inspect discovery rather than assuming parity |

This is not a ranking. OpenAI, Anthropic, and Google are the cleanest choices when their native feature sets create product value. OpenRouter is closer to the model-access problem. Portkey is closer to the gateway-governance problem. A broader unified backend catalog can reduce administrative work for a one-person SaaS, but that breadth is useful only when public readiness data covers the required workflow.

The selection rule is concrete. Pick direct SDKs if at least one proprietary capability is part of the acceptance criteria. Pick a unified endpoint if the acceptance criteria say, "same validated triage contract, selectable supported model, one server-side credential." Then run a privacy, retention, region, and contractual review before any health-related or identifying ticket content crosses a third-party boundary. API compatibility does not answer compliance questions.

## What I would change at scale

First, move the model choice from a raw environment string into an authenticated admin setting backed by an allowlist. Refresh that allowlist from the model catalog on a controlled schedule, but keep the last known valid configuration if refresh fails. A dropdown is safer than free-form input. It also makes a model switch an observable configuration event.

Second, separate model output from dispatch. The model proposes `{ queue, priority, rationale }`; deterministic code validates it and enqueues the ticket. That boundary permits replay, human review, and a future rules-only path. It also prevents a creative response from becoming an operational instruction.

At larger volume, I would add a fixed evaluation set before adding routing cleverness. Use redacted examples that cover ambiguous billing language, device failures, account lockouts, and messages that require prompt human attention. Compare candidate models on the same schema-validity and routing rubric. Do not claim a winner from a handful of anecdotes, and do not switch models automatically on a price signal alone.

Finally, treat retries and side effects differently. Retrying a chat inference after a rate limit is reasonable and the SDK has bounded exponential retry behavior that respects retry guidance. Dispatching the resulting ticket is a write: give that step its own stable ticket ID and idempotency control so two successful inference attempts cannot create two assignments.

## Trade-offs and the decision boundary

Unified routing buys replaceability, smaller server code, and less credential administration. It also adds an intermediary, asks the team to accept a common denominator, and requires diligence about which upstream provider actually handles data. Direct SDKs expose native capabilities sooner and make the commercial relationship obvious, but each additional provider multiplies integration and operational work.

For this text-only healthtech triage service, **choose unified routing first**. The contract is small, model choice should remain reversible, and the founder's scarce resource is engineering attention. Unified catalogs, OpenRouter, and Portkey deserve evaluation inside that category for different reasons. A direct OpenAI, Anthropic, or Google integration wins when a native capability moves from "nice to have" to a tested product requirement.

That boundary keeps the decision honest. One API is an architectural convenience, not a substitute for model evaluation, data governance, or a human escalation path.

## References

- [OpenAI JavaScript library](https://github.com/openai/openai-node)
- [OpenAI structured outputs and function calling](https://platform.openai.com/docs/guides/function-calling)
- [Anthropic TypeScript SDK](https://github.com/anthropics/anthropic-sdk-typescript)
- [Google Gen AI JavaScript SDK](https://github.com/googleapis/js-genai)
- [OpenRouter API reference](https://openrouter.ai/docs/api-reference/overview)
- [Portkey AI Gateway documentation](https://portkey.ai/docs/product/ai-gateway)

## Sources

- https://github.com/openai/openai-node
- https://platform.openai.com/docs/guides/function-calling
- https://github.com/anthropics/anthropic-sdk-typescript
- https://github.com/googleapis/js-genai
- https://openrouter.ai/docs/api-reference/overview
- https://portkey.ai/docs/product/ai-gateway
