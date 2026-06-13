# Designing an Infinite Scroll Feed

> **Topic:** Frontend System Design · **Level:** Intermediate · **Author:** [@gideonabe](https://github.com/gideonabe)

## The Problem

Infinite scroll is one of the most common UX patterns in modern frontend applications — social feeds, product listings, timelines, and discovery pages.

At first glance, it seems simple:

> “Load more items when the user scrolls.”

But in production systems, it introduces real engineering challenges:

- How do we fetch data efficiently at scale?
- How do we avoid duplicate or missing items during pagination?
- How do we maintain performance as the DOM grows?
- How do we handle images, layout shift, and smooth scrolling?
- How do we ensure stability on slow or unreliable networks?

Without proper design, infinite scroll becomes:

- Memory-heavy
- Slow to render
- Hard to maintain
- Bug-prone during pagination

---

## Requirements Gathering

Before designing the solution, clarify requirements.

### Functional Requirements

- Display a feed of items
- Load more items on scroll
- Support refresh / reload
- Maintain correct ordering of items

### Non-Functional Requirements

- Smooth scrolling performance
- Minimal layout shift
- Efficient network usage
- No duplicate or missing items
- Resilient to network failures

---

## The Solution

An infinite scroll system has three main parts:

- Data fetching strategy
- Rendering strategy
- Scroll detection mechanism

---

## Data Fetching Strategy

### Offset Pagination (Simple but flawed)

```
GET /feed?offset=0&limit=20
```

Pros:
- Easy to implement

Cons:
- Inconsistent when new items are inserted
- Can cause duplicates or missing items

---

### Cursor-Based Pagination (Recommended)

```
GET /feed?cursor=abc123&limit=20
```

Response:

```json
{
  "items": [...],
  "nextCursor": "def456"
}
```

Why this works better:

- Stable ordering
- No duplication issues
- Scales better for large datasets

---

## Frontend Implementation

### Basic Approach

```tsx
function Feed() {
  const [items, setItems] = useState([]);
  const [cursor, setCursor] = useState(null);

  async function loadMore() {
    const res = await fetch(
      `/api/feed?cursor=${cursor}`
    ).then(r => r.json());

    setItems(prev => [...prev, ...res.items]);
    setCursor(res.nextCursor);
  }

  return (
    <div>
      {items.map(item => (
        <FeedItem key={item.id} item={item} />
      ))}
      <button onClick={loadMore}>
        Load More
      </button>
    </div>
  );
}
```

---

## Scroll Detection (Intersection Observer)

Instead of scroll event listeners, use Intersection Observer.

```ts
const observer = new IntersectionObserver((entries) => {
  if (entries[0].isIntersecting) {
    loadMore();
  }
});
```

Attach it to a sentinel element at the bottom of the list.

---

## Rendering Optimization

### Problem: Large DOM Growth

As the feed grows:

- More DOM nodes
- Higher memory usage
- Slower scroll performance

### Solution: Virtualization

Render only visible items.

Example using `react-window`:

```tsx
<FixedSizeList
  height={600}
  itemCount={items.length}
  itemSize={100}
>
  {Row}
</FixedSizeList>
```

---

## Handling Images and Layout Shift

Common issue: layout shifts when images load.

Fixes:

- Always set width/height
- Use lazy loading
- Reserve space for images

```tsx
<img
  src={item.image}
  loading="lazy"
  width={300}
  height={200}
/>
```

---

## Scroll Restoration

When users navigate away and return:

- Restore scroll position
- Restore feed state
- Avoid unnecessary refetching

Common approaches:

- In-memory caching
- React Query / SWR caching
- Session storage persistence

---

## Tradeoffs

### When this shines

- Social feeds
- Product listings
- Content discovery platforms

### When to avoid it

- SEO-critical pages requiring full pagination
- Tables or reports where users expect page numbers
- Small datasets where pagination is simpler

### What you give up

- Clear page boundaries
- Easier navigation
- Simpler state management

---

## Key Takeaways

- Cursor-based pagination is more reliable than offset pagination
- Intersection Observer is preferred over scroll events
- Virtualization is essential for large feeds
- Layout shift must be explicitly handled
- Infinite scroll is a UX pattern, not a default architecture choice

---

## References

- [MDN — Intersection Observer API](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API)
- [React Window Documentation](https://react-window.vercel.app/)
- [Google Web Vitals — Cumulative Layout Shift (CLS)](https://web.dev/cls/)