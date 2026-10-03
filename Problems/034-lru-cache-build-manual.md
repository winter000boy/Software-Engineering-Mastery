# #034 — Build (cont.): LRU Cache — Manual HashMap + Doubly Linked List

**Date:** 2026-10-03  
**Type:** Project — Phase 3 of 5  
**Roadmap row:** 34 — P1 Java Core · Collections & Generics  
**Prerequisite:** **#033** (API + contract suite)

---

## Goal

Build `ManualLruCache<K, V>` **without** `LinkedHashMap`, from a `HashMap<K, Node<K, V>>` plus a doubly linked list you write yourself, so that you understand exactly what `LinkedHashMap`'s access order does internally (#023 §2). It must pass the **same** contract suite as #033 with zero changes to the tests.

---

## Required Design

```
   head ⇄ [LRU] ⇄ [ ] ⇄ ... ⇄ [MRU] ⇄ tail        head, tail = sentinel nodes (no data)
   index: HashMap<K, Node<K, V>>                     key → its node, so any node is reachable in O(1)
```

| Operation | Steps | Cost |
|-----------|-------|------|
| `get(k)` | look up the node in `index`; if found, **move it to the tail** | O(1) |
| `put(k, v)` existing | update the value; move to the tail | O(1) |
| `put(k, v)` new, full | **unlink `head.next`** (the LRU) and remove it from `index`; then insert at the tail | O(1) |
| `remove(k)` | remove from `index`; unlink | O(1) |

---

## Tasks

1. A `private static final class Node<K, V>` with `key`, `value`, `prev`, `next`. Explain why it must be **static** and why it must store the **key**.
2. Three private list primitives: `unlink(node)`, `linkAtTail(node)`, `moveToTail(node)`. Each touches at most 4 pointers, and `moveToTail` is a no-op if the node is already the MRU.
3. Implement every `Cache` method on top of them. Use `HashMap.newHashMap(capacity)` so the index never resizes.
4. `clear()` must leave a usable empty list. `keysLruToMru()` walks from `head.next` to `tail`.
5. `ManualLruCacheTest extends CacheContractTest`: only the factory method.
6. Draw the pointer changes for `moveToTail` on paper: before, after `unlink`, after `linkAtTail`.

---

## Hints

- With sentinels, **no method ever checks for `null` neighbours**, because every real node always has a `prev` and a `next`. Without them, you need special cases for empty, first and last, and those are exactly where pointer bugs live.
- Setting `node.prev = node.next = null` in `unlink` costs nothing and makes reusing a stale node fail fast (with an NPE) instead of corrupting the list silently.
- Eviction needs the key of the LRU node, to remove it from `index`. That's why the node stores the key.

---

## Acceptance Criteria

- [ ] No `LinkedHashMap`, `LinkedList` or other JDK list in the implementation (`HashMap` for the index is fine)
- [ ] Sentinel head/tail; no `null` checks on neighbours
- [ ] All `get`/`put`/`remove` paths O(1)
- [ ] The **unchanged** contract suite passes
- [ ] Pointer diagram for `moveToTail`

---

## Key takeaways (fill after solving)

1.  
2.  
3.  
