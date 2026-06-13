# Designing a Feature Flag System for React SaaS Applications

> **Topic:** Frontend System Design · **Level:** Intermediate · **Author:** [@gideonabe](https://github.com/gideonabe)

## The Problem

As applications grow, releasing new features becomes increasingly risky.

A simple deployment strategy works when there are only a few users and a small codebase. However, once an application serves multiple customer groups, regions, subscription plans, or internal teams, deployments and releases become separate concerns.

Consider the following situations:

* A feature should only be available to beta users.
* A new dashboard should be rolled out gradually.
* A bug in a recently released feature requires immediate rollback.
* Enterprise customers need access to functionality unavailable to standard users.
* Product teams want to run experiments without creating separate deployments.

Without feature flags, teams often rely on long-lived branches, environment-specific builds, or emergency redeployments. These approaches increase operational risk and slow down delivery.

A feature flag system allows teams to deploy code safely while controlling who can access functionality and when.

---

## Requirements

Before designing a feature flag system, define what problems it must solve.

For a typical SaaS application, the system should support:

| Requirement            | Why It Matters                                                 |
| ---------------------- | -------------------------------------------------------------- |
| Runtime updates        | Enable or disable features without redeploying                 |
| User targeting         | Release features to specific users                             |
| Role-based access      | Restrict features to admins, support agents, etc.              |
| Percentage rollouts    | Gradually expose functionality                                 |
| Environment support    | Different behavior across development, staging, and production |
| Centralized management | Prevent scattered configuration                                |
| Easy removal           | Avoid long-term flag debt                                      |

---

## The Solution

A common architecture consists of three layers:

```text
Feature Flag Service
          │
          ▼
Frontend Flag Provider
          │
          ▼
Feature Gate Layer
          │
          ▼
Application Components
```

Each layer has a clear responsibility.

### 1. Flag Service

Flags should not be hardcoded into components.

Instead, they should be retrieved from a central source:

```json
{
  "newDashboard": true,
  "billingV2": false,
  "advancedAnalytics": true
}
```

The source may be:

* Internal backend API
* LaunchDarkly
* Unleash
* Config service
* Edge configuration system

The frontend should treat flags as data.

---

### 2. Create a Flag Provider

A provider centralizes access to feature flags.

```tsx
import { createContext, useContext } from "react";

type Flags = {
  newDashboard: boolean;
  billingV2: boolean;
};

const FeatureFlagContext =
  createContext<Flags | null>(null);

export function useFlags() {
  const flags = useContext(FeatureFlagContext);

  if (!flags) {
    throw new Error(
      "FeatureFlagProvider missing"
    );
  }

  return flags;
}
```

This prevents components from making independent requests for configuration.

---

### 3. Build Feature Gates

Instead of scattering conditional checks throughout the application:

```tsx
if (flags.newDashboard) {
  return <NewDashboard />;
}
```

Create a reusable gate:

```tsx
type FeatureGateProps = {
  enabled: boolean;
  children: React.ReactNode;
};

export function FeatureGate({
  enabled,
  children,
}: FeatureGateProps) {
  if (!enabled) return null;

  return <>{children}</>;
}
```

Usage:

```tsx
<FeatureGate enabled={flags.newDashboard}>
  <NewDashboard />
</FeatureGate>
```

This creates a consistent pattern across the application.

---

## Handling Role-Based Access

Feature flags and authorization solve different problems.

A common mistake is treating feature flags as permission systems.

Avoid:

```tsx
if (
  flags.adminPanel &&
  user.role === "admin"
) {
  return <AdminPanel />;
}
```

Move access rules into a dedicated layer:

```ts
export function canAccessAdminPanel(
  user: User,
  flags: Flags
) {
  return (
    flags.adminPanel &&
    user.role === "admin"
  );
}
```

This keeps business rules centralized and testable.

---

## Percentage Rollouts

Large releases often benefit from gradual exposure.

For example:

* Day 1 → 5% of users
* Day 3 → 25% of users
* Day 7 → 100% of users

A simple approach uses a deterministic hash:

```ts
export function isIncludedInRollout(
  userId: string,
  percentage: number
) {
  const value =
    Number(userId.slice(-2)) % 100;

  return value < percentage;
}
```

The same user always receives the same experience.

This prevents features from randomly appearing and disappearing between sessions.

---

## Performance Considerations

Feature flags can accidentally hurt performance if implemented incorrectly.

Common mistakes include:

* Fetching flags multiple times
* Blocking application startup
* Performing expensive flag calculations during rendering

Prefer:

* Single initialization request
* Cached flag state
* Context-based access
* Server-side hydration where possible

Feature evaluation should be inexpensive enough to occur on every render.

---

## Managing Flag Debt

One of the biggest long-term problems is stale flags.

Teams often create flags but never remove them.

Over time this leads to:

```tsx
if (flagA) {
  if (flagB) {
    if (flagC) {
      // actual feature
    }
  }
}
```

The codebase becomes difficult to reason about.

A useful rule:

> Every feature flag should have an owner and an expected removal date.

Once a rollout reaches 100% and confidence is high, remove the flag and simplify the code.

Feature flags are temporary tools, not permanent architecture.

---

## Tradeoffs

### When this shines

* Large SaaS applications
* Multiple customer segments
* Continuous delivery environments
* Experimentation platforms
* Enterprise products

### When to avoid it

* Small internal tools
* MVPs with few users
* Applications with infrequent releases

### What you give up

* Additional infrastructure
* More operational complexity
* Ongoing maintenance
* Risk of accumulating flag debt

The flexibility gained from controlled releases comes at the cost of system complexity.

---

## Key Takeaways

* Feature flags separate deployment from release.
* Centralized flag management scales better than component-level checks.
* Feature flags should not replace authorization systems.
* Percentage rollouts reduce risk during large releases.
* Every feature flag should have a removal plan to prevent technical debt.

## References

* Martin Fowler — Feature Toggles
* LaunchDarkly Documentation
* Unleash Documentation
* React Context Documentation
