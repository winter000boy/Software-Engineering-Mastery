# #022 — Solution: Solve 2–3 Exercises on Collections

**Date:** 2026-09-24  
**Problem:** [022-collections-exercises.md](../problems/022-collections-exercises.md)

> Verified on Java 21: **7/7 JUnit tests pass**. Each exercise is checked against a brute-force reference on hundreds of random inputs (200 top-k trials, 500 sliding-window trials, 200 calendar trials).

---

## Summary

| Exercise | Key collection(s) | Time | Space | Obvious alternative | Its cost |
|----------|-------------------|------|-------|---------------------|----------|
| 1. Top-K frequent words | `HashMap` (count) + **size-k min-heap** `PriorityQueue` | **O(n log k)** | O(n) | Sort all distinct words | O(n + m log m) |
| 2. First unique in a stream | **`LinkedHashSet`** (unique, in order) + `HashSet` (repeated) | **O(1)** per op | O(n) | `ArrayList` scan / `LinkedList` search-and-remove | O(n) per query or removal |
| 3. Sliding window max | **`ArrayDeque`** as a monotonic deque of indexes | **O(n)** total | O(k) | Rescan each window / `TreeMap` of counts | O(n·k) / O(n log k) |
| Bonus: booking calendar | **`TreeMap`** start → end, `floorEntry` + `ceilingEntry` | **O(log n)** per booking | O(n) | Scan every booking | O(n) per booking |

---

## Exercise 1 — Top-K Frequent Words

### Approach

1. **Count** with `HashMap.merge(word, 1, Integer::sum)`: O(n).
2. Define "better" = higher count, then alphabetically earlier.
3. Keep a **min-heap of size k** ordered by "worse first". Its root is always the *weakest* of the current top k. For each distinct word: `offer`, and if the size exceeds k, `poll` the weakest. Each step is O(log k) because the heap never holds more than k + 1 entries.
4. Drain the heap (weakest → strongest) and reverse.

**Why a min-heap for the top k?** You need quick access to the element that would be *evicted*, which is the weakest one. A max-heap of all m words would work too, but it's O(m) space in the heap and O(m + k log m) time.

### Code

### `TopKFrequent.java`

```java
package com.mastery.exercises;

import java.util.ArrayList;
import java.util.Collections;
import java.util.Comparator;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.PriorityQueue;

/** Exercise 1: the k most frequent words; ties broken alphabetically. */
public final class TopKFrequent {
    private TopKFrequent() { }

    /** O(n log k) time, O(n) space. */
    public static List<String> topK(List<String> words, int k) {
        if (k <= 0) {
            return List.of();
        }
        // 1. Count: O(n)
        Map<String, Integer> counts = new HashMap<>();
        for (String w : words) {
            counts.merge(w, 1, Integer::sum);
        }

        // 2. Order by "better": higher count first, then alphabetical.
        Comparator<Map.Entry<String, Integer>> better =
                Map.Entry.<String, Integer>comparingByValue().reversed()
                        .thenComparing(Map.Entry.comparingByKey());

        // 3. MIN-heap of size k keyed on "worse first": the root is the weakest of the current top k.
        PriorityQueue<Map.Entry<String, Integer>> heap = new PriorityQueue<>(better.reversed());
        for (Map.Entry<String, Integer> e : counts.entrySet()) {
            heap.offer(e);
            if (heap.size() > k) {
                heap.poll();                 // evict the weakest: heap never exceeds k → log k per op
            }
        }

        // 4. Drain weakest → strongest, then reverse.
        List<String> result = new ArrayList<>(heap.size());
        while (!heap.isEmpty()) {
            result.add(heap.poll().getKey());
        }
        Collections.reverse(result);
        return result;
    }

    /** Reference implementation for testing: sort everything. O(m log m) over m distinct words. */
    static List<String> topKBySorting(List<String> words, int k) {
        Map<String, Integer> counts = new HashMap<>();
        words.forEach(w -> counts.merge(w, 1, Integer::sum));
        return counts.entrySet().stream()
                .sorted(Map.Entry.<String, Integer>comparingByValue().reversed()
                        .thenComparing(Map.Entry.comparingByKey()))
                .limit(Math.max(k, 0))
                .map(Map.Entry::getKey)
                .toList();
    }
}
```

