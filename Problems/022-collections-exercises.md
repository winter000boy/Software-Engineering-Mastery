# #022 — Solve 2–3 Exercises on Collections

**Date:** 2026-09-24  
**Type:** Exercise  
**Roadmap row:** 22 — P1 Java Core · Collections & Generics · *Flex*  
**Topics:** choosing the right collection, `HashMap` + `PriorityQueue` (top-k), `LinkedHashSet` (order + O(1) removal), `ArrayDeque` (monotonic deque), `TreeMap` navigation (bonus)

---

## Overview

Three interview-style problems (plus a bonus), each designed so that **picking the right collection is the whole solution**. For each: state the collections you'll use and their Big-O **before** coding, implement, then test against a brute-force reference on random inputs.

---

## Exercise 1 — Top-K Frequent Words

Given a list of words and `k`, return the `k` most frequent words, **most frequent first**. Ties are broken **alphabetically**.

```
words = ["the","day","is","sunny","the","the","the","sunny","is","is"], k = 4
→ ["the", "is", "sunny", "day"]        (the ×4, is ×3, sunny ×2, day ×1)

words = ["i","love","leetcode","i","love","coding"], k = 2
→ ["i", "love"]                       (both ×2; "i" < "love")
```

**Constraints:** `1 ≤ words.length ≤ 10⁵`; `k` may exceed the number of distinct words (return all of them); `k ≤ 0` → an empty list.  
**Target:** **O(n log k)** time. Sorting all distinct words (O(m log m)) is the baseline to beat.

---

## Exercise 2 — First Unique Element in a Stream

Design `FirstUnique<T>`:

- `add(T value)`: a new element arrives
- `Optional<T> firstUnique()`: the **earliest-arrived** element that has so far appeared **exactly once**, or empty

```
add 2, 3, 5    → firstUnique() = 2
add 2          → 3        (2 is now repeated)
add 3          → 5
add 5, add 5   → empty
add 7          → 7
```

**Target:** **both operations O(1)**. (Scanning a list is O(n) per query.)

---

## Exercise 3 — Sliding Window Maximum

Given `int[] nums` and a window size `k`, return the maximum of every contiguous window of size `k`.

```
nums = [1, 3, -1, -3, 5, 3, 6, 7], k = 3
windows: [1 3 -1] [3 -1 -3] [-1 -3 5] [-3 5 3] [5 3 6] [3 6 7]
→ [3, 3, 5, 5, 6, 7]
```

**Constraints:** `1 ≤ k ≤ nums.length ≤ 10⁵`; invalid `k` → `IllegalArgumentException`.  
**Target:** **O(n)** total. The brute force is O(n·k); a `TreeMap` of counts gives O(n log k), which is good but not optimal.

---

## Bonus — Booking Calendar (no double booking)

Design `BookingCalendar` with `boolean book(int start, int end)` for **half-open** intervals `[start, end)`. Return `false` (and don't store the booking) if it would overlap an existing booking. Intervals that only **touch** (`[10,20)` and `[20,30)`) don't overlap.

**Target:** **O(log n)** per booking. (A list scan is O(n).)

---

## Tasks (for each exercise)

1. Write down the collection(s) you'll use and the complexity of each operation you rely on.
2. Implement.
3. Test the examples above **plus** a **randomised comparison with a brute-force reference** (hundreds of random inputs).
4. Explain why a more obvious choice (sorting everything, `ArrayList` scans, `TreeMap`) is worse.

---

## Acceptance Criteria

- [ ] Ex. 1 is O(n log k), with alphabetical tie-breaks, matching a full-sort reference on random inputs
- [ ] Ex. 2: O(1) `add` and `firstUnique`
- [ ] Ex. 3 is O(n), matching brute force on random inputs
- [ ] Bonus is O(log n), with touching intervals accepted, matching brute force on random inputs
- [ ] Each solution has a complexity explanation

---

## Key takeaways (fill after solving)

1.  
2.  
3.  
