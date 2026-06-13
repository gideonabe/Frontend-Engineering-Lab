# Feature-Driven Architecture and Strict Colocation

> **Topic:** Architecture · **Level:** Intermediate · **Author:** [@gideonabe](https://github.com/gideonabe)

## The Problem

Most frontend applications begin with a folder structure organized by technical concerns.

Files are grouped by what they are:

```text
components/
hooks/
services/
types/
utils/
```

This structure works well when the application is small. However, as features grow and teams expand, it becomes increasingly difficult to understand ownership and manage change.

Imagine you're tasked with removing a deprecated **Billing** feature.

With a by-type structure, you might need to track down files scattered throughout the repository:

```text
components/BillingCard.tsx
hooks/useBilling.ts
services/stripe.ts
types/billing.ts
```

Over time, these files become imported by unrelated parts of the application. Deleting a feature turns into a risky exercise in dependency hunting.

The root problem is simple:

> Code that changes together does not live together.

As a result:

* Feature ownership becomes unclear
* Refactoring becomes more expensive
* Dependency graphs become harder to reason about
* Onboarding new developers takes longer
* Deleting features becomes risky

---

## The Solution

Feature-Driven Architecture organizes code around business domains rather than technical categories.

Instead of grouping files by what they are, group them by the feature they belong to.

Every feature becomes a self-contained module with its own internal implementation details and a clearly defined public API.

The goal is to make the feature—not the component—the primary unit of organization.

---

## Before: By-Type Organization

A common folder structure looks like this:

```text
src/
├── components/
│   ├── Button.tsx
│   ├── BillingCard.tsx
│   └── UserProfile.tsx
├── hooks/
│   ├── useClickOutside.ts
│   └── useBilling.ts
├── services/
│   ├── api.ts
│   └── stripe.ts
└── types/
```

At first glance, everything appears organized.

The problem emerges when trying to answer questions like:

* What code belongs to Billing?
* What can be safely removed?
* Who owns this functionality?

The structure optimizes for file type, not for understanding the application.

---

## After: Feature-Driven Colocation

Shared UI elements remain in common folders, while feature-specific logic moves into dedicated domains.

```text
src/
├── components/
│   └── Button.tsx
│
├── features/
│   ├── billing/
│   │   ├── components/
│   │   │   └── BillingCard.tsx
│   │   ├── hooks/
│   │   │   └── useBilling.ts
│   │   ├── api/
│   │   │   └── stripe.ts
│   │   ├── types/
│   │   └── index.ts
│   │
│   └── users/
│       ├── components/
│       │   └── UserProfile.tsx
│       └── index.ts
```

Now all Billing-related code lives in one place.

A developer investigating billing functionality no longer needs to search across multiple global directories.

The feature becomes a self-contained unit.

---

## The Secret Weapon: The Public API Boundary

The most important part of this architecture is not the folder structure.

It is the `index.ts` file.

Every feature should expose a limited public API through a single entry point.

```ts
// features/billing/index.ts

export { BillingCard } from "./components/BillingCard";

export type { BillingPlan } from "./types";
```

Only exports that are intended for external consumption should be exposed.

Internal implementation details remain private.

```ts
// NOT exported

useBilling();
stripeClient();
billingHelpers();
```

The rest of the application should never know these files exist.

---

## Why This Matters

Without a public API boundary, other features eventually start importing internal files directly:

```ts
import { useBilling } from
  "@/features/billing/hooks/useBilling";
```

These deep imports create hidden coupling.

Now a small internal refactor can break unrelated parts of the application.

Instead:

```ts
import { BillingCard } from
  "@/features/billing";
```

Consumers depend only on the feature's public contract.

The internal structure remains free to evolve.

---

## Enforcing Boundaries

Architecture patterns are most effective when they are enforced automatically.

Many teams prevent deep imports using ESLint rules such as:

```json
{
  "no-restricted-imports": [
    "error",
    {
      "patterns": [
        "@/features/*/*"
      ]
    }
  ]
}
```

This ensures developers interact with features only through their public API.

The result is stronger encapsulation and fewer accidental dependencies.

---

## Benefits in Practice

### Easier Refactoring

Feature internals can change without affecting consumers.

### Clear Ownership

Developers immediately know where functionality belongs.

### Faster Onboarding

New engineers can explore the application feature-by-feature rather than learning an entire technical hierarchy.

### Safer Deletions

Removing a feature becomes significantly easier because all related code lives together.

### Better Scalability

As the application grows, new features can be added without increasing the complexity of existing domains.

---

## Tradeoffs

No architecture is free.

| Aspect               | The Reality                                                                                                   |
| -------------------- | ------------------------------------------------------------------------------------------------------------- |
| **When this shines** | Medium-to-large applications, multiple contributors, rapidly evolving domains, and long-lived products        |
| **When to avoid it** | Small prototypes, simple marketing sites, or applications with minimal business complexity                    |
| **What you give up** | Simplicity of a flat structure, additional folder nesting, and the need to enforce boundaries through tooling |

Feature-driven architecture introduces more structure up front, but that structure often pays for itself as the codebase grows.

---

## Common Mistakes

### Moving Files Without Defining Boundaries

Creating feature folders without enforcing public APIs simply relocates the problem.

### Promoting Everything to Shared

Teams often move domain-specific code into shared directories too early.

Shared code should emerge naturally from repeated usage, not anticipation.

### Deep Importing

If consumers can access internal files directly, the feature is no longer truly encapsulated.

### Overengineering Small Projects

A five-page marketing site rarely needs feature modules.

Apply architectural complexity in proportion to application complexity.

---

## Key Takeaways

* Organize code by business domain rather than technical type.
* Code that changes together should live together.
* Every feature should expose a clear public API through `index.ts`.
* Internal implementation details should remain private.
* Feature-driven architecture improves maintainability, ownership, and scalability as applications grow.

## References

* Bulletproof React Project Structure
* Domain-Driven Design (DDD)
* Feature-Sliced Design
* ESLint `no-restricted-imports` Rule