The comparator reuse is worth noticing: `better.reversed()` gives the heap's "worse first" order, so the tie-break rule is written **once**. The ordering is expressed entirely with `Map.Entry.comparingByValue()`/`comparingByKey()` combinators (#029).

---

## Exercise 2 — First Unique Element in a Stream

### Approach

- `unique`: a **`LinkedHashSet`** of elements seen exactly once. It keeps **arrival order** (so the head is the answer) and supports **O(1) removal of any element** through its hash index.
- `repeated`: a `HashSet` of elements seen 2+ times, so a third occurrence doesn't re-add the element to `unique`.

That's the key insight: you need *order* **and** *O(1) removal from the middle*. A queue gives order but O(n) removal; a hash set gives O(1) removal but no order. `LinkedHashSet` gives both (a hash table + a doubly linked list, #023).

### Code

### `FirstUnique.java`

```java
package com.mastery.exercises;

import java.util.HashSet;
import java.util.Iterator;
import java.util.LinkedHashSet;
import java.util.Optional;
import java.util.Set;

/**
 * Exercise 2: elements arrive one at a time; at any moment report the FIRST element (in arrival
 * order) that has appeared exactly once so far. add() and firstUnique() are both O(1).
 */
public final class FirstUnique<T> {
    private final Set<T> unique = new LinkedHashSet<>();   // seen exactly once, in arrival order
    private final Set<T> repeated = new HashSet<>();       // seen 2+ times

    public void add(T value) {
        if (repeated.contains(value)) {
            return;                          // already known to be a duplicate
        }
        if (!unique.add(value)) {            // add returns false if it was already present
            unique.remove(value);            // 2nd occurrence: O(1) unlink from the linked hash set
            repeated.add(value);
        }
    }

    public Optional<T> firstUnique() {
        Iterator<T> it = unique.iterator();  // head of the insertion-ordered set: O(1)
        return it.hasNext() ? Optional.of(it.next()) : Optional.empty();
    }
}
```

The trick in `add`: `Set.add` returns `false` if the element was already present. That single call both detects the second occurrence and avoids a separate `contains` lookup.

---

## Exercise 3 — Sliding Window Maximum

### Approach: a monotonic deque

The deque holds **indexes** whose values are **strictly decreasing** from head to tail. The head is the current window's maximum. For each new index `i`:

1. **Evict from the front** if the head index has left the window (`≤ i − k`).
2. **Evict from the back** every index whose value is **≤ `nums[i]`**. Those elements can never be a maximum again, because `nums[i]` is at least as big and stays in the window longer.
3. Push `i` at the back.
4. Once `i ≥ k − 1`, record `nums[head]`.

