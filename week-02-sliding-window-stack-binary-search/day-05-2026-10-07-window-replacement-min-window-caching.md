# Day 5 — 2026-10-07 (Week 2, Day 2)

**Week 2:** sliding window with a budget / "need" counts (DSA) · caching fundamentals (design)
**Language:** Java

## Topics covered
- Variable window with a budget ("at most k changes")
- Sliding window with a `need[]` array and a `missing` counter
- Caching: hit ratio, patterns, invalidation, stampede, races, versioning

## Problems

### 1. Longest Repeating Character Replacement (LeetCode 424)
**Brute force:** for each start `i`, extend `j`, keep counts, check `length - maxCount <= k`. O(n²) x 26.
**Key insight:** changes needed for a window = `length - maxCount`. The window is valid when that is `<= k`.
**Invariant:** the window `[l, r]` can be turned into one repeated letter with at most `k` changes (after shrinking).

```java
class Solution {
    public int characterReplacement(String s, int k) {
        int[] counts = new int[26];
        int l = 0, best = 0;
        for (int r = 0; r < s.length(); r++) {
            counts[s.charAt(r) - 'A']++;
            while ((r - l + 1) - maxCount(counts) > k) {   // invalid: shrink one step at a time
                counts[s.charAt(l) - 'A']--;
                l++;
            }
            best = Math.max(best, r - l + 1);
        }
        return best;
    }

    private int maxCount(int[] counts) {
        int max = 0;
        for (int c : counts) max = Math.max(max, c);
        return max;
    }
}
```
**Complexity:** O(26·n) = O(n) time, O(1) space.
**Learnings:**
- Shrink one step at a time while invalid; no clever jump needed.
- My version used `if` instead of `while`. It still works because the window never gets shorter than the best answer so far (each step asks "can a window one longer than `best` exist here?"). `while` is easier to defend.
- Optional trick: keep a single `maxCount` that never decreases. A stale max can never make a window look better than it is, and the answer only grows when a larger max appears.
- I did not write the invariant comment. It is a habit for explaining out loud, not a code requirement.

### 2. Minimum Window Substring (LeetCode 76), Hard
**Brute force:** for each start, extend until all of `t` is covered, track the minimum. O(n²·c), O(1) space. The "copy and restore the counts" idea is just this brute force in disguise.
**Invariant:** `missing` = number of characters of `t` not yet covered by `[l, r]`; `missing == 0` exactly when the window covers `t`.

**How `need[]` and `missing` work:**
- `need` is `int[128]`, `need[c]` = how many more of `c` the window still needs. Start from the counts in `t`. `missing` starts at `t.length()`.
- `need[c] > 0` still needed; `== 0` exactly enough; `< 0` surplus (or `c` not in `t`).
- **Right pointer adds `c`:** if `need[c] > 0` then `missing--`; **always** `need[c]--`.
- **Left pointer removes `c`:** `need[c]++`; if `need[c] > 0` afterward then `missing++`.
- Always decrementing records the surplus, so shrinking knows how many copies it can drop before the window breaks.
- `need[]` is not all zeros when valid; it is "no entry above 0", which is what `missing == 0` means.
- Here you **shrink while the window is valid** (minimizing), unlike the earlier problems that shrank when invalid.

```java
class Solution {
    public String minWindow(String s, String t) {
        if (t.length() > s.length()) return "";
        int[] need = new int[128];
        for (char c : t.toCharArray()) need[c]++;
        int missing = t.length(), l = 0, bestStart = 0, bestLen = Integer.MAX_VALUE;
        // Invariant: missing = chars of t not yet covered by [l, r]; missing == 0 iff window covers t
        for (int r = 0; r < s.length(); r++) {
            if (need[s.charAt(r)]-- > 0) missing--;
            while (missing == 0) {
                if (r - l + 1 < bestLen) { bestLen = r - l + 1; bestStart = l; }
                if (++need[s.charAt(l++)] > 0) missing++;
            }
        }
        return bestLen == Integer.MAX_VALUE ? "" : s.substring(bestStart, bestStart + bestLen);
    }
}
```
**Complexity:** O(|s| + |t|) time (each pointer moves forward at most |s| times, O(1) per step), O(1) space (fixed 128-element array).
**Learnings:**
- Took about an hour; it is a Hard problem. The goal is to rebuild it from the idea, not to match speed yet.
- My first solution was correct but: called `substring` on every new minimum (save `bestStart`/`bestLen` instead), used string `"add"`/`"remove"` actions in the hot loop (use `++`/`--`), and did not state the complexity.
- Test cases: `"ADOBECODEBANC"`/`"ABC"` -> `"BANC"`, `"a"`/`"a"` -> `"a"`, `"a"`/`"aa"` -> `""`, `"aa"`/`"aa"` -> `"aa"`.

## System design: caching

