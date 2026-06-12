# Data Structures & Systems Design — SE Intern_Assessment

## Problem 1: LRU Cache Implementation

### Logic & Approach

The O(1) constraint on both `get` and `put` is really the crux of the problem. It rules out
anything that requires scanning through elements — you need a structure where you can jump
directly to any item and reorder it without touching everything else.

My starting point was a **dictionary**, since it gives O(1) lookups by key. But a dict alone
doesn't track which item was used least recently — you'd have to iterate to figure that out.
An **array** could track order, but moving an element to the front means shifting everything
else, which is O(N). Neither alone is enough.

The combination that solves both problems is a **hash map paired with a doubly linked list**.
The key insight is that the dictionary doesn't store the value directly — it stores a reference
to the **node** in the linked list. So when I do a `get`, I'm not just retrieving a value; I'm
getting a direct pointer to the exact position in the list, which means I can move that node to
the front (marking it as most recently used) with just a few pointer changes — no iteration.
Same for eviction: the node just before the dummy tail is always the least recently used, and
removing it is constant time.

I used dummy head and tail sentinel nodes to keep the insert/remove logic clean. Without them,
you end up handling special cases for empty lists or single-element lists, which complicates
every operation.

### Code

```python
class Node:
    def __init__(self, key=0, value=0):
        self.key = key
        self.value = value
        self.prev = None
        self.next = None


class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = {}

        # Dummy sentinel nodes — avoids edge cases for insert/delete
        self.head = Node()
        self.tail = Node()
        self.head.next = self.tail
        self.tail.prev = self.head

    def _remove(self, node):
        """Unlinks a node from wherever it currently sits in the list."""
        prev_node = node.prev
        next_node = node.next
        prev_node.next = next_node
        next_node.prev = prev_node

    def _insert_at_front(self, node):
        """Places a node right after the head (most recently used position)."""
        node.next = self.head.next
        node.prev = self.head
        self.head.next.prev = node
        self.head.next = node

    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1

        node = self.cache[key]

        # Accessing promotes it to most recently used
        self._remove(node)
        self._insert_at_front(node)

        return node.value

    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            node = self.cache[key]
            node.value = value
            self._remove(node)
            self._insert_at_front(node)
            return

        if len(self.cache) >= self.capacity:
            # The node just before the tail is the least recently used
            lru = self.tail.prev
            self._remove(lru)
            del self.cache[lru.key]

        new_node = Node(key, value)
        self.cache[key] = new_node
        self._insert_at_front(new_node)
```

---

## Problem 2: Event Scheduler

### Logic & Approach

**`can_attend_all`:** I sort the events by start time first, then do a single linear pass.
For each event, I check if it starts before the previous one ends. If it does, there's an
overlap. The strict `<` comparison naturally satisfies the constraint that adjacent events
(where one ends exactly when the next starts) are not considered overlaps.

**`min_rooms_required`:** The core idea is that the number of rooms needed at any given
moment equals the number of events running simultaneously. I track this with a **min-heap
storing end times** of all currently active meetings.

