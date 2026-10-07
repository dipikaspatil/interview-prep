# Day 4 — 2026-10-06 (Week 2, Day 1)

**Week 2:** sliding window (DSA) · load balancing, stateless vs stateful (design)
**Language:** Java
**Break before this day:** 3 days on ML Zoomcamp; prep resumed with a blank-editor Valid Palindrome re-solve (see Day 3 notes).

## Topics covered
- Sliding window: fixed vs variable size, invariant, why it is O(n)
- Load balancing: algorithms, L4 vs L7, health checks, avoiding a single point of failure
- Stateless vs stateful services, session handling

## Sliding window brushup
- Window `[l, r]` over an array or string. Expand `r`; shrink `l` while the window breaks a rule.
- `l` and `r` only move forward, so each element enters and leaves once, giving O(n).
- Trigger words: longest/shortest substring or subarray, "at most k", contiguous.
- Habit: write the **invariant** as a comment ("the window [l, r] always contains ___") and say the **brute force** before coding.

```java
int l = 0;
for (int r = 0; r < n; r++) {
    // add nums[r] to window state
    while (/* window invalid */) {
        // remove nums[l] from window state
        l++;
    }
    // window [l, r] is valid: update answer
}
```

## Problems

### 1. Best Time to Buy and Sell Stock (LeetCode 121)
**Brute force:** every buy/sell pair, O(n²) time, O(1) space.
**Idea:** track a buy index; if `prices[sell] <= prices[buy]`, today is at least as cheap, so buying today is never worse than buying on the old day for any later sell day. Move `buy` there.

```java
class Solution {
    public int maxProfit(int[] prices) {
        int buy = 0, max = 0;
        for (int sell = 1; sell < prices.length; sell++) {
            if (prices[sell] <= prices[buy]) {
                // today is at least as cheap: buying today is never worse for later sell days
                buy = sell;
            } else {
                max = Math.max(max, prices[sell] - prices[buy]);
            }
        }
        return max;
    }
}
```
**Complexity:** O(n) time, O(1) space.
**Learning:** the reasoning must be about the *buy* day being replaced by a better candidate. My first comment mixed up "buy" and "sell". Keep comments short enough to say aloud.

### 2. Longest Substring Without Repeating Characters (LeetCode 3)
**Brute force:** check every substring for duplicates, O(n²) to O(n³).
**Invariant:** the window `[start, end]` always contains distinct characters.

```java
class Solution {
    public int lengthOfLongestSubstring(String s) {
        int start = 0, max = 0;
        Map<Character, Integer> last = new HashMap<>();   // char -> last seen index
        for (int end = 0; end < s.length(); end++) {
            char ch = s.charAt(end);
            if (last.containsKey(ch)) start = Math.max(start, last.get(ch) + 1);
            last.put(ch, end);
            max = Math.max(max, end - start + 1);
        }
        return max;
    }
}
```
**Complexity:** O(n) time. Space O(min(n, m)) where m is the alphabet size (O(1) only if you state a bounded alphabet such as ASCII).
**Learnings:**
- `"abba"` is the bug-catching input: the earlier `a` is no longer inside the window, so `start` must never move backward. `Math.max(start, ...)` handles it.
- Compute the length every iteration to avoid special-casing the end of the loop.
- I skipped the brute force and the invariant comment twice. Say both before any code.

### 3. Group Anagrams re-solve (LeetCode 49), blank editor
**Brute force:** compute character counts per string, compare every pair, O(n²) comparisons of 26-length arrays.
**Key:** the 26 counts joined with a separator. Anagrams have identical counts, so identical keys. Different counts give different keys because of the separator (`1-11-` vs `11-1-`).

