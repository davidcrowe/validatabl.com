---
layout: default
title: validatabl
---

# validatabl

**fine-grained access control and policy enforcement for llm and agentic apps (apps sdk + mcp)**

once a request is tied to a real user and tenant (via `identifiabl`) and its content is safely preprocessed (via `transformabl`), the next question is simple:

**is this request allowed?**

validatabl answers that question.

it is the policy engine of gatewaystack — enforcing permissions, scopes, org-level rules, schemas, and safety constraints with user-level precision.

## at a glance

validatabl is a **user-scoped policy enforcement layer** for llm apps.  

it lets you:

- enforce per-user and per-org model/tool permissions  
- apply schema, safety, and boundary checks  
- define tenant and data-isolation rules  
- implement scopes and role-based policies  
- reject, modify, or reshape unsafe or disallowed requests  

> 📦 **implementation**: [`ai-policy-gateway`](https://github.com/davidcrowe/gatewaystack) (roadmap)

## why now?

as orgs shift to agentic systems using personal data, tools, and workflows, they need to ensure that:

- only the right users call the right models  
- only the right tools are accessible for a given role or scope  
- sensitive operations follow policy and regulatory rules  
- regulated data stays within its boundary  
- governance decisions are logged and explainable  

shared api keys cannot express or enforce this.

validatabl brings **real authorization** to llm systems — something the industry has been missing.

## designing access control for ai

validatabl evaluates every request **after identity is verified** but **before routing and execution**.

### within the shared requestcontext

all gatewaystack modules operate on a shared [`RequestContext`](https://github.com/davidcrowe/gatewaystack/blob/main/docs/reference/interfaces.md) object.

**validatabl is responsible for**:

- **reading**: `identity` (from identifiabl), `content` and `metadata` (from transformabl), `modelRequest` (requested model, tools, params), policy definitions
- **writing**: `policyDecision` — structured authorization outcome (`allow` | `deny` | `modify` + reasons + modifications)

the `policyDecision` determines whether the request proceeds, is blocked, or is modified before reaching the model.

### evaluation principles

validatabl provides:

- deterministic policy evaluation  
- user → role → scope → permission resolution  
- schema and safety rule checks  
- enforcement for both human- and agent-initiated calls  
- structured decision outputs for downstream modules  

## the core functions

**1. `checkPermissions` — verify user/model/tool access**  
ensures a user or agent can call a specific model, tool, or operation.

**2. `checkScopes` — enforce scope boundaries**  
validates that requested actions fall within granted scopes (similar to oauth).

**3. `checkSchema` — validate request structure**  
ensures tool calls, structured payloads, and params conform to expected schemas.

**4. `checkSafety` — evaluate content against safety rules**  
applies safety policies on transformed content (categories, risk, sensitivity).

**5. `applyPolicies` — run org- and tenant-wide governance rules**  
regulatory constraints, internal controls, business rules, environment-specific policies.

**6. `decision` — return `allow`, `deny`, or `modify`**  
returns a structured decision object and, when configured, can modify the request  
(for example, stripping fields, masking values, or downgrading access).

## what validatabl does

- enforces model and tool permissions  
- applies scopes and roles  
- validates schema and structural constraints  
- evaluates safety rules on transformed inputs  
- isolates tenants and data boundaries  
- returns allow/deny/modify decisions for every request  
- populates the `policyDecision` field in `RequestContext`  

## validatabl works with

- `identifiabl` to authenticate users
- `transformabl` to preprocess content
- `limitabl` to apply rate limits or quotas
- `proxyabl` to perform provider routing
- `explicabl` to store or ship audit logs

## validatabl works with

- `identifiabl` to verify identity  
- `transformabl` to analyze and classify content  
- `limitabl` to enforce quotas or spend based on policy outcomes  
- `proxyabl` to route or proxy traffic using the `policyDecision`  

## defining policies

validatabl policies are defined in yaml and evaluate identity, content, and metadata:

```yaml
policies:
  - name: "restrict-medical-models"
    priority: 1
    condition: |
      user.role != "physician" AND 
      request.model in ["gpt-4-medical", "claude-medical"]
    action: deny
    reason: "Medical models require physician role"

  - name: "block-pii-for-contractors"
    priority: 2
    condition: |
      user.type == "contractor" AND 
      content.metadata.contains_pii == true
    action: deny
    reason: "Contractors cannot process PII"

  - name: "downgrade-free-tier"
    priority: 3
    condition: |
      user.tier == "free" AND 
      request.model == "gpt-4"
    action: modify
    modification:
      model: "gpt-3.5-turbo"
```

## policy management

policies are defined declaratively (yaml) and loaded at startup. gatewaystack compiles them into an internal representation for fast evaluation.

- **hot-reload**: policy files can be reloaded without redeploying the gateway  
- **versioning**: policies are versioned so `explicabl` can record "which policy version applied?"  
- **storage**: policies can be stored in git, object storage, or a config service  

## example policy scenarios
```yaml
# policy 1: allow only licensed doctors to use medical models
- name: "medical-model-access"
  condition: user.role == "doctor" AND user.licensed == true
  action: allow
  models: ["gpt-4-medical", "claude-medical"]

# policy 2: deny non-admin users from sending pii
- name: "pii-block-non-admin"
  condition: user.role != "admin" AND content.contains_pii == true
  action: deny
  reason: "Non-admin users cannot send PII"

# policy 3: remove attachments for basic tier users
- name: "tier-based-attachments"
  condition: user.tier == "basic" AND request.has_attachments
  action: modify
  modification: remove_attachments
```

## policy evaluation model

validatabl uses a **deny-by-default, priority-ordered** evaluation:

1. policies are sorted by priority (1 = highest)  
2. each policy condition is evaluated against request context  
3. the first matching `deny` or `allow` wins  
4. `modify` actions are applied cumulatively  
5. if no policies match, the request is denied  

this ensures security by default while allowing granular control.

## modification actions

**what does `modify` actually do?**

policies that return `modify` can adjust the request while still allowing it to proceed. common modification actions include:

- stripping fields from the request  
- redacting content further  
- downgrading model selection (for example, gpt-4 → gpt-3.5)  
- reducing token limits  
- removing tool access  
- adding warning or annotation headers  

**example:**

> a free-tier user requests gpt-4.  
> a policy modifies the request to use gpt-3.5 instead.  
> the request continues, but with a cheaper model and within plan limits.

modification actions are recorded in `policyDecision.modifications` so downstream modules and `explicabl` can reconstruct "what changed and why."

## scope system

validatabl enforces oauth-style scopes:

**common scopes:**

- `models:gpt-4` — can access gpt-4  
- `tools:calendar` — can use calendar tool  
- `data:read:org` — can read org-level data  
- `data:write:user` — can write user-scoped data  

**scopes are granted via:**

- identity provider (oidc token claims)  
- role mappings (defined in gatewaystack config)  
- explicit user grants (consent flows)  

scopes become part of the `identity` section of `RequestContext` and are evaluated alongside roles and tenant metadata.

## performance and caching

validatabl evaluates policies on every request. to keep latency low:

- policies are compiled at load time into an optimized evaluation tree  
- decisions may be cached per `(user, model, tool, scope)` tuple for a short ttl (for example, 30–60s), when safe  
- external lookups (for example, org settings) are protected behind small, configurable caches  

the default configuration prioritizes correctness and safety; caching is opt-in per policy or per tenant.

## end to end flow
```text
user
   → identifiabl       (who is calling?)
   → transformabl      (prepare, clean, classify, anonymize)
   → validatabl        (is this allowed?)
   → limitabl          (how much can they use? pre-flight constraints)
   → proxyabl          (where does it go? execute)
   → llm provider      (model call)
   → [limitabl]        (deduct actual usage, update quotas/budgets)
   → explicabl         (what happened?)
   → response
```

validatabl sits at the **governance checkpoint** of gatewaystack — enforcing rules before traffic enters the llm layer.

## input context

validatabl receives enriched request context (simplified view of `RequestContext`):

```javascript
{
  identity: {  // from identifiabl
    user_id: "user_123",
    org_id: "org_456",
    roles: ["engineer"],
    scopes: ["models:gpt-4", "tools:*"]
  },
  content: {  // original + transformed
    messages: [...],
    attachments: [...]
  },
  metadata: {  // from transformabl
    contains_pii: false,
    classification: ["technical"],
    risk_score: 0.2,
    topics: ["code", "debugging"]
  },
  modelRequest: {  // requested model + tools
    model: "gpt-4",
    tools: ["web_search"],
    max_tokens: 2000
  }
}
```

policies can reference any field in this context.

## use cases

**example 1: healthcare compliance**  
a hospital uses gatewaystack to ensure only licensed physicians can access medical diagnosis models, and all requests containing patient identifiers are logged for hipaa compliance. validatabl enforces:

```text
if user.role != "physician" then deny access to medical models
```

**example 2: multi-tenant saas**  
a crm platform ensures sales reps can only access ai features for their own customers. validatabl enforces:

```text
if request.customer_id not in user.assigned_customers then deny
```

**example 3: cost control**  
a startup allows free-tier users to access gpt-3.5 but requires paid plans for gpt-4. validatabl enforces:

```text
if user.tier == "free" and request.model == "gpt-4" then modify to "gpt-3.5"
```

## integrates with your existing stack

validatabl plugs into gatewaystack and your existing llm stack without requiring application-level changes. it exposes http middleware and sdk hooks for:

- chatgpt apps sdk  
- model context protocol (mcp)  
- oauth2 / oidc identity providers  
- any llm provider (openai, anthropic, google, internal models)  

## getting started

**for policy examples and patterns**:  
→ [policy examples library](https://github.com/davidcrowe/gatewaystack/blob/main/docs/examples/policy-examples.md)  
→ [policy testing guide](https://github.com/davidcrowe/gatewaystack/blob/main/docs/guides/testing-policies.md)

**for implementation**:  
→ [integration guide](https://github.com/davidcrowe/gatewaystack/blob/main/docs/guides/integration.md)

## links

want to explore the full gatewaystack architecture?  
→ [view the gatewaystack github repo](https://github.com/davidcrowe/gatewaystack)

want to contact us for enterprise deployments?  
→ [reducibl applied ai studio](https://reducibl.com)

<div class="arch-diagram">
  <div class="arch-row">
    <div class="arch-node">
      <div class="arch-node-title">app / agent</div>
      <div class="arch-node-sub">chat ui · internal tool · agent runtime</div>
    </div>

    <div class="arch-arrow">→</div>

    <div class="arch-node arch-node-gateway">
      <div class="arch-node-title">gatewaystack</div>
      <div class="arch-node-sub">user-scoped trust &amp; governance gateway</div>

      <div class="arch-pill-row">
        <span class="arch-pill">identifiabl</span>
        <span class="arch-pill">transformabl</span>
        <span class="arch-pill">validatabl</span>
        <span class="arch-pill">limitabl</span>
        <span class="arch-pill">proxyabl</span>
        <span class="arch-pill">explicabl</span>
      </div>
    </div>

    <div class="arch-arrow">→</div>

    <div class="arch-node">
      <div class="arch-node-title">llm providers</div>
      <div class="arch-node-sub">openai · anthropic · internal models</div>
    </div>
  </div>

  <p class="arch-caption">
    every request flows from your app through gatewaystack's modules before it reaches an llm provider —
    <strong>identified, transformed, validated, constrained, routed, and audited.</strong>
  </p>
</div>