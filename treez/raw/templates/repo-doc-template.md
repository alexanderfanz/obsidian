# [Repo Name]

> One-paragraph summary of what this repo does and its role in the Treez ecom system.

---

## Purpose

What business problem does this repo solve? What is it responsible for, and what is explicitly out of scope?

---

## Key Entities / Domain Models

The core data structures and domain objects. For each:
- **Name** — what it represents, key fields, relationships to other entities

---

## API Surface

Exposed interfaces — HTTP routes, RPC methods, exported modules, events published. Focus on the contract, not the implementation.

| Method | Path / Name | Description |
|--------|-------------|-------------|
| GET    | /example    | ...         |

---

## Data Layer

- Database(s) used and why
- Key tables / collections and their purpose
- Notable relationships or constraints
- Any caching layer

---

## Core Business Logic

The non-obvious parts — rules, calculations, workflows, state machines. Avoid describing CRUD. Focus on what makes this domain interesting or complex.

---

## Cross-Repo Connections

How this repo connects to others:
- **Calls**: services or APIs it depends on
- **Called by**: who depends on this
- **Shared types / contracts**: any shared interfaces or schemas
- **Events**: publishes or subscribes to

---

## Third-Party Services

External services and APIs this repo depends on. Captured here to track the full dependency surface across repos and spot redundancy.

| Service | Category | What it's used for |
|---------|----------|--------------------|
| e.g. Stripe | Payments | ... |
| e.g. SendGrid | Email | ... |
| e.g. Datadog | Monitoring | ... |

Categories: Payments, Email, SMS, Auth, Analytics, Monitoring, Infrastructure, Search, Storage, Shipping, Tax, Other

---

## Notable Patterns & Decisions

Non-obvious architectural choices, design patterns in use, or anything a new engineer would need to understand that isn't visible from the code alone.

---

## Potential Improvements

Things noticed during analysis that could or should be addressed. Not a complete audit — just what stands out.

| Area | Observation | Priority |
|------|-------------|----------|
| Tech debt | e.g. X module does Y but should Z | High / Med / Low |
| Missing feature | ... | ... |
| Performance | ... | ... |
| Architecture | ... | ... |
| DX / tooling | ... | ... |

---

*Generated: [date] — Source: [repo path or commit]*