```java
class Solution {
    public List<List<String>> groupAnagrams(String[] strs) {
        Map<String, List<String>> map = new HashMap<>();
        for (String str : strs) {
            map.computeIfAbsent(getKey(str), x -> new ArrayList<>()).add(str);
        }
        return new ArrayList<>(map.values());
    }

    private String getKey(String str) {
        int[] counts = new int[26];                       // assumes lowercase a-z
        for (char ch : str.toCharArray()) counts[ch - 'a']++;
        StringBuilder sb = new StringBuilder();
        for (int c : counts) sb.append(c).append('-');
        return sb.toString();
    }
}
```
**Complexity:** O(n·m) time (n strings, max length m: each `getKey` costs O(m + 26)). Space: O(n) if you count only references (Java does not copy the input strings) plus O(u) short keys; the common stated answer is O(n·m), since the output holds all the characters. Either is fine if you say what you are counting.
**Learnings:**
- Overall time was written as "O(m + n)". It is a **product**, not a sum: the outer loop runs n times and each iteration costs O(m). Rule: complexity = number of iterations x cost of one iteration.
- The separator was remembered without prompting, so the Day 2 lesson held.
- For the key explanation, say *why* it is unique, not just that it gives O(1) lookup.

## System design

### Load balancing
- **Where:** in front of the servers, between clients and the service tier (client -> DNS -> load balancer -> servers). Large systems also balance between internal tiers.
- **Algorithms:** round robin (cycles in order), weighted round robin, least connections / least response time (when request cost varies), hash-based (affinity, caches).
- **Mistake to avoid:** hashing by request type sends one type to one server and creates hot spots. Routing by type (`/api/*` vs `/images/*`) is **path-based routing**, an L7 feature that picks a server *pool*. The balancing algorithm then spreads load *within* the pool.
- **Health checks:** probe a `/health` endpoint and remove unhealthy servers from rotation.
- **Single point of failure:** run several balancers active-active across availability zones; use DNS health checks or anycast (or a floating IP with failover). A stateless service needs no balancer-to-balancer state sync.

### L4 vs L7
- **Layer 4 (transport, TCP/UDP):** sees IPs and ports only. Fast, cheap, works for any protocol, no content-based decisions. AWS: Network Load Balancer.
- **Layer 7 (application, HTTP):** reads path, headers, cookies. Path routing, TLS termination, auth checks, cookie stickiness. More CPU. AWS: Application Load Balancer.
- Large systems often put L4 in front of a fleet of L7 balancers.
- Analogy: L4 reads the address on the envelope; L7 opens it.

### Stateless vs stateful
- **State** = anything a server remembers between requests.
- **Stateless:** each request carries what is needed; any server can handle it; scale, failover and deploy freely.
- **Stateful:** the server keeps earlier-request data (in-memory sessions, WebSocket connections, databases).
- **Problem:** in-memory sessions behind round robin cause random logouts (request 1 on server A, request 2 on server B). Round robin *cycles* in order; the effect is the same.
- **Fixes:** (1) shared session store such as Redis with TTL (default choice); (2) sticky sessions via cookie (quick fix; uneven load, sessions lost if the server dies); (3) signed tokens such as JWT (no server state, hard to revoke).
- State does not disappear, it moves: stateless services over stateful stores (DB, cache, object storage).
- Chat design link: WebSocket gateways are stateful (a `user -> gateway` registry in Redis); the message service is stateless; DB and queues hold durable state.

## Mistakes log entries (copy into mistakes-log.md)
- Complexity of a loop with work inside = iterations x cost per iteration (n·m, not m + n).
- Sliding window: `start = Math.max(start, last + 1)`, so the start never moves backward ("abba").
- Hashing by request type creates hot spots; route by path to a pool, balance within it.
- Load balancer sits in front of servers, not after the application layer.
- Say brute force and invariant before coding.

## Still open
- Full chat design session (API, data model, architecture diagram, deep dives), planned for Monday

## Questions to revisit
- Why is moving `buy` to a cheaper day always safe?
- Why must `start` never move backward in Longest Substring?
- Why is path-based routing different from the balancing algorithm?
- Name three ways to keep sessions working across many servers, with one downside each.