**Why O(n) despite the inner `while`?** Each index is pushed once and popped at most once, so there are at most 2n deque operations in total (**amortized** analysis, the same style of reasoning as #021 Q3).

Trace for `[1, 3, -1, -3, 5, 3, 6, 7]`, k = 3 (showing values):

| i | nums[i] | deque after | window max |
|--:|--:|---|--:|
| 0 | 1 | [1] | — |
| 1 | 3 | [3] (1 popped: smaller) | — |
| 2 | -1 | [3, -1] | **3** |
| 3 | -3 | [3, -1, -3] | **3** |
| 4 | 5 | [5] (3 left the window; -1, -3 popped) | **5** |
| 5 | 3 | [5, 3] | **5** |
| 6 | 6 | [6] | **6** |
| 7 | 7 | [7] | **7** |

### Code

### `SlidingWindowMax.java`

```java
package com.mastery.exercises;

import java.util.ArrayDeque;
import java.util.Deque;

/** Exercise 3: the maximum of every window of size k. O(n) with a monotonic deque. */
public final class SlidingWindowMax {
    private SlidingWindowMax() { }

    public static int[] maxPerWindow(int[] nums, int k) {
        if (k <= 0 || k > nums.length) {
            throw new IllegalArgumentException("k must be in 1.." + nums.length + ", was " + k);
        }
        int[] result = new int[nums.length - k + 1];
        // Holds INDEXES; their values are strictly decreasing from head to tail.
        // The head is always the index of the current window's maximum.
        Deque<Integer> deque = new ArrayDeque<>();

        for (int i = 0; i < nums.length; i++) {
            // 1. Drop the head if it has slid out of the window [i-k+1, i].
            if (!deque.isEmpty() && deque.peekFirst() <= i - k) {
                deque.pollFirst();
            }
            // 2. Drop smaller-or-equal values from the tail: they can never be a max again,
            //    because nums[i] is both larger and stays in the window longer.
            while (!deque.isEmpty() && nums[deque.peekLast()] <= nums[i]) {
                deque.pollLast();
            }
            deque.offerLast(i);
            // 3. Once the first full window exists, record its max.
            if (i >= k - 1) {
                result[i - k + 1] = nums[deque.peekFirst()];
            }
        }
        return result;
    }

    /** O(n·k) reference for tests. */
    static int[] bruteForce(int[] nums, int k) {
        int[] result = new int[nums.length - k + 1];
        for (int start = 0; start + k <= nums.length; start++) {
            int max = Integer.MIN_VALUE;
            for (int j = start; j < start + k; j++) {
                max = Math.max(max, nums[j]);
            }
            result[start] = max;
        }
        return result;
    }
}
```

`ArrayDeque`, not `LinkedList`: both ends are O(1), with no per-element node allocation, and it was measured 2.2× faster as a deque in #018.

---

## Bonus — Booking Calendar

### Approach

Store bookings in a `TreeMap<start, end>`. Because stored bookings never overlap each other, only **two** bookings could overlap a new `[start, end)`:

- the latest one starting at or before `start` (`floorEntry(start)`): it overlaps if it **ends after** `start`
- the earliest one starting at or after `start` (`ceilingEntry(start)`): it overlaps if it **starts before** `end`

Two O(log n) lookups replace an O(n) scan.

### Code

### `BookingCalendar.java`

```java
package com.mastery.exercises;

import java.util.Map;
import java.util.TreeMap;

/**
 * Bonus: book half-open intervals [start, end) with no double booking. O(log n) per booking.
 * Only the neighbours of the new interval can overlap it, and a TreeMap finds them directly.
 */
public final class BookingCalendar {
    private final TreeMap<Integer, Integer> bookings = new TreeMap<>();   // start → end

    public boolean book(int start, int end) {
        if (start >= end) {
            throw new IllegalArgumentException("start must be < end");
        }
        Map.Entry<Integer, Integer> before = bookings.floorEntry(start);  // latest booking starting ≤ start
        if (before != null && before.getValue() > start) {
            return false;                                                 // it hasn't ended yet
        }
        Map.Entry<Integer, Integer> after = bookings.ceilingEntry(start); // earliest booking starting ≥ start
        if (after != null && after.getKey() < end) {
            return false;                                                 // it starts before we end
        }
        bookings.put(start, end);
        return true;
    }

    public int count() {
        return bookings.size();
    }
}
```

Half-open intervals `[start, end)` make "touching" (`[10,20)` + `[20,30)`) naturally non-overlapping, with no ±1 fiddling. That's why most calendar and time-range APIs use them.

---

## Tests

### `test: ExercisesTest.java`

```java
package com.mastery.exercises;

import static org.junit.jupiter.api.Assertions.assertArrayEquals;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertTrue;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.Random;
import org.junit.jupiter.api.Test;

class ExercisesTest {

    @Test
    void topKWithTiesBrokenAlphabetically() {
        List<String> words = List.of("the", "day", "is", "sunny", "the", "the", "the", "sunny", "is", "is");
        assertEquals(List.of("the", "is", "sunny", "day"), TopKFrequent.topK(words, 4));
        assertEquals(List.of("i", "love"),
                TopKFrequent.topK(List.of("i", "love", "leetcode", "i", "love", "coding"), 2));
        assertEquals(List.of(), TopKFrequent.topK(words, 0));
        assertEquals(4, TopKFrequent.topK(words, 99).size());
    }

    @Test
    void topKMatchesSortingReferenceOnRandomInput() {
        Random rnd = new Random(1);
        for (int trial = 0; trial < 200; trial++) {
            List<String> words = new ArrayList<>();
            for (int i = 0; i < 300; i++) {
                words.add("w" + rnd.nextInt(40));
            }
            int k = 1 + rnd.nextInt(45);
            assertEquals(TopKFrequent.topKBySorting(words, k), TopKFrequent.topK(words, k));
        }
    }

    @Test
    void firstUniqueTracksTheStream() {
        FirstUnique<Integer> fu = new FirstUnique<>();
        assertEquals(Optional.empty(), fu.firstUnique());
        fu.add(2); fu.add(3); fu.add(5);
        assertEquals(Optional.of(2), fu.firstUnique());
        fu.add(2);
        assertEquals(Optional.of(3), fu.firstUnique());
        fu.add(3);
        assertEquals(Optional.of(5), fu.firstUnique());
        fu.add(5); fu.add(5);
        assertEquals(Optional.empty(), fu.firstUnique());
        fu.add(7);
        assertEquals(Optional.of(7), fu.firstUnique());
    }

    @Test
    void slidingWindowMaxExample() {
        assertArrayEquals(new int[]{3, 3, 5, 5, 6, 7},
                SlidingWindowMax.maxPerWindow(new int[]{1, 3, -1, -3, 5, 3, 6, 7}, 3));
        assertArrayEquals(new int[]{4}, SlidingWindowMax.maxPerWindow(new int[]{4}, 1));
        assertThrows(IllegalArgumentException.class, () -> SlidingWindowMax.maxPerWindow(new int[]{1}, 2));
    }

    @Test
    void slidingWindowMaxMatchesBruteForce() {
        Random rnd = new Random(2);
        for (int trial = 0; trial < 500; trial++) {
            int n = 1 + rnd.nextInt(60);
            int[] nums = rnd.ints(n, -20, 20).toArray();
            int k = 1 + rnd.nextInt(n);
            assertArrayEquals(SlidingWindowMax.bruteForce(nums, k), SlidingWindowMax.maxPerWindow(nums, k));
        }
    }

    @Test
    void calendarRejectsOverlapsAcceptsTouching() {
        BookingCalendar cal = new BookingCalendar();
        assertTrue(cal.book(10, 20));
        assertFalse(cal.book(15, 25));    // overlaps the end
        assertFalse(cal.book(5, 11));     // overlaps the start
        assertFalse(cal.book(12, 18));    // inside
        assertFalse(cal.book(0, 100));    // contains
        assertTrue(cal.book(20, 30));     // touches: [10,20) and [20,30) don't overlap
        assertTrue(cal.book(0, 10));
        assertEquals(3, cal.count());
    }

    @Test
    void calendarMatchesBruteForce() {
        Random rnd = new Random(3);
        for (int trial = 0; trial < 200; trial++) {
            BookingCalendar cal = new BookingCalendar();
            List<int[]> accepted = new ArrayList<>();
            for (int i = 0; i < 50; i++) {
                int s = rnd.nextInt(200);
                int e = s + 1 + rnd.nextInt(20);
                boolean free = accepted.stream().noneMatch(b -> s < b[1] && b[0] < e);
                assertEquals(free, cal.book(s, e));
                if (free) {
                    accepted.add(new int[]{s, e});
                }
            }
        }
    }
}
```

```text
$ mvn test -Dtest=ExercisesTest
Tests run: 7, Failures: 0, Errors: 0, Skipped: 0
```

**The testing pattern:** for each optimised solution, keep a slow-but-obviously-correct reference (`topKBySorting`, `bruteForce`, a list scan) and compare them on random inputs with a **fixed seed** (so failures reproduce). It's the same differential idea as #021 and #024, applied to algorithms.

---

## Key Takeaways

1. **Top-k → a size-k heap** (O(n log k)); the heap's root is the element to evict.
2. **Order + O(1) arbitrary removal → `LinkedHashSet`/`LinkedHashMap`.**
3. **Sliding-window max/min → a monotonic deque**, O(n) by amortized analysis.
4. **Interval overlap → `TreeMap.floorEntry`/`ceilingEntry`**: only the neighbours matter.
5. Always pair a clever solution with a **brute-force reference and randomised tests**.
