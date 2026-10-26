# Day 7 (Week 2, Day 4): binary search, rotated array, binary search on the answer, replication

**Week 2:** binary search (DSA) · replication (design)
**Language:** Java
**Also:** blank-editor re-solve of Minimum Window Substring passed before this session (see Day 5 notes for the pattern).

## Key terms (quick glossary)
- **Binary search:** repeatedly discard half of the remaining candidates using one check; O(log n).
- **Invariant:** a fact about the search range that stays true after every iteration.
- **Pivot / drop:** in a rotated sorted array, the single place where a large value is followed by a small one.
- **Binary search on the answer:** binary search over the possible values of the answer, using a yes/no feasibility check.
- **Monotonic check:** once the check passes for a value, it passes for every larger value (or the reverse).
- **Replication:** keeping copies of the same data on several nodes.
- **Leader / follower:** the leader accepts writes; followers copy its changes and can serve reads.
- **Replication lag:** how far a follower is behind the leader.
- **Read-your-writes:** a user must see their own writes in later reads.
- **Split brain:** two nodes both believe they are the leader.

## DSA

### Binary search brushup
- If one check can discard half of the candidates, finish in about $\log_2 n$ steps (about 30 checks for a billion items).
- Safe midpoint: `mid = lo + (hi - lo) / 2` (avoids overflow of `lo + hi`).
- Two classic bugs: loops that never shrink (`lo = mid` or `hi = mid` can leave the range unchanged) and off-by-one errors (decide whether `hi` is inclusive and keep the loop condition consistent).

### 1. Binary Search (LeetCode 704)
**Brute force:** scan the array, O(n).
**Invariant:** if `target` is in `nums`, its index is inside `[lo, hi]`, both ends inclusive.
- `lo <= hi` is the loop condition because a range with `lo == hi` still holds one candidate.
- `mid` has been checked, so the range moves past it (`mid + 1`, `mid - 1`), which also guarantees the range shrinks.

```java
class Solution {
    public int search(int[] nums, int target) {
        int lo = 0, hi = nums.length - 1;
        while (lo <= hi) {
            int mid = lo + (hi - lo) / 2;
            if (nums[mid] == target) return mid;
            else if (nums[mid] < target) lo = mid + 1;
            else hi = mid - 1;
        }
        return -1;
    }
}
```
**Complexity:** O(log n) time, O(1) space.

### 2. Search in Rotated Sorted Array (LeetCode 33)
**Brute force:** scan, O(n).
**Why one half is always sorted:** a rotation creates exactly one drop, so the drop sits in at most one of the two halves, and the other half has no drop and is sorted.

```
[4, 5, 6, 7, 0, 1, 2]     drop between 7 and 0
 lo        mid      hi    mid = index 3 (value 7)
[4 5 6 7]  sorted         [7 0 1 2] has the drop, not sorted
```

**Detecting the sorted half:** `nums[lo] <= nums[mid]` means `[lo, mid]` is sorted (the `=` covers a one-element half, `lo == mid`). Otherwise `[mid, hi]` is sorted.
**Rule at each step:**
1. Find the sorted half.
2. Check whether `target` is inside that sorted half (a range check is reliable only there).
3. If yes keep that half, else keep the other half. Never search both.

**Invariant:** if `target` is in `nums`, its index is inside `[lo, hi]` (inclusive), and at every step at least one of `[lo, mid]` or `[mid, hi]` is sorted.

```java
class Solution {
    public int search(int[] nums, int target) {
        int lo = 0, hi = nums.length - 1;
        while (lo <= hi) {
            int mid = lo + (hi - lo) / 2;
            if (nums[mid] == target) return mid;
            if (nums[lo] <= nums[mid]) {                    // left half sorted
                if (nums[lo] <= target && target < nums[mid]) hi = mid - 1;
                else lo = mid + 1;
            } else {                                        // right half sorted
                if (nums[mid] < target && target <= nums[hi]) lo = mid + 1;
                else hi = mid - 1;
            }
        }
        return -1;
    }
}
```
**Complexity:** O(log n) time, O(1) space.
**Bugs I hit and fixed:**
- Searched the wrong range (`lo..mid`) when the right half held the target.
- `lo = mid` and `hi = mid` did not shrink the range (infinite loop on two elements).
- The sorted-left test used `<`, which misses the one-element half; use `<=`.
- A second `else if` could leave no branch to run; when the left half is not sorted the right one is, so use a plain `else`.
- The helper function was not needed; shrink `[lo, hi]` and let the same loop continue.
**Follow-up:** with duplicates (LeetCode 81), `nums[lo] <= nums[mid]` cannot tell which half is sorted (for example `[1,0,1,1,1]`), and the worst case becomes O(n).

### 3. Koko Eating Bananas (LeetCode 875), binary search on the answer
**Brute force:** try every speed `k` from 1 to the largest pile, compute total hours as the sum of $\lceil \text{pile} / k \rceil$ and accept the first with total at most `h`. Cost is $O(M \cdot n)$ where $M$ is the largest pile; with $M = 10^9$ and $n = 10^4$ that is about $10^{13}$ operations, far too slow.
**Key insight:** feasibility is monotonic. If speed `k` works, every larger speed works; if `k` fails, every smaller speed fails. So binary search over the speed.
**Invariant:** the answer is inside `[lo, hi]` (inclusive); `hi` always works, and every speed below `lo` fails.
- On "yes", `hi = mid` (mid might be the answer, so it stays in the range).
- On "no", `lo = mid + 1` (mid fails, so it is excluded).
- Stop when `lo == hi`: one candidate remains.

