# How an Infinite Re-Render Increased API Traffic by 20x

> **Topic:** Case Study · **Level:** Intermediate · **Author:** [@gideonabe](https://github.com/gideonabe)

## Context

This incident occurred in a React-based internal dashboard used by several thousand active users.

The application had been in production for over a year and relied on:

* React
* TypeScript
* REST APIs
* Custom data-fetching hooks

The team was making a small refactor to improve readability and standardize how dashboard filters were managed.

At the time, there was no indication that the change would affect application performance or infrastructure.

The update passed code review, testing, and staging validation before being deployed to production.

---

## The Problem

A few hours after deployment, backend monitoring began showing unusual traffic patterns.

Symptoms included:

* A sharp increase in API requests
* Higher backend response times
* Increased rate-limit errors
* Reports of slower dashboard loading

Initially, the problem appeared to be a backend issue.

The API infrastructure was healthy, database performance was normal, and there were no deployment-related errors.

The challenge was identifying why traffic had suddenly increased without any significant increase in user activity.

---

## What We Tried

### Attempt 1: Investigating the Backend

The first assumption was that something in the backend had regressed.

The team investigated:

* Database query performance
* API response times
* Infrastructure metrics
* Load balancer logs

Everything looked healthy.

Requests were arriving at a much higher rate than expected, but the services themselves were functioning correctly.

This shifted attention back to the frontend.

---

### Attempt 2: Looking for Duplicate Requests

Browser network logs revealed something unusual.

A page that normally generated a single request was generating dozens of requests within a short period.

Instead of:

```text id="v9gh1n"
GET /dashboard
```

once per page visit, the endpoint was being called repeatedly.

This suggested that some part of the frontend was continuously triggering data fetching.

---

### Attempt 3: Profiling Component Renders

Using React DevTools, the team inspected component rendering behavior.

One dashboard component was re-rendering continuously.

The component contained a data-fetching effect:

```tsx id="jlwmkt"
useEffect(() => {
  fetchDashboardData(filters);
}, [filters]);
```

At first glance, nothing looked suspicious.

The dependency array appeared correct.

---

### Root Cause Discovery

The issue was caused by an object created inside the component.

```tsx id="fp7wm4"
const filters = {
  status: "active",
};
```

Every render created a new object reference.

React compared dependencies by reference, not by value.

As a result:

1. Component renders
2. New filters object created
3. React detects dependency change
4. Effect runs
5. API request fires
6. State updates
7. Component renders again

The cycle repeated indefinitely.

A small refactor had accidentally introduced an infinite re-render loop.

---

## The Outcome

The fix was straightforward.

The filters object was stabilized using `useMemo`:

```tsx id="8m7i6u"
const filters = useMemo(
  () => ({
    status: "active",
  }),
  []
);
```

An even cleaner solution was later adopted:

```tsx id="p4m3z6"
const DEFAULT_FILTERS = {
  status: "active",
};

function Dashboard() {
  useEffect(() => {
    fetchDashboardData(DEFAULT_FILTERS);
  }, []);
}
```

After deployment:

* API traffic returned to normal levels
* Rate-limit errors disappeared
* Dashboard performance improved
* Backend resource consumption dropped significantly

No infrastructure changes were required.

The entire incident was caused by a frontend rendering issue.

---

## Tradeoffs & What We'd Do Differently

The technical fix itself was simple.

The larger lesson was about process and tooling.

After the incident, we introduced:

### Stronger Hook Reviews

Code reviews now explicitly check:

* Effect dependencies
* Object references
* Function references
* Memoization requirements

### Better Tooling

Additional React Hook linting rules were enabled to catch similar issues earlier.

### Less Manual Data Fetching

We gradually adopted React Query for server-state management.

This reduced the number of custom data-fetching effects throughout the application.

In hindsight, the original code review focused too heavily on functionality and not enough on rendering behavior.

---

## Key Takeaways

* Objects and arrays in dependency arrays can trigger unexpected effect executions.
* Frontend performance bugs can create significant backend load.
* Browser network logs are often the fastest way to identify request-related incidents.
* Small React changes can have system-wide consequences.
* Reviewing rendering behavior is just as important as reviewing business logic.