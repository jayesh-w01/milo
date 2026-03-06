# Domain-Driven Design Principles
## Reference for Milo's solution-architecture workflow

---

## Core Concepts

### Bounded Context

A bounded context is the primary unit of modularity in Milo. It is a boundary within which a particular domain model applies — where the language, the concepts, and the rules are consistent and unambiguous.

**Defining a bounded context:**
- One team owns it
- One codebase implements it
- One ubiquitous language describes it
- It can evolve independently of other contexts

**The test for a good bounded context:**
Can you describe what this module does in one sentence using domain language (not technical language), and does everything inside it relate to that sentence?

**Common mistake:** defining boundaries around technical concerns ("the database layer", "the API layer") rather than domain concerns. Always carve by business domain, not by technology.

---

### Ubiquitous Language

The language used inside a bounded context. Every concept, entity, and operation has a specific meaning that all stakeholders — developers, product managers, business — agree on.

**Why it matters for Milo:**
Module names, function names, and type names should reflect the ubiquitous language of their domain. A function called `processItem` is worse than one called `placeOrder` or `verifyIdentity`. The code should read like the business.

---

### Aggregates

An aggregate is a cluster of domain objects treated as a single unit for data changes. Every aggregate has a root — the only entry point for external access.

**In Milo's context:** an aggregate usually corresponds to one of the major types a module works with. The Auth module's aggregate root might be `UserCredential`. The Orders module's might be `Order`. When thinking about what functions a module should expose, think about operations on its aggregate root.

---

### Domain Events

Something that happened in the domain that other parts of the system might need to know about. Events are named in the past tense: `UserAuthenticated`, `OrderPlaced`, `PaymentProcessed`.

**In Milo's context:** if a module emits events, they are part of its API contract and must be documented in the Product Brief alongside function signatures. Events are an alternative to direct function calls for asynchronous communication.

---

## Applying DDD to Module Decomposition

Use this process during `/solution-architecture`:

### Step 1: Event Storming (lightweight)

List every significant thing that can happen in the feature. Write each as a past-tense event:
- UserLoggedIn
- OrderCreated
- ProductAddedToCart
- PaymentProcessed
- EmailSent

### Step 2: Group by Domain

Group the events by the domain area they belong to. Events that share the same core entities and language belong in the same group.

### Step 3: Identify Commands

For each group, what actions (commands) trigger these events?
- LoginUser → UserLoggedIn
- CreateOrder → OrderCreated
- ProcessPayment → PaymentProcessed

### Step 4: Name the Modules

Each group of related events and commands becomes a bounded context — a module. Name it after its domain:
- Auth (owns: login, logout, token validation)
- Orders (owns: create order, cancel order, order status)
- Payments (owns: process payment, issue refund)
- Notifications (owns: send email, send SMS)

### Step 5: Draw the Boundaries

For each proposed module, state explicitly:
1. What it owns (its core responsibility)
2. What it does NOT own (where the boundary stops)
3. What data or capabilities it needs from other modules

If a module needs to reach into another module's data directly (not via the other module's function API), the boundary is wrong. Redraw it.

---

## Red Flags — Bad Module Boundaries

**Boundary too large:** One module handles authentication, user profiles, permissions, AND billing. This is a "god module" — it will accumulate too much logic and become impossible to evolve independently.

**Boundary too small:** A separate module for every database table. This is not DDD — it's an ORM layer. Modules should map to business capabilities, not data structures.

**Technology boundary:** A module called "EmailService" that just wraps SendGrid. This is a vendor adapter, not a domain module. It belongs inside a "Notifications" or "Communications" module that owns the domain of user communication.

**Shared mutable state:** Two modules both write to the same database table without going through each other's API. This creates hidden coupling. Each module should own its data exclusively — other modules access it only via the owning module's functions.

**Anemic model:** A module that only exposes CRUD operations (`createUser`, `getUser`, `updateUser`, `deleteUser`) with no domain logic. If there's no business behaviour, question whether this is a real bounded context or just a data store wrapper.

---

## Module Interaction Rules

**Rule 1: No direct database access across modules.**
Module A cannot query Module B's database table. Module A calls Module B's function. Module B queries its own table.

**Rule 2: No shared domain objects across module boundaries.**
If Module A passes data to Module B, the data is a primitive or a shared DTO — not Module A's internal domain entity. Module B should not know about Module A's internal structure.

**Rule 3: One direction of dependency.**
If Module A calls Module B, Module B should never call Module A back in the same request. This creates circular dependencies. If both modules need something from each other, there's a missing third module that owns the shared concern.

**Rule 4: The API is the contract.**
The only thing one module knows about another module is its function signatures. The implementation — the database structure, the internal algorithms, the classes — is hidden. This is enforced by the deployment abstraction: in distributed mode, all you have is the REST API.

---

## Special Module Types

### Standard API Module
- Has a bounded context with real domain logic
- Exports functions (API contract)
- Gets full deployment abstraction (function exports + REST server + smart call wrapper)
- Example: Auth, Orders, Catalog, Payments, Notifications

### Web UI
- A frontend application
- Consumes Standard API modules via function calls (routed through proxies)
- Has no API contract of its own — it does not expose functions to other modules
- Does not get deployment abstraction generation

### Mobile App
- Same rules as Web UI — consumer only, no API contract

### UI Component Library
- A shared library of reusable UI components
- Used by Web UI and/or Mobile App
- Exports components and their prop interfaces
- No REST API — components are imported directly, not called over network
- No smart call wrapper — component imports are always direct

---

## Naming Conventions

| Thing | Convention | Example |
|-------|-----------|---------|
| Module name | Lowercase noun, singular | `auth`, `orders`, `catalog` |
| Function name | camelCase verb + noun | `authenticateUser`, `placeOrder` |
| Event name | PascalCase past tense | `UserAuthenticated`, `OrderPlaced` |
| Entity/Type name | PascalCase noun | `User`, `Order`, `AuthResult` |
| Error name | PascalCase + "Error" | `AuthenticationError`, `OrderNotFoundError` |
