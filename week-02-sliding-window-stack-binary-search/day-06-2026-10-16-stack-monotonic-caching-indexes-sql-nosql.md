# Day 6 (Week 2, Day 3): stacks, monotonic stack, caching recap, indexes, SQL vs NoSQL

**Week 2:** stack pattern (DSA) · caching recap, database indexes, SQL vs NoSQL (design)
**Language:** Java

## Key terms (quick glossary)
- **Stack:** last-in, first-out (LIFO) structure; the most recently added item is removed first.
- **Monotonic stack:** a stack kept sorted in one direction by popping items that break the order before each push.
- **Hit ratio:** share of reads served by the cache, hits divided by total reads.
- **Index:** a separate sorted structure (usually a B-tree) that lets the database find rows without scanning the whole table.
- **Composite index:** an index on several columns, sorted by the first column, then the second within it.
- **ACID:** atomicity, consistency, isolation, durability; the guarantees of a database transaction.
- **Partition key / clustering key:** in Cassandra, the key that decides which node stores a row, and the key that sorts rows inside that partition.

## DSA

### Stack brushup
- A stack fits "handle the most recent unmatched thing first": nested brackets, undo, next greater element.
- Use `ArrayDeque`, not the legacy `Stack` class (`Stack` extends `Vector`, is synchronized, and exposes index operations that break the stack idea).
- `ArrayDeque.peek()` returns `null` on empty, and `pop()` throws; check `isEmpty()` first. Unboxing a `null` `Integer` in a comparison throws a `NullPointerException`.
- Push, pop and peek are O(1); a single pass is O(n).

### 1. Valid Parentheses (LeetCode 20)
**Brute force:** repeatedly remove matched pairs `()`, `{}`, `[]` until nothing changes; O(n²).
**Invariant:** the stack holds the closers still expected, in the order they must arrive.

```java
class Solution {
    public boolean isValid(String s) {
        Deque<Character> stack = new ArrayDeque<>();
        for (char c : s.toCharArray()) {
            if (c == '(') stack.push(')');
            else if (c == '{') stack.push('}');
            else if (c == '[') stack.push(']');
            else if (stack.isEmpty() || stack.pop() != c) return false;
        }
        return stack.isEmpty();
    }
}
```
**Complexity:** O(n) time, O(n) space.
**Learnings:**
- My original solution was correct but pushed a stray closer onto an empty stack, which only worked because the final emptiness check then failed. Return `false` immediately instead.
- Pushing the expected closer turns matching into one equality check.

### 2. Min Stack (LeetCode 155)
**Simple approach:** move elements between two stacks to find the minimum; `getMin()` is O(n), which is not allowed.
**Invariant:** the top of `minStack` is always the minimum of everything currently in `stack`, and `minStack` is non-increasing from bottom to top.
**Idea:** push to `minStack` when it is empty or `value <= top` (use `<=` so duplicates are tracked). On `pop`, pop `minStack` too if the values are equal.

```java
class MinStack {
    Deque<Integer> stack = new ArrayDeque<>();
    Deque<Integer> minStack = new ArrayDeque<>();

    public void push(int value) {
        stack.push(value);
        if (minStack.isEmpty() || value <= minStack.peek()) minStack.push(value);
    }

    public void pop() {
        int value = stack.pop();
        int minVal = minStack.peek();     // unbox into int before comparing
        if (value == minVal) minStack.pop();
    }

    public int top() { return stack.peek(); }

    public int getMin() { return minStack.peek(); }
}
```
**Complexity:** O(1) per operation; O(n) extra space in the worst case (values arriving in non-increasing order).
**Bugs caught:**
- With a strict `<`, the sequence `push(2), push(2), pop(), getMin()` loses the minimum. Use `<=`.
- Comparing two `Integer` objects with `==` compares references; it works only for small cached values. Unbox to `int` (or use `equals`).
- Follow-up to know: a single stack of pairs `(value, minSoFar)` also works.

### 3. Daily Temperatures (LeetCode 739), monotonic stack
**Brute force:** for each day scan forward for a warmer day; O(n²). The worst case is a **decreasing** (or all-equal) array, not an increasing one.
**Invariant:** the stack holds the indices of days still waiting for a warmer day, and their temperatures are non-increasing from bottom to top.

