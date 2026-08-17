# Poiera

**From application intent to verified release.**

Poiera is an agent-native application control plane that enables AI agents to understand, change, build, deploy, and verify applications through compact semantic contracts.

**[Explore the Poiera concept site](https://asopitech-labs.github.io/poiera/)**

Instead of requiring an agent to repeatedly inspect an entire repository, infrastructure configuration, interface implementation, and deployment pipeline, Poiera presents the smallest relevant view of an application and provides the authority and execution machinery needed to act on it.

```text
                         Human / AI Agent
                                │
                       MCP / CLI / API / SDK
                                │
                                ▼
                    ┌───────────────────────┐
                    │        Poiera         │
                    │                       │
                    │  Context Projection   │
                    │  Change Planning      │
                    │  Policy & Delegation  │
                    │  Build & Deployment   │
                    │  Verification         │
                    └───────────┬───────────┘
                                │
                 ┌──────────────┴──────────────┐
                 │                             │
          Theatora Driver                Rhyzora Driver
                 │                             │
       Backend composition           Interface projection
       Modules and providers         CLI, TUI, Web, MCP
                 │                             │
                 └──────────────┬──────────────┘
                                ▼
                     Verified Application
```

## Why Poiera?

Application development is shifting from humans directly editing every implementation detail to agents working on behalf of humans.

Agents can already read code and invoke development tools. The difficult part is giving them:

- enough application context without loading an entire system,
- a stable semantic model instead of disconnected files,
- narrowly delegated authority instead of long-lived credentials,
- a safe plan for changes spanning multiple projects,
- a way to build and deploy the exact approved change,
- and evidence that the resulting application behaves as declared.

Poiera provides that boundary.

## Follow and participate

Poiera is being shaped in the open. You do not need to write code to help define the boundary for trustworthy agent-led delivery.

- **[Star the repository](https://github.com/asopitech-labs/poiera)** to save the project and help others discover it.
- **[Watch the repository](https://github.com/asopitech-labs/poiera/subscription)** to follow the activity that matters to you.
- **[Share the concept site](https://asopitech-labs.github.io/poiera/)** with people working on AI agents, developer tools, or delivery systems.
- **[Join the discussion](https://github.com/asopitech-labs/poiera/discussions)** to propose a use case, question the trust model, or share a counterexample.
- **[Open an issue](https://github.com/asopitech-labs/poiera/issues/new/choose)** for a concrete feature, bug, driver, or implementation task.

## Core Idea

Poiera treats an application as four related models:

```text
Semantic Contract
    What the application can do

Desired State
    What should be built and deployed

Observed State
    What is currently running

Change Set
    How the application should change
```

These models allow agents to work with application meaning before descending into source code, infrastructure, or raw logs.

```text
Semantic Contract ─┐
Desired State      ├── Reconciliation ── Execution Plan
Observed State     ┘                         │
                                             ▼
                                  Build / Deploy / Verify
```

## Compact Agent Context

Poiera does not send the entire application definition to every agent.

It produces task-specific contract slices containing only the resources, operations, policies, interfaces, deployments, and constraints relevant to the current intent.

```console
poiera inspect \
  --intent "add ticket escalation" \
  --format agent
```

Conceptual output:

```yaml
focus:
  resource: Ticket

current:
  fields:
    id: string
    status: [open, assigned, closed]
    assignee: User?

  operations:
    - assignTicket
    - closeTicket

affected:
  theatora:
    module: Tickets
    capabilities:
      - RelationalStore
      - EventBus

  rhyzora:
    application: OperatorConsole
    interfaces:
      - web
      - tui
      - mcp

constraints:
  - production deployment requires approval
  - database changes must remain backward compatible
  - destructive operations require explicit confirmation
```

Agents can progressively request deeper information when it becomes necessary.

```text
Level 0  Application summary
Level 1  Relevant resources and operations
Level 2  Full semantic definitions
Level 3  Implementation bindings
Level 4  Source locations and generated artifacts
Level 5  Logs and runtime evidence
```

## Application Contract

The Application Contract is Poiera's portable description of application behavior.

It is independent of backend implementation, interface technology, cloud provider, and deployment environment.

```yaml
contract: poiera.application/v1
name: support-desk
version: 1.4.0

resources:
  Ticket:
    identity: id

    fields:
      id: string
      title: string
      status:
        enum: [open, assigned, closed]

operations:
  closeTicket:
    input:
      ticketId: string

    output: Ticket

    errors:
      - TicketNotFound
      - PermissionDenied

    effect: destructive

    authorization:
      requires: tickets.close

events:
  TicketClosed:
    data:
      ticket: Ticket
```

A contract describes:

- types and resources,
- operations and workflows,
- inputs, outputs, and errors,
- effects and idempotency,
- authentication and authorization requirements,
- events and subscriptions,
- pagination and streaming,
- invocation bindings,
- and compatibility constraints.

It does not prescribe UI layout, databases, cloud products, or business-logic implementation.

## Semantic Change Sets

Agents propose application changes as semantic operations rather than unrelated file edits.

```yaml
change:
  intent: Add ticket escalation

  contract:
    addField:
      resource: Ticket
      field:
        name: escalationLevel
        type: integer
        default: 0

    addOperation:
      name: escalateTicket
      effect: write

    addEvent:
      name: TicketEscalated

  experience:
    exposeAction:
      application: OperatorConsole
      resource: Ticket
      operation: escalateTicket
      interfaces:
        - web
        - tui
        - mcp
```

Poiera validates the change, determines its impact, and expands it into an executable plan.

```text
1. Validate the semantic change
2. Check contract compatibility
3. Resolve affected implementations
4. Plan backend changes
5. Plan interface changes
6. Generate build and migration steps
7. Evaluate policies
8. Request required approvals
9. Build immutable artifacts
10. Deploy the approved plan
11. Verify declared behavior
12. Record evidence and observed state
```

## Theatora and Rhyzora

Poiera acts as the agent-facing control plane across Theatora and Rhyzora.

### Theatora

Theatora defines and realizes backend behavior from capabilities, modules, providers, bindings, and runtimes.

```text
Poiera Change Set
        │
        ▼
Theatora Driver
├── resolve affected modules
├── resolve required capabilities
├── generate configuration
├── plan database migrations
├── build backend artifacts
├── deploy runtimes
└── verify contract conformance
```

### Rhyzora

Rhyzora defines application interactions and projects them into multiple interfaces.

```text
Poiera Change Set
        │
        ▼
Rhyzora Driver
├── detect affected interactions
├── validate operation bindings
├── update application projections
├── build interface artifacts
├── run interaction tests
└── deploy interfaces
```

A single operation may be exposed through several interfaces:

```text
escalateTicket
├── CLI command
├── TUI action
├── Web interaction
├── Desktop interaction
└── MCP tool
```

Neither project is required to use the other.

Poiera connects them through portable contracts and drivers while allowing both to remain independently useful.

## Agent Gateway

Agents can connect to Poiera through:

- MCP,
- CLI,
- HTTP API,
- language SDKs,
- or CI integrations.

The high-level operation surface is intentionally small.

```text
discover
inspect
propose
plan
authorize
apply
verify
observe
rollback
```

An agent does not need direct access to every cloud provider, deployment system, registry, and runtime API.

```text
Agent
  │
  │ apply approved release
  ▼
Poiera
  ├── Theatora deployment
  ├── Rhyzora builds
  ├── database migrations
  ├── provider operations
  ├── behavioral verification
  └── evidence collection
```

## Delegated Authority

Poiera does not expose long-lived infrastructure credentials to agents.

It brokers short-lived, task-specific execution identities governed by explicit policy.

```yaml
delegation:
  principal: agent:release-worker
  application: support-desk
  environment: preview

  actions:
    - build
    - deploy
    - verify

  denied:
    - delete-data
    - read-secrets
    - change-identity-provider

  expiresIn: 20m

  budget:
    maxCostIncrease: 5%

  approval:
    production: required
```

Delegation can be constrained by:

- application,
- environment,
- action,
- resource,
- duration,
- cost,
- risk,
- and approved plan digest.

```text
Human or CI Identity
          │
          │ Poiera-audience OAuth token
          ▼
       Poiera
          │
          ├── evaluates policy
          ├── derives a short-lived Execution Mandate
          └── returns compact Control Directives
                         │
                         ▼
                Execution Agent
                  (self-authenticated)
                         │
                         ▼
               Poiera Executor Gateway
                         │
                         │ separate product credential
                         ▼
              Cloud / Runtime / Registry
```

The inbound token terminates at Poiera. It is never forwarded to another agent or product API. Delegated execution is represented by a narrower, short-lived mandate; product credentials are acquired separately and remain bound to their intended audience.

The design baseline, protocol mapping, feasibility questions, and validation plan are maintained in [Authorization and Agent Delegation: Grand Zero](docs/authorization-grand-zero.md).

## Immutable Plans and Approval

Planning and execution are separate operations.

```text
Agent proposes a change
          │
          ▼
Poiera creates an immutable plan
          │
          ├── affected components
          ├── compatibility impact
          ├── migrations
          ├── policy changes
          ├── cost impact
          ├── deployment strategy
          └── rollback capability
          │
          ▼
Approval is bound to the plan digest
          │
          ▼
Poiera applies exactly that plan
```

If the plan changes, its approval is no longer valid.

```yaml
plan:
  id: release-01K...
  digest: sha256:...

  environment: production

  changes:
    contract: additive
    database: additive-migration
    backend: canary-replacement
    web: rolling-update
    mcp: add-tool

  risk:
    level: medium
    reasons:
      - production database migration
      - new write operation

  rollback:
    backend: automatic
    interfaces: automatic
    database: forward-recovery-only
```

## Build and Deployment

Poiera builds a release graph across all affected projects.

```text
Contract Validation
        │
        ├── Compatibility Analysis
        ├── Policy Evaluation
        └── Impact Analysis
        │
        ▼
Cross-project Build Graph
        │
        ├── Theatora modules
        ├── backend runtimes
        ├── database migrations
        ├── Rhyzora applications
        └── interface artifacts
        │
        ▼
Release Plan
        │
        ├── build
        ├── preview
        ├── approve
        ├── deploy
        ├── verify
        └── promote or rollback
```

Existing CI systems can invoke Poiera as thin execution hosts.

```yaml
steps:
  - run: poiera reconcile
  - run: poiera plan --environment production
  - run: poiera apply --approved-plan "$PLAN_DIGEST"
  - run: poiera verify --release current
```

## Verification and Evidence

A successful command is not sufficient evidence of a successful release.

Poiera verifies that deployed behavior conforms to the declared contract.

```text
Operation declared
       │
       ▼
Binding reachable?
       │
       ▼
Authorization enforced?
       │
       ▼
Input and output conform?
       │
       ▼
Expected event emitted?
       │
       ▼
Interfaces connected?
```

Every release produces a compact evidence receipt.

```yaml
release:
  id: release-01K...
  status: verified

contract:
  version: 1.4.0
  conformance: passed

theatora:
  deployment: healthy
  migration: applied
  modules:
    Tickets: verified

rhyzora:
  web: verified
  tui: built
  mcp: verified

checks:
  passed: 42
  failed: 0

observedStateDigest: sha256:...
evidence: evidence:release-01K...
```

Raw logs and detailed artifacts remain available by reference and are only added to agent context when requested.

## CLI

The intended workflow is:

```console
# Discover the application
poiera inspect

# Request a task-specific view
poiera inspect --intent "add ticket escalation"

# Validate a semantic change
poiera propose change.yaml

# Create an immutable execution plan
poiera plan --environment preview

# Apply the plan with delegated authority
poiera apply --plan release-01K...

# Verify deployed behavior
poiera verify --release release-01K...

# Promote the verified release
poiera promote release-01K... --to production
```

## Operating Modes

Poiera is designed to run in three forms.

### Local

For developers and local coding agents:

```console
poiera inspect
poiera plan --environment local
poiera apply
```

### CI Agent

For automated build and release workflows:

```console
poiera reconcile --ci
poiera verify --release current
```

### Control Plane

For persistent environments requiring:

- observed-state management,
- delegated credentials,
- approvals,
- production deployment,
- drift detection,
- agent connections,
- audit records,
- and release evidence.

## Driver Model

Poiera is extensible through drivers.

```text
Drivers
├── Theatora
├── Rhyzora
├── OpenAPI
├── MCP
├── native libraries
├── container runtimes
├── cloud platforms
└── CI systems
```

Drivers translate between portable Poiera concepts and implementation-specific operations.

They do not redefine the semantic contract.

## Project Structure

The project is expected to contain:

```text
poiera/
├── specification
│   ├── semantic-contract
│   ├── desired-state
│   ├── observed-state
│   └── change-set
│
├── context-engine
│   ├── contract-slicing
│   ├── dependency-index
│   └── progressive-disclosure
│
├── control-plane
│   ├── planner
│   ├── reconciler
│   ├── release-coordinator
│   └── verifier
│
├── agent-gateway
│   ├── mcp
│   ├── cli
│   ├── api
│   └── sdk
│
├── security
│   ├── policy
│   ├── delegation
│   ├── credential-broker
│   └── audit
│
└── drivers
    ├── theatora
    └── rhyzora
```

## Non-goals

Poiera is not:

- a general-purpose AI agent framework,
- a replacement for source control,
- a new cloud provider,
- a UI framework,
- a backend framework,
- or a system that gives agents unrestricted production access.

Poiera coordinates existing implementations through explicit contracts, constrained authority, immutable plans, and verifiable results.

## Initial Milestone

The first end-to-end milestone is intentionally narrow:

```text
Agent request
    "Add ticket escalation"
             │
             ▼
     Poiera Change Set
             │
       ┌─────┴─────┐
       ▼           ▼
   Theatora     Rhyzora
   operation    CLI action
   migration    TUI action
   deployment   MCP tool
       │           │
       └─────┬─────┘
             ▼
      Poiera Verification
             │
             ▼
       Evidence Receipt
```

The milestone is complete when an agent can:

1. inspect a task-specific contract slice,
2. propose a semantic change,
3. produce a cross-project release plan,
4. deploy to a preview environment using delegated credentials,
5. verify contract conformance,
6. receive a compact evidence receipt,
7. and promote the same immutable release to production after approval.

## Vision

Agents should not need unlimited context or unlimited authority to produce meaningful software changes.

They need:

- a compact model of application meaning,
- a precise description of the requested change,
- constrained authority to perform it,
- an execution system that coordinates the affected components,
- and trustworthy evidence of the result.

Poiera provides that application-level boundary.

**Describe the intent. Approve the plan. Verify the result.**