Processing events in sorted start-time order simulates moving through time. For each new
meeting, I check whether the meeting that finishes soonest (the heap's minimum) is already
done by the time this one starts. If it is, that room can be reused — I pop the old end
time and push the new one in its place. If not, I need a new room, so I push without
popping, growing the heap. The heap's size after processing all events is the peak number
of concurrent meetings, which is our answer.

The greedy logic works because we're always asking "can I reuse *any* free room?" — and
checking the one that ended soonest is the best candidate for that.

### Code

```python
import heapq


def can_attend_all(events):
    if not events:
        return True

    events.sort(key=lambda x: x[0])

    for i in range(1, len(events)):
        prev_end = events[i - 1][1]
        curr_start = events[i][0]

        # Strict less-than: adjacent events (end == start) are allowed
        if curr_start < prev_end:
            return False

    return True


def min_rooms_required(events):
    if not events:
        return 0

    events.sort(key=lambda x: x[0])

    min_heap = []  # Stores end times of ongoing meetings

    for start, end in events:
        # If the earliest-ending meeting is done, reclaim that room
        if min_heap and min_heap[0] <= start:
            heapq.heappop(min_heap)

        heapq.heappush(min_heap, end)

    # Heap size = number of rooms still occupied = minimum rooms needed
    return len(min_heap)
```

---

## Final Discussion & Analysis

### Time & Space Complexity

**Problem 1 — LRU Cache**

| Operation       | Time Complexity | Space (auxiliary) |
|-----------------|-----------------|-------------------|
| `get(key)`      | O(1) average    | O(1)              |
| `put(key, val)` | O(1) average    | O(1)              |
| Overall storage | —               | O(C), C = capacity|

Both operations are constant time because dict access is O(1) and linked list
pointer reassignments don't depend on the number of elements. The "average" qualifier
covers the theoretical possibility of hash collisions, though this doesn't affect
practical performance.

**Problem 2 — Event Scheduler**

| Function                | Time Complexity | Space Complexity   |
|-------------------------|-----------------|--------------------|
| `can_attend_all`        | O(N log N)      | O(N) for the sort  |
| `min_rooms_required`    | O(N log N)      | O(N) worst case    |

Both are dominated by the sort. In `min_rooms_required`, each event involves at most
one push and one pop, each costing O(log N), so the loop is O(N log N) overall.
Heap space is O(N) in the worst case — if every event overlaps with every other,
all end times accumulate in the heap simultaneously.

---

### Trade-offs: Why Hash Map + Doubly Linked List?

Each structure fills the gap the other leaves.

A **dictionary alone** gives O(1) key access but has no concept of order. Figuring out
which item is least recently used would mean scanning the whole structure every time —
O(N) per operation.

A **doubly linked list alone** tracks order perfectly. You can move any node to the
front or remove the back node in O(1) *once you have a reference to it* — but finding
that node by key takes O(N).

Together, the dictionary maps keys directly to list nodes, giving instant access. The
list handles ordering and eviction in constant time. The trade-off is memory: every
node carries `prev` and `next` pointers on top of its actual data, and you're
maintaining two structures in sync. For a cache where speed is the priority, that's
a straightforward trade worth making.

---

### Future Proofing: Assigning Specific Room Names

The current implementation returns a count. To assign actual labels like "Room A",
"Room B", I'd modify `min_rooms_required` as follows:

1. Initialise a pool of available room names (a `deque` or list of strings).
2. Change the heap to store `(end_time, room_name)` tuples instead of just end times.
3. When a meeting ends (i.e., `min_heap[0][0] <= start`), pop it and return that room
   name to the available pool.
4. Pull a name from the available pool for the incoming meeting, push
   `(end_time, room_name)` onto the heap, and record the assignment in a result dict.

If the number of rooms isn't known upfront, room names can be generated dynamically
the first time a new room is opened (e.g., `f"Room {chr(65 + room_counter)}"`). The
rest of the logic stays the same since we're just replacing an integer count with a
label.

---

### Concurrency: Making LRU Cache Thread-Safe

In a multi-threaded environment, two threads could call `get` or `put` at the same
time. One thread could be mid-way through a pointer reassignment while another reads
a partially updated state, leading to corrupted list links or a desynchronised dict.

The fix is to add a lock and wrap all mutation logic inside it:

```python
import threading

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = {}
        self.lock = threading.RLock()  # Reentrant lock
        self.head = Node()
        self.tail = Node()
        self.head.next = self.tail
        self.tail.prev = self.head

    def get(self, key: int) -> int:
        with self.lock:
            if key not in self.cache:
                return -1
            node = self.cache[key]
            self._remove(node)
            self._insert_at_front(node)
            return node.value

    def put(self, key: int, value: int) -> None:
        with self.lock:
            # ... rest of put logic unchanged

```

I chose `RLock` (reentrant lock) over a plain `Lock` because it's safer if internal
helper methods ever call each other — a plain `Lock` would deadlock in that case.
The `with self.lock:` pattern also ensures the lock is released even if an exception
is raised mid-operation.

The downside is lock contention: under very high concurrency, threads queue up waiting
for the lock. For those scenarios, you could shard the cache (split it into segments,
each with its own lock) to reduce contention, but for most use cases a single lock is
sufficient and much simpler to reason about.