```java
class Solution {
    public int[] dailyTemperatures(int[] temperatures) {
        Deque<Integer> stack = new ArrayDeque<>();   // indices of days still waiting
        int[] result = new int[temperatures.length];
        for (int i = 0; i < temperatures.length; i++) {
            while (!stack.isEmpty() && temperatures[i] > temperatures[stack.peek()]) {
                int idx = stack.pop();
                result[idx] = i - idx;
            }
            stack.push(i);
        }
        return result;     // days left on the stack keep 0
    }
}
```
**Complexity:** O(n) time (each index is pushed once and popped at most once, so about 2n steps in total even with the inner loop), O(n) space.
**Learnings:**
- Store **indices**, not values: the answer is a distance, and the value is always available as `temperatures[idx]`.
- Use strict `>`: an equal temperature is not warmer.
- Write-up slip: I wrote runtime O(1); it is O(n).

### Monotonic stack explained
Two flavors:

| Stack kind | Pop condition | Finds |
|---|---|---|
| Decreasing (non-increasing bottom to top) | new value is greater than the top | next greater element |
| Increasing (non-decreasing bottom to top) | new value is less than the top | next smaller element |

Template:

```java
Deque<Integer> stack = new ArrayDeque<>();   // indices
for (int i = 0; i < n; i++) {
    while (!stack.isEmpty() && /* nums[i] breaks the order against nums[stack.peek()] */) {
        int j = stack.pop();
        // nums[i] is the answer for index j
    }
    stack.push(i);
}
```

Trace on `[73, 74, 75, 71, 69, 72, 76, 73]` (stack shows temperatures, bottom to top):

```
day 0 (73): stack [73]
day 1 (74): pops day 0 -> answer 1      stack [74]
day 2 (75): pops day 1 -> answer 1      stack [75]
day 3 (71): no pop                      stack [75, 71]
day 4 (69): no pop                      stack [75, 71, 69]
day 5 (72): pops day 4 (1), day 3 (2)   stack [75, 72]
day 6 (76): pops day 5 (1), day 2 (4)   stack [76]
day 7 (73): no pop                      stack [76, 73]
result: [1, 1, 4, 2, 1, 1, 0, 0]
```

Other problems that use it: Next Greater Element, Largest Rectangle in Histogram, Trapping Rain Water, Online Stock Span.

## System design

### Caching recap (rewritten from memory, with corrections)
- **Why:** lower latency and protect the database. The formula to remember:

$$\text{DB read load} = \text{read QPS} \times (1 - \text{hit ratio})$$

  Writes always go to the database, so add the write QPS on top. Example: 20,000 reads per second at a 90% hit ratio sends 2,000 to the database; at 99% only 200. Nine more points of hit ratio gives 10x less read load.
- **Where (closest to farthest):** browser (HTTP cache headers, hard to invalidate), CDN (edge servers near the user), in-process (fastest, but each server has its own copy), Redis (shared by all servers, sits beside the application servers, not near DNS), database internals.
- **Cache-aside:** read the cache; on a miss read the database and fill the cache; on a write update the database and delete the entry. A lock (request coalescing, with a timeout) lets one request reload a hot key while others wait.
- **Staleness control:** TTL (each entry expires after a fixed time), delayed second delete, versioned fills.
- **Failure mode:** a slow reader fills the cache with an old value after a writer's delete. Reject the old fill by comparing versions atomically.

Layers diagram:

```
client -> browser cache -> CDN -> load balancer -> app server (in-process cache)
                                                        |
                                                     Redis (shared)
                                                        |
                                                     database
```

Model 60-second answer: "We cache to cut latency and protect the database: database read load equals read QPS times one minus the hit ratio. Caches live at the browser, the CDN, inside the app process, and in a shared Redis tier. I use cache-aside: read the cache, on a miss load from the database and fill it, and on a write update the database and delete the entry. TTLs bound staleness, a delayed second delete or versioned fills handle the read-write race, and request coalescing with a lock timeout prevents stampedes on hot keys."

### Database indexes
- Without an index, `WHERE email = ?` scans the table: O(n). A B-tree index makes the lookup O(log n). A node holds hundreds of keys, so even a billion rows need about 4 levels.
- Costs: extra storage, slower writes (every insert, update and delete must update each index), helps only queries that filter or sort on the indexed columns.
- Weak candidates: columns with few distinct values, tiny tables, rarely-run queries on write-heavy tables, and values shared by most rows.

