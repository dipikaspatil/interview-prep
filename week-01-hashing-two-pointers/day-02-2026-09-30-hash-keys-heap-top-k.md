# Day 2 — 2026-09-30

**Week 1:** Hashing, heaps (DSA) · system design planned but not started
**Language:** Java

## Topics covered
- Re-solving from a blank editor (derive, don't recall)
- Building hash keys from counts (separator rule)
- Heap of size k for "top k" problems
- Complexity: cost *inside* each loop iteration

## Problems

### 1. 3Sum re-solve (LeetCode 15)
Solved from scratch, with one bug that I found by tracing `[-1,0,2]`.
**Bug:** called `twoSum(result, i, ...)`, so the pair search started at the *same* index as the fixed number and one element was used twice (`[-1,0,2]` returned `[[-1,-1,2]]`, expected `[]`).
**Fix:** the search for "the rest" starts at `i + 1`.
**Why my tests missed it:** all four sample tests passed by accident, so I need to invent the smallest *failing* input myself (a case with a single copy of a value).

(Final code is in Day 1's 3Sum.)

### 2. Group Anagrams (LeetCode 49)
**Idea:** key = character-count array turned into a string; anagrams share a key.
**Complexity:** O(n·k) time, O(n·k) space (n strings, max length k). The sorted-key alternative is O(n·k log k).

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
        int[] counts = new int[26];
        for (int i = 0; i < str.length(); i++) counts[str.charAt(i) - 'a']++;
        StringBuilder sb = new StringBuilder();
        for (int c : counts) sb.append(c).append('-');   // separator is essential
        return sb.toString();
    }
}
```
**Bug:** without a separator, `"abbbbbbbbbbb"` (1 a, 11 b) and `"aaaaaaaaaaab"` (11 a, 1 b) both give `111000...`.
**Rule:** when building a string key from several numbers, add a separator.
**Trade-off:** count key is faster for long strings with a small known alphabet. Sorted key is simpler and works for any character set. `n` cancels out of the comparison, so the real factors are `k` and the alphabet.
**Alphabet question:** ASCII → `new int[128]` (or 256), no `- 'a'` offset. Full Unicode → use the sorted key.

### 3. Top K Frequent Elements (LeetCode 347), heap version
**Idea:** count with a map, then keep a **min-heap of size k** ordered by frequency; evict the top when something more frequent arrives.

```java
class Solution {
    public int[] topKFrequent(int[] nums, int k) {
        Map<Integer, Integer> map = new HashMap<>();
        for (int num : nums) map.merge(num, 1, Integer::sum);

        PriorityQueue<int[]> minHeap = new PriorityQueue<>((a, b) -> Integer.compare(a[1], b[1]));
        for (Map.Entry<Integer, Integer> e : map.entrySet()) {
            if (minHeap.size() < k) {
                minHeap.add(new int[]{e.getKey(), e.getValue()});
            } else if (minHeap.peek()[1] < e.getValue()) {
                minHeap.poll();
                minHeap.add(new int[]{e.getKey(), e.getValue()});
            }
        }

        int[] result = new int[k];
        int i = 0;
        while (!minHeap.isEmpty()) result[i++] = minHeap.poll()[0];
        return result;
    }
}
```
**Complexity (u = unique values):**
- Counting: O(n) time, O(u) space
- Heap loop: O(u log k) time, O(k) space (the heap never holds more than k items)
- Draining: O(k log k)
- **Total: O(n log k) time, O(u + k) space**

**Learnings:**
- Min-heap for "top k largest": the top is the weakest of the current k, so it's the one to evict.
- Don't reuse `k` for "unique elements"; use `u`.
- Complexity habit: for each loop, ask "what does *one iteration* cost?" and multiply.

## Still open (carry to Day 3)
- Top K Frequent, **bucket sort** version (`n + 1` buckets indexed by frequency, walk from high to low)
- System design: estimation exercise (photo-sharing app, 10M DAU)
- System design: clarifying questions for "Design a chat app"
- Product of Array Except Self (LeetCode 238)

## Questions to revisit
- Why is the heap size k and not u? What does that change in the complexity?
- Why must a count-based key have a separator?