### Why and the numbers
- Cache = small, fast copy of expensive-to-fetch data; works because access is skewed.
- **Database load = read QPS x (1 - hit ratio).** Hit ratio = hits / (hits + misses). Hit + miss ratios must add up to 100%.
- Memory = items x size x overhead (1.3-2x). Reloads per second from expiry ~ items / TTL.
- Catalog exercise (50K reads/s, 500 writes/s, DB limit ~5K QPS, 2M products x ~5 KB):
  - Required hit ratio: 90% from reads alone; **91%** once writes use 500 of the 5K capacity (allowed miss ratio 4.5K/50K = **9%**; I once wrote 0.9% by mistake).
  - Memory: 10 GB raw, **13-20 GB** provisioned (fits one Redis node; add replicas for availability).
  - TTL is a database-protection decision: 2M / 1 hour is about 550 reloads/s (fine); 2M / 1 minute is about 33K/s (overwhelms the DB).

### Where to cache
Client/browser, CDN, API gateway, in-process (fast but per-server copies disagree), distributed cache (Redis/Memcached, shared, sub-millisecond). The cache is never the source of truth.

### Patterns
- **Cache-aside (default for read-heavy):** read cache; on miss read DB and fill; on write update DB and **delete** the entry.
- **Write-through:** write cache and DB together; cache always fresh; slower writes; may cache data nobody reads; two writes can disagree.
- **Write-back:** write cache, flush later; fast, but data loss if the cache dies.
- **Write-around:** write DB only; reads fill the cache.
- Why cache-aside with delete: deleting can never leave a wrong value; written items are mostly not read soon, so write-through wastes memory.

### Stampede and the reload lock
- Hot key expires, thousands of requests miss and hit the DB at once.
- Fix: only one request reloads the key (**request coalescing / single flight**); waiters wait briefly and re-read the cache, or return a stale value. The lock needs a **timeout** so a crashed loader cannot block everyone.
- The lock guards the **reload path** (miss, DB read, cache fill), not DB writes and not cache reads. A single Redis `SET` is atomic anyway.
- Lock location: per-server map (simple; at most one reload per server) or Redis `SET lock:key NX EX 5` (one reload fleet-wide, extra Redis call per miss). Start per-server.
- Also: TTL jitter, refresh early.

### The cache-aside race and fixes
- Reader loads old row; writer updates DB and deletes the cache entry; reader then fills the cache with the old value. Stale until TTL. Rare (a few ms window), but real.
- **Short TTL:** bounds staleness, does not remove the race, and too short overloads the DB.
- **Delayed second delete:** delete again after ~0.5-1 s to remove a stale fill. Cheap, narrows the window, not airtight.
- **Versioning:** compare the version stored in the cache (not the DB) with an atomic compare-and-set (Redis Lua). Plain delete leaves no version to compare, so pair it with either a versioned write (write-through for simple data) or a **short-lived version marker** (better when the cached object is derived/large, writes are many, and most written items are never read).
- Ranking: TTL baseline, delayed double delete next, versioned writes when correctness matters.

### Stale price vs stale stock (Step 3, question 2)
- Browsing/display: a price a few seconds or minutes stale is usually fine. Stock shown as "in stock / low stock" can be approximate.
- **Checkout reads the source of truth:** re-read the authoritative price and charge that; reserve stock with an atomic conditional update in the database (`UPDATE ... SET stock = stock - 1 WHERE stock >= 1`). Never trust the cache for transactions.
- Write-through is **not** a guarantee of correctness: with two concurrent writers the DB may see A then B while the cache sees B then A, leaving a stale value until TTL. Versions fix that.
- Money-critical live data (market prices) comes from a streaming feed or the source of truth, not a plain DB cache.

## Design fundamentals plan
- Five caching ideas to know cold: why (hit ratio formula), where, cache-aside, staleness control (TTL + delete on write), stampede.
- One building block per two days, each ending with a 60-second explanation in my own words.
- Reading: Designing Data-Intensive Applications, chapters 1, 5, 6 over the next few weeks.

## Mistakes log entries (copy into mistakes-log.md)
- Hit ratio and miss ratio must sum to 100% (4.5K / 50K is 9%, not 0.9%).
- Sliding window shrink: when minimizing, shrink while valid; when maximizing, shrink while invalid.
- Do not recreate state by "copy and restore"; undo one step per pointer move.
- Do not call `substring` repeatedly; save start and length.
- Write-through does not make a cache correct under concurrent writers.
- A cache fill needs an atomic version check, or the stale value wins.

## Plan changes
- Saturday's 1-hour re-solve: **Minimum Window Substring** from a blank editor (instead of 3Sum). 3Sum re-solve moves to Monday.

## Questions to revisit
- Why does always-decrementing `need[c]` make shrinking correct?
- Why is the reload lock needed, and why does it need a timeout?
- Why is deleting safer than writing the value on a database write?
- What stays authoritative at checkout, and why?