Exercise: `orders` has 100M rows and this query takes 5 seconds:

```
SELECT * FROM orders WHERE user_id = ? ORDER BY created_at DESC LIMIT 20
```

- An index on `user_id` alone still reads and sorts all of that user's rows.
- Better: a **composite index on (user_id, created_at DESC)**. The database jumps to the user's section, the newest orders are already first, and it reads just 20 entries.
- Order matters: equality column first, sort or range column second.
- **Leftmost-prefix rule:** a composite index is used efficiently only for a leading part of its columns, in order. `WHERE user_id = ? ORDER BY created_at DESC LIMIT 20` uses both columns (ideal). `WHERE user_id = ?` uses the first column (still efficient). `WHERE created_at > ...` skips the first column, so it cannot seek; add a separate index on `created_at` if needed.
- Wording fix: "partitioning" means splitting data across shards; what the composite index does is sort order inside the index.

Index layout sketch:

```
(user_id, created_at)  sorted entries:
  user 7  | 2026-01-03
  user 7  | 2026-02-11
  user 7  | 2026-03-02   <- newest orders for user 7 are at the end of this section
  user 8  | 2026-01-09
  user 8  | 2026-02-20
  ...
Query "user 8, newest 20" jumps straight to user 8's section.
Query "created_at > X" has no user_id to seek on, so entries of every user must be checked.
```

### SQL vs NoSQL
- **SQL (PostgreSQL, MySQL):** fixed schema, joins, ACID transactions, strong consistency on the primary; scales up a long way and reads scale out with replicas; sharding is harder because of joins and cross-shard transactions.
- **NoSQL families:** key-value (DynamoDB, Redis), document (MongoDB), wide-column (Cassandra), graph (Neo4j). Built for horizontal scale and flexible schemas; usually no joins, limited transactions, and access patterns designed up front.
- **Default:** relational unless there is a specific reason not to (massive write volume, simple key-based access, flexible attributes, multi-region distribution). "SQL does not scale" is not a reason.

Exercise results:

| Case | Choice | Reason |
|---|---|---|
| Bank transfers | SQL | ACID transactions, strict consistency |
| Chat message history (about 4 billion messages a day, read by conversation, newest first) | Cassandra | Huge write throughput; partition key is the conversation, clustering key is time (or sequence number) descending, so "newest first" is a sequential read within one partition; bucket by time so partitions stay bounded. My first reason ("long messages, long column") was off. |
| Product catalog with different attributes per category | MongoDB (document) | Flexible JSON per category. Alternative: PostgreSQL with a JSONB column. Searching across many attributes may need a search engine. |
| Login session store with expiry | Redis (or DynamoDB with TTL) | Key is a random session ID, value is session data (user id, roles); set a TTL so the store deletes it automatically. No "expired/not-expired" flag, and never store login credentials in the session. |

## Mistakes log entries (copy into mistakes-log.md)
- Do not push a stray closer on an empty stack; return `false`.
- Min stack: push on `<=`, not `<`; compare unboxed `int` values, not `Integer` references.
- Brute force worst case for next-greater problems is a decreasing array, not an increasing one.
- A loop inside a loop is not automatically O(n²); count pushes and pops (about 2n).
- Use read QPS, not total QPS, in the cache formula.
- Composite index order: equality column first, sort column second; leftmost-prefix rule.
- Use "sort order inside the index", not "partitioning", for composite index behavior.
- Session store key is a session ID, never credentials.

## Plan
- Next: blank-editor re-solve of **Minimum Window Substring** (1 hour), then Binary Search day (Binary Search, Search in Rotated Sorted Array, Koko Eating Bananas), then the full chat design session.
- Design fundamentals: one building block at a time, each ending with a 60-second explanation from memory.

## Questions to revisit
- Why does the monotonic stack hold indices, and why is it still O(n)?
- Why does `<=` matter in Min Stack?
- Why does a composite index on `(user_id, created_at)` not help `WHERE created_at > X`?
- Why Cassandra for chat history, in terms of partition key and clustering key?
