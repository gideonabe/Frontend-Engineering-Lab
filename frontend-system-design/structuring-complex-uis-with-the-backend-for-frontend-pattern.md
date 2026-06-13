# Structuring Complex UIs with the Backend-For-Frontend (BFF) Pattern

> **Topic:** Frontend System Design · **Level:** Advanced · **Author:** [@gideonabe](https://github.com/gideonabe)

## The Problem

As frontend applications grow, the backend often evolves from a single monolith into multiple domain-specific services.

While this improves backend scalability and ownership, it can create significant complexity for frontend teams.

Consider a dashboard page that needs data from:

* A Users service
* An Orders service
* A Notifications service

If the browser communicates directly with each service, several problems emerge:

* **Network waterfalls** — sequential requests increase page load times and hurt Largest Contentful Paint (LCP)
* **Over-fetching** — APIs return large payloads when the UI only needs a small subset of the data
* **Heavy client-side orchestration** — frontend code becomes responsible for data aggregation, transformation, filtering, and joining
* **Complex loading states** — handling multiple asynchronous dependencies increases UI complexity

At this point, the frontend is no longer just rendering a user interface. It is acting as a distributed data orchestration layer running on the user's device.

---

## Background

The Backend-For-Frontend (BFF) pattern was popularized as a way to tailor backend APIs to the specific needs of individual client applications.

Instead of exposing generic APIs and forcing clients to assemble data themselves, a dedicated backend layer is introduced for a specific frontend.

The BFF becomes responsible for:

* Aggregating data from multiple services
* Shaping responses for UI consumption
* Reducing payload sizes
* Hiding backend complexity from frontend applications

This allows frontend teams to optimize data contracts around user experiences rather than backend service boundaries.

---

## The Solution

The Backend-For-Frontend pattern introduces a lightweight server layer that sits between the client and backend services.

Instead of communicating directly with multiple APIs, the frontend talks to a single endpoint owned by the BFF.

```text
Frontend
    │
    ▼
Backend For Frontend (BFF)
    │
    ├── Users Service
    ├── Orders Service
    └── Notifications Service
```

The BFF's responsibility is simple:

> Fetch data from multiple backend services and return exactly what the UI needs.

---

## Before: Client-Side Orchestration

Without a BFF, the browser must coordinate multiple requests and perform data transformations itself.

```tsx
function Dashboard() {
  const [data, setData] = useState(null);

  useEffect(() => {
    async function loadDashboard() {
      const user = await fetch(
        "https://api.example.com/users/me"
      ).then((r) => r.json());

      const [ordersRes, notificationsRes] =
        await Promise.all([
          fetch(
            `https://api.example.com/orders?userId=${user.id}`
          ),
          fetch(
            `https://api.example.com/notifications?userId=${user.id}`
          ),
        ]);

      const orders = await ordersRes.json();
      const notifications =
        await notificationsRes.json();

      setData({
        userName: `${user.firstName} ${user.lastName}`,
        hasUnread: notifications.some(
          (n) => !n.read
        ),
        recentPurchases: orders
          .slice(0, 3)
          .map((o) => o.itemName),
      });
    }

    loadDashboard();
  }, []);

  return <DashboardUI data={data} />;
}
```

### Problems

* Multiple network requests
* Additional client-side processing
* Larger payloads
* Increased bundle complexity
* More difficult testing

---

## After: The BFF Pattern

The frontend makes a single request and receives data already shaped for the UI.

### Frontend

```tsx
function Dashboard() {
  const { data, isLoading } = useQuery(
    "/api/bff/dashboard",
    fetcher
  );

  if (isLoading) {
    return <Spinner />;
  }

  return <DashboardUI data={data} />;
}
```

### BFF

```ts
app.get("/api/bff/dashboard", async (req, res) => {
  const userId = req.session.userId;

  const [user, orders, notifications] =
    await Promise.all([
      userService.getUser(userId),
      orderService.getOrders(userId),
      notificationService.getNotifications(userId),
    ]);

  const uiPayload = {
    userName: `${user.firstName} ${user.lastName}`,
    hasUnread: notifications.some(
      (n) => !n.read
    ),
    recentPurchases: orders
      .slice(0, 3)
      .map((o) => o.itemName),
  };

  res.json(uiPayload);
});
```

The orchestration now happens inside the data center rather than on the user's device.

---

## Why This Improves Frontend Systems

### Reduced Network Overhead

The browser performs one request instead of several.

### Smaller Payloads

The BFF returns only the data required by the UI.

### Simpler Components

Components focus on rendering rather than data assembly.

### Better Performance

Server-to-server communication is typically much faster than browser-to-service communication.

### Clear Ownership

Frontend teams can evolve UI contracts independently of backend service structures.

---

## Tradeoffs

Like any architectural pattern, introducing a BFF comes with costs.

| Aspect               | The Reality                                                                                                                                        |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| **When this shines** | Multiple backend services, mobile and web clients with different requirements, strict performance budgets, or complex dashboard-style applications |
| **When to avoid it** | Simple CRUD applications, monolithic backends that already provide tailored endpoints, or small teams with limited operational capacity            |
| **What you give up** | Infrastructure simplicity, additional deployments, monitoring requirements, operational overhead, and another potential point of failure           |

A BFF simplifies frontend development by introducing complexity elsewhere in the system.

That tradeoff is often worthwhile, but it should be intentional.

---

## Common Mistakes

### Turning the BFF into a Monolith

A BFF should orchestrate and shape data, not contain extensive business logic.

### Exposing Backend Models Directly

The BFF should define contracts optimized for the UI rather than forwarding backend responses unchanged.

### Creating a Single Shared BFF for Every Client

Different clients often have different needs.

A web application, mobile application, and admin dashboard may benefit from separate BFF layers.

### Ignoring Caching

Repeated aggregation calls can become expensive.

Caching frequently requested data can significantly reduce load on downstream services.

---

## Key Takeaways

* Move data orchestration closer to backend services and away from the browser.
* A BFF allows complex views to fetch all required data in a single request.
* UI contracts should be designed around frontend needs rather than backend service boundaries.
* The BFF pattern improves performance and developer experience at the cost of additional infrastructure.
* A BFF should remain focused on aggregation and presentation concerns rather than becoming a second business-logic layer.

## References

- [Sam Newman — Pattern: Backends For Frontends (BFF)](https://samnewman.io/patterns/architectural/bff/)