```java
class Solution {
    public int minEatingSpeed(int[] piles, int h) {
        int hi = 0;
        for (int p : piles) hi = Math.max(hi, p);
        int lo = 1;
        while (lo < hi) {
            int mid = lo + (hi - lo) / 2;
            if (canFinish(piles, h, mid)) hi = mid;
            else lo = mid + 1;
        }
        return lo;
    }

    private boolean canFinish(int[] piles, int h, int k) {
        long hours = 0;
        for (int p : piles) {
            hours += (p - 1) / k + 1;      // integer ceiling of p / k
            if (hours > h) return false;   // early exit
        }
        return true;
    }
}
```
Integer ceiling identity (for positive integers):

$$\lceil p / k \rceil = \left\lfloor \frac{p - 1}{k} \right\rfloor + 1$$

**Complexity:** $O(n \log M)$ time (each of about $\log_2 M$ steps scans all piles, plus $O(n)$ to find the maximum); with $M = 10^9$, $n = 10^4$ that is about $3 \times 10^5$ operations. $O(1)$ space.
**Learnings:**
- `float` has about 7 digits of precision and breaks for `pile = 1000000000`, `k = 499999986`. Use `double`, or better the integer ceiling above.
- My first complexity answer, $O(\log M + n)$, forgot that each step scans all piles. Iterations times cost of one iteration.
- Use `while (lo < hi)` and `return lo`; no early return inside the loop.
- Test cases: `[3,6,7,11]`/8 gives 4, `[30,11,23,4,20]`/5 gives 30, `/6` gives 23, `[1,1,1,1]`/4 gives 1, `[1000000000]`/2 gives 500000000, `[312884470]`/312884469 gives 2.

### Binary search on the answer: template
```java
int lo = smallestPossible, hi = largestUseful;   // hi must always work
while (lo < hi) {
    int mid = lo + (hi - lo) / 2;
    if (feasible(mid)) hi = mid;      // mid might be the answer
    else lo = mid + 1;                // mid fails
}
return lo;
```
Related problems: Capacity to Ship Packages Within D Days, Split Array Largest Sum, Minimum Days to Make Bouquets.

## System design: replication

### Concepts
- Reasons to replicate: survive node failure (availability), serve more reads, keep data near users, durability.
- **Leader-follower:** writes go to the leader, which streams changes to followers; reads can go to followers.
- **Synchronous:** the leader waits for the follower before acknowledging. No acknowledged write is lost on a leader crash; writes are slower; one slow follower can stall everything.
- **Asynchronous:** the leader acknowledges immediately. Fast writes; a leader crash can lose recent writes; followers lag.
- **Semi-synchronous:** one follower is synchronous, the rest asynchronous. A common compromise.
- **Failover:** detect leader failure (heartbeats and timeouts), promote the most up-to-date follower, repoint clients. Risks: lost writes (async) and split brain.

```
write -> leader --(sync)---> follower A   (acknowledged before the client is told)
            |
            +----(async)---> follower B   (may lag)
read  -> follower B                       (may return old data)
```

### Exercise: order missing after redirect to "My orders"
Setup: one primary, two read replicas, asynchronous replication. A user places an order and the next page reads from a replica.
1. **Why:** replication lag. The write went to the primary and was acknowledged; the replica has not applied it yet, so the read misses it. This breaks read-your-writes.
2. **Fixes:**
   - Sync or semi-sync changes when the write is acknowledged and improves durability, but by itself does not guarantee that the replica serving the read has the data (other replicas still lag, and "received" is not "applied"). Full sync also stalls writes if any replica is slow.
   - The usual fixes for this symptom are read routing: (a) read from the primary for a short window after the user's own write, or for the user's own critical data (such as "My orders"); (b) carry a position token (log position) from the write to the next read, and let a replica serve the read only if it has applied at least that position.
3. **Primary crashes right after confirming an order that reached no replica:** the order is lost. If that is unacceptable, use semi-sync (at least one follower has the data before the acknowledgement) and promote that follower on failover. Cost: extra write latency (a round trip plus the follower's disk write), and if the synchronous follower is slow or down, writes stall or degrade to async (which reopens the loss window).
- **Split brain:** the old primary comes back still believing it is the leader, so two nodes accept writes. Prevent it with a majority (quorum) election, epoch numbers (terms) and fencing so the old leader's writes are rejected.
- Next level: quorum replication with consensus (Raft, Paxos) puts every acknowledged write on a majority and elects the node with the latest log.

## Mistakes log entries (copy into mistakes-log.md)
- A binary search range must shrink every iteration: never `lo = mid` or `hi = mid` unless the loop condition guarantees progress (the `lo < hi` pattern with `hi = mid`).
- Sorted-half check in a rotated array: `nums[lo] <= nums[mid]`, and the other branch is a plain `else`.
- Never search both halves of a rotated array; test the sorted half and keep one half.
- Binary search on the answer: `hi` must always be feasible; on "yes" keep `mid`, on "no" exclude it.
- Complexity of a loop calling a scan is iterations times cost per call ($n \log M$, not $\log M + n$).
- `float` precision breaks large integer ceilings; use integer ceiling $\lfloor (p-1)/k \rfloor + 1$.
- Sync/semi-sync replication is about durability; read-your-writes needs read routing or a position token.
- Write the invariant as a fact about the range, not as a description of the mechanism.

## Plan
- Next: Week 2 mixed timed set, 3Sum re-solve from a blank editor, and the full chat design session (API, data model, architecture diagram, deep dives).
- Design topics coming: sharding, then queues and consistency.

## Questions to revisit
- Why is one half of a rotated array always sorted?
- Why is `hi = mid` (not `mid - 1`) correct on "yes" in Koko, and why does `lo < hi` terminate?
- Why does feasibility being monotonic make binary search on the answer valid?
- Why does semi-sync not fully solve read-your-writes?
- How do quorum and epochs prevent split brain?
