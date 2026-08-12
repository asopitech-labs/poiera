# Authorization and Agent Delegation: Grand Zero

Status: design baseline  
Baseline date: 2026-08-13  
Target MCP specification: 2026-07-28

## Purpose

This document is Poiera's starting point for authorization, agent-to-agent delegation, product-specific control, and delegated deployment. It records initial decisions, assumptions, feasibility questions, and experiments. It is not yet a stable protocol specification.

Poiera's goal is to let agents produce large, verifiable application changes from small, task-specific context without receiving broad or long-lived authority.

## Executive decision

Poiera will not inherit an inbound OAuth access token by forwarding it to another agent or product.

Poiera will instead:

1. terminate and validate the inbound token at the Poiera boundary;
2. preserve the human or workload identity as the subject;
3. identify every acting agent independently;
4. evaluate Application Contract, organization, plan, environment, and product policy;
5. derive a narrower, short-lived **Execution Mandate**;
6. return compact, product-specific **Control Directives** to the agent;
7. enforce the mandate in a Poiera Executor Gateway or exchange it for an audience-bound downstream token;
8. acquire separate product-native credentials for Theatora, Rhyzora, GitHub, cloud, registry, and runtime operations;
9. bind approvals and evidence to immutable digests.

OAuth establishes an enforceable authority boundary. Control Directives reduce agent context and guide planning. Directives never replace enforcement.

## Standards baseline

The initial design is based on:

- [MCP specification 2026-07-28](https://github.com/modelcontextprotocol/modelcontextprotocol/tree/main/docs/specification/2026-07-28)
- [MCP Authorization](https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/docs/specification/2026-07-28/basic/authorization/index.mdx)
- [MCP OAuth Client Credentials extension](https://modelcontextprotocol.io/extensions/auth/oauth-client-credentials)
- [MCP Enterprise-Managed Authorization extension](https://modelcontextprotocol.io/extensions/auth/enterprise-managed-authorization)
- [OAuth 2.0 Token Exchange, RFC 8693](https://www.rfc-editor.org/rfc/rfc8693.html)
- [OAuth 2.0 Rich Authorization Requests, RFC 9396](https://www.rfc-editor.org/rfc/rfc9396.html)
- [OAuth 2.0 Resource Indicators, RFC 8707](https://www.rfc-editor.org/rfc/rfc8707.html)
- [OAuth 2.0 Protected Resource Metadata, RFC 9728](https://www.rfc-editor.org/rfc/rfc9728.html)
- [Identity Assertion JWT Authorization Grant](https://datatracker.ietf.org/doc/draft-ietf-oauth-identity-assertion-authz-grant/)

MCP authorization protects HTTP transport access to an MCP server. It does not define a complete agent-to-agent delegation protocol or product-specific authorization model. Poiera supplies that application-level layer.

Extensions are optional and client support varies. Poiera must discover supported mechanisms and fail safely when the required mechanism is unavailable.

## Non-negotiable invariants

### Audience termination

An access token presented to Poiera must be intended for the canonical Poiera resource.

Poiera must not:

- accept a token intended for another resource;
- forward the inbound token to a child agent;
- forward the inbound token to Theatora, Rhyzora, GitHub, a cloud API, a registry, or another MCP server;
- treat an MCP routing header, tool name, or model instruction as proof of authorization.

### No privilege amplification

Each delegation can preserve or reduce authority, never increase it.

~~~text
Effective Authority
  = Subject authority
  ∩ Organization policy
  ∩ Calling actor authority
  ∩ Application Contract policy
  ∩ Approved plan
  ∩ Environment policy
  ∩ Delegate capability
  ∩ Product-native permissions
~~~

A child mandate must be a subset of its parent mandate.

### Independent actor identity

- **Subject**: the human or workload whose authority is exercised.
- **Actor**: the agent, worker, service, or executor performing an action.
- **Delegate**: a downstream actor receiving a derived mandate.

Every actor authenticates independently. A child agent should use workload identity, preferably an asymmetric JWT assertion or equivalent platform identity, rather than a shared long-lived secret.

### Enforcement over instruction

Agent-facing instructions are advisory planning context. Security boundaries are enforced by token validation, policy evaluation, mandate verification, plan-digest matching, approval checks, product adapter validation, and product-native authorization.

An agent ignoring a Control Directive must still be unable to perform a forbidden action.

### Explicit, stateless handles

MCP 2026-07-28 has no protocol-level session. Cross-call state must use explicit identifiers such as mandate ID, plan digest, approval ID, task handle, and evidence receipt ID.

Authorization decisions must not depend on hidden MCP session state.

## Trust model

~~~text
User or Enterprise IdP
          │
          │ Poiera-audience token
          ▼
Calling Agent ── independent client identity
          │
          ▼
Poiera MCP Resource Server
          │
          ├── Token Validator
          ├── Policy Decision Point
          ├── Mandate Issuer
          ├── Directive Projector
          └── Audit Recorder
                     │
                     ▼
              Execution Mandate
                     │
              Child Agent / Worker
                     │
                     ▼
          Poiera Executor Gateway
          ├── Theatora adapter
          ├── Rhyzora adapter
          ├── GitHub adapter
          ├── Cloud adapter
          └── Registry adapter
                     │
                     │ audience-bound product credential
                     ▼
                Product API
~~~

Poiera is a Resource Server for inbound MCP calls. It may contain a logical Security Token Service or integrate with an external Authorization Server, but it should not become a general-purpose identity provider by default.

## Supported identity cases

| Case | Authentication | Authority source | Poiera result |
|---|---|---|---|
| Human asks an agent to change an application | Standard MCP OAuth | User consent and AS policy | User-subject mandate |
| Enterprise agent acts for an employee | Enterprise-Managed Authorization and ID-JAG | Enterprise IdP plus local policy | Enterprise-subject mandate |
| CI/CD operates without a user | OAuth Client Credentials | Workload policy | Workload-subject mandate |
| Poiera starts a child worker | Child workload identity | Parent mandate and delegation policy | Narrow child mandate |
| Child calls a product through Poiera | Child identity plus mandate | Mandate and adapter policy | Poiera performs product call |
| Child must call a product directly | Product token exchange or STS | Product AS policy | Audience-bound short token |

## Two-plane model

### Authority plane

The enforceable plane contains OAuth tokens, identity assertions, workload identities, Execution Mandates, immutable plan digests, approvals, product-native tokens, and evidence receipts.

### Context plane

The compact agent plane contains allowed and denied actions, exact resources, constraints, approvals, budgets, evidence requirements, escalation instructions, and progressive-disclosure references.

The context plane is derived from authority and policy. It cannot grant authority.

## Execution Mandate

An Execution Mandate is Poiera's signed or server-held authorization artifact for a specific unit of work.

~~~json
{
  "mandate_id": "mdt_01K...",
  "issuer": "https://poiera.example",
  "subject": "user:alice",
  "actor": "agent:planner",
  "delegate": "agent:production-deployer",
  "tenant": "acme",
  "application": "support-desk",
  "environment": "production",
  "contract_digest": "sha256:...",
  "plan_digest": "sha256:...",
  "parent_mandate_id": "mdt_01J...",
  "allow": [
    "rhyzora.interface.generate",
    "theatora.infrastructure.plan",
    "theatora.infrastructure.apply"
  ],
  "deny": [
    "database.drop",
    "secret.read",
    "iam.modify"
  ],
  "resources": [
    "poiera://applications/support-desk/environments/production"
  ],
  "conditions": {
    "max_cost_delta_usd": 50,
    "max_destructive_changes": 0,
    "require_approval_for": [
      "public_endpoint",
      "irreversible_migration"
    ]
  },
  "issued_at": "2026-08-13T00:00:00Z",
  "expires_at": "2026-08-13T00:10:00Z",
  "nonce": "..."
}
~~~

Required properties include a short lifetime, explicit resource and environment, digest binding, subject/actor/delegate separation, parent reference, allow/deny/conditions, replay protection, revocation or rapid expiry, and stable audit correlation.

The first implementation should prefer an opaque, server-held handle for rapid revocation and simpler policy evolution. Signed offline mandates remain a later feasibility question.

## Control Directives

Control Directives are a compact projection of the effective mandate into product-specific instructions.

~~~json
{
  "decision": "conditional",
  "mandate_id": "mdt_01K...",
  "expires_at": "2026-08-13T00:10:00Z",
  "directives": {
    "github": {
      "allowed": ["create_branch", "commit_generated_files", "open_pull_request"],
      "forbidden": ["push_to_main", "force_push", "modify_branch_protection"],
      "constraints": {
        "repository": "acme/support-desk",
        "base_branch": "main",
        "required_checks": ["test", "policy", "preview"]
      }
    },
    "theatora": {
      "allowed": ["plan", "apply"],
      "constraints": {
        "environment": "production",
        "max_destructive_changes": 0
      }
    },
    "rhyzora": {
      "allowed": ["validate", "generate"],
      "constraints": {
        "contract_digest": "sha256:..."
      }
    }
  },
  "approval_required": [
    {"condition": "public_endpoint_added", "approval_class": "security"}
  ],
  "evidence_required": [
    "git_commit",
    "deployment_id",
    "contract_diff",
    "verification_report"
  ]
}
~~~

Directives should be deterministic, versioned, hashable, and reproducible from the same mandate and policy revision. Product adapters must enforce the corresponding constraints independently.

## Delegation flows

### Human-delegated operation

1. The agent obtains a Poiera-audience token through standard MCP OAuth.
2. Poiera validates issuer, audience, expiration, client, scopes, and resource.
3. Poiera resolves the user subject and calling-agent actor.
4. Poiera plans and evaluates the requested change.
5. Approval is bound to the immutable plan digest.
6. Poiera creates an Execution Mandate.
7. A worker authenticates with its own workload identity.
8. Poiera derives a narrower worker mandate.
9. The Executor Gateway performs product calls with separate credentials.
10. Evidence records subject, actor chain, mandate, plan, and release.

### Enterprise-managed operation

1. The agent authenticates to the enterprise IdP.
2. It obtains an ID-JAG for the Poiera authorization domain.
3. The Poiera AS validates the ID-JAG and applies local policy.
4. It issues a Poiera-audience access token.
5. Poiera follows the normal mandate flow.

ID-JAG conveys delegated user identity across a trust domain. It does not authorize arbitrary onward delegation by Poiera.

### Workload operation

1. CI or a worker authenticates with OAuth Client Credentials.
2. Poiera treats the workload as subject unless a verified user delegation is present.
3. Workload policy limits application, environment, action, duration, and budget.
4. Poiera derives an Execution Mandate.
5. Production actions may still require a separate approval artifact.

### Child-agent delegation

1. Parent actor requests a delegate and reduced permissions.
2. Delegate authenticates independently.
3. Poiera verifies that child authority is a subset of the parent.
4. Poiera issues a shorter or equal-lived child mandate.
5. Audit preserves subject, actors, parent mandate, and policy decision.
6. Recursive delegation is denied unless explicitly allowed.

## Downstream product access

Preferred order:

1. Poiera Executor Gateway obtains product credentials and performs the operation.
2. Product-native token exchange or STS issues an audience-bound short credential.
3. A narrowly scoped installation or workload identity is used by the executor.
4. Static credentials are a compatibility fallback and remain in the credential broker.

Agents receive credential handles, not secret material, whenever possible.

Product access must not be inferred from OAuth scopes alone. Each adapter validates the exact repository, account, project, environment, region, branch, resource, and operation.

## MCP surface

Initial conceptual tools:

~~~text
poiera.authority.inspect
poiera.plan
poiera.delegation.request
poiera.mandate.derive
poiera.dispatch
poiera.approval.respond
poiera.task.get
poiera.verify
poiera.receipt.get
~~~

Expected behavior:

- transport-scope failures use the MCP/OAuth scope challenge;
- application-authority failures return a structured denial;
- approval or missing information can use Multi Round-Trip Requests;
- long-running releases use the MCP Tasks extension;
- tools return explicit handles rather than relying on a session;
- permission-sensitive lists use authorization-safe caching.

## Policy inputs and outputs

Inputs may include OAuth claims, tenant/group/role claims, workload identity, Application Contract effects and requirements, desired and observed state, semantic change, plan digest, environment, product inventory, cost/risk budgets, approvals, delegate capability, time, incident mode, revocation state, and policy version.

Output must include the decision, reasons, effective permissions, unmet requirements, mandate parameters, directive projection, and evidence requirements.

## Threats to test

- token passthrough and audience confusion;
- confused deputy behavior;
- privilege amplification;
- subject/actor confusion;
- mandate theft and replay;
- stale approval after plan change;
- policy change after mandate issue;
- adapter parameter smuggling;
- prompt injection attempting to override directives;
- unauthorized recursive delegation;
- credential leakage into model context, logs, or evidence;
- cross-tenant confusion;
- cached authorization data crossing subjects;
- compromised worker identity;
- rollback authority exceeding deployment authority;
- partial execution after expiry or revocation.

## Feasibility questions

### MCP and clients

- Which target clients support MCP 2026-07-28?
- Which support Client Credentials and Enterprise-Managed Authorization?
- How is extension discovery exposed in Tier 1 SDKs?
- Can MRTR carry approval and step-up flows reliably?
- Can Tasks represent progress, cancellation, and recovery?

### Authorization servers

- Which identity platforms implement ID-JAG?
- Which implement RFC 8693 with subject and actor semantics?
- Can RFC 9396 carry Poiera details interoperably?
- How do revocation and policy changes affect issued mandates?
- Should Poiera embed an AS, integrate one, or support both?

### Product adapters

For each target product:

- Does it support token exchange, workload identity, installation tokens, or short credentials?
- What is the smallest enforceable native permission?
- Can credentials be resource- and environment-bound?
- Can destructive operations be blocked independently?
- What dry-run, idempotency, audit, and evidence facilities exist?
- Can credentials remain inside the Executor Gateway?

### Mandates and context

- Opaque handle or signed document?
- Central introspection or local verification?
- Maximum lifetime and delegation depth?
- Immediate revocation and replay prevention?
- Partial completion and retry semantics?
- What minimum directive schema preserves task success?
- Which facts are included versus progressively disclosed?
- Can model-specific projections preserve identical authority?

## Feasibility experiments

### F0 — Audience boundary

Prove Poiera and downstream tokens are mutually rejected and no inbound token appears downstream or in logs.

### F1 — Workload identity

Connect a CI worker with the Client Credentials extension and signed JWT assertion. Verify rotation, expiry, replay rejection, and actor attribution.

### F2 — Subject and actor

Execute a user-delegated request through a separately authenticated worker and preserve both identities through evidence.

### F3 — Derived mandate

Derive a child mandate and reject attempts to add actions, resources, lifetime, environment, or delegation depth.

### F4 — Executor isolation

Run a GitHub or preview adapter through the Executor Gateway while the agent receives only a mandate handle and directives.

### F5 — Plan-bound approval

Approve one digest, change a plan parameter, and reject execution until the new digest is approved.

### F6 — Product directives

Generate GitHub, Theatora, and Rhyzora directives from one mandate. Measure context reduction and task success while enforcing forbidden operations.

### F7 — Enterprise-managed flow

Validate ID-JAG exchange, local permission mapping, step-up, revocation, and tenant isolation with one IdP and client.

### F8 — Long-running release

Represent build, deploy, verify, cancellation, and recovery through Tasks, with MRTR for conditional approval.

## Initial implementation boundary

The first implementation should include:

- MCP 2026-07-28 over HTTP;
- one user OAuth flow;
- Client Credentials for workers;
- Poiera as Resource Server and policy decision point;
- opaque, server-held Execution Mandates;
- delegation depth of one;
- one Executor Gateway;
- GitHub plus preview deployment adapters;
- Theatora and Rhyzora directive projections;
- immutable plan and approval digests;
- audit and evidence receipts;
- no product tokens in agent context.

Enterprise-Managed Authorization remains a feasibility track until target IdP and client support is confirmed.

## Deferred decisions

Not fixed by Grand Zero:

- authorization server implementation;
- mandate wire format;
- RFC 9396 usage;
- direct child-agent token exchange;
- signed offline mandates;
- multi-level delegation;
- cross-organization federation;
- production credential providers;
- policy language and engine;
- final MCP tool names;
- audit retention and privacy policy.

Each decision should become an ADR with threat analysis, interoperability evidence, and fallback behavior.

## Exit criteria

Grand Zero can advance to a normative specification when:

1. F0 through F6 have working prototypes and evidence.
2. One human flow and one workload flow interoperate.
3. Subject and actor attribution survive the full release.
4. A child cannot broaden a parent mandate.
5. No credential enters model-visible context.
6. Adapters enforce resource and action constraints independently.
7. Approval invalidates when the plan digest changes.
8. Revocation and expiry stop new work predictably.
9. Directives materially reduce context without reducing task success.
10. Unsupported extensions fail closed or have an explicit fallback.

## Working definition

> Poiera is an application-level delegation control plane that converts verified identity, Application Contract policy, and an approved immutable plan into a minimal Execution Mandate and compact product-specific Control Directives, then enforces the mandate through audience-bound product adapters and records verifiable evidence.
