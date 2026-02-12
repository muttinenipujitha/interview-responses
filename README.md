


# React Native Interview Responses

This document contains technical responses regarding React Native performance and architecture, written with a focus on senior-level engineering concepts.



## Performance: How do you prevent unnecessary re-renders in a large FlatList?

When evaluating performance, it is critical to understand that re-renders are caused by changing data or changing function references. A robust optimization strategy addresses both.

### 1. Memoization of the Item Component (`React.memo`)
Wrapping the list item component in `React.memo` is the first line of defense. This prevents the item from re-rendering if its props haven't changed.

> **Senior Nuance:** `React.memo` is useless if you are passing inline functions (arrow functions) as props. If the parent re-renders, it creates a new function reference every time, bypassing the memoization. Handlers must be wrapped in `useCallback` in the parent or attached to a stable ref.

### 2. Implementing `getItemLayout`
If list items have a fixed height, using `getItemLayout` is critical. It skips the expensive measurement of the content view for every item. Without this, React Native must asynchronously measure items before scrolling, causing "white flashes" or jank.

```javascript
getItemLayout={(data, index) => ({
  length: ITEM_HEIGHT,
  offset: ITEM_HEIGHT * index,
  index,
})}
```

### 3. Unique and Stable Keys (`keyExtractor`)
Ensure the `keyExtractor` returns unique strings (usually IDs from your data), not array indices. Using indices causes React to destroy and recreate the DOM node for every item if the list order changes, destroying scrolling performance.

### 4. Optimization Flags
*   **`removeClippedSubviews`:** Set to `true`. This unmounts views that are off-screen, significantly lowering memory pressure for long lists.
*   **`windowSize`:** Adjust this to control how many items are rendered *off-screen*. A smaller window means faster initial load but a higher chance of seeing blank space while scrolling fast.

### 5. Data Optimization
Avoid creating new array instances for the `data` prop on every render of the parent. If the parent re-renders, ensure you are passing a reference to the *same* array if the data hasn't actually changed.

---

## The "Bridge": Explanation and Performance Impact

### What is it?
The Bridge is the core architectural layer enabling communication between the **JavaScript thread** (where React code runs) and the **Native threads** (where the iOS/Android UI is rendered).

Think of it as an asynchronous, serialized message queue. When you write `<Text>Hello</Text>` in JS, React Native serializes that instruction into a JSON message, sends it across the Bridge, and the Native side deserializes it to create a specific `UIView` or `android.widget.TextView`.

### Why does it matter for performance?
The Bridge is often the performance bottleneck in React Native apps because:

1.  **Serialization Overhead:** Every interaction involves converting complex objects to JSON and back. This consumes CPU cycles.
2.  **Asynchronous Latency:** The bridge is asynchronous. While this prevents the JS thread from blocking the UI thread (keeping animations smooth at 60fps), it means there is a delay between a user action and the app's response if the message queue is clogged.
3.  **Single Point of Failure:** If the JS thread becomes heavy (complex calculations, large image processing), it cannot send messages across the bridge quickly. Consequently, the Native UI stops receiving updates, leading to "jank" or dropped frames.

> **The "Senior" Signal:** A top-tier candidate will mention **The New Architecture (Fabric/TurboModules)**. This architecture replaces the asynchronous Bridge with **JSI (JavaScript Interface)**, allowing for direct, synchronous memory access between JS and Native. This eliminates the serialization tax and drastically improves performance.
```
