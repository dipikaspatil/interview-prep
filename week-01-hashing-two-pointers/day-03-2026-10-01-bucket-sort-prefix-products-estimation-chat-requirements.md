# Day 3 — 2026-10-01 (Valid Palindrome re-solve done 2026-10-06 after a short break)

**Week 1:** hashing, bucket sort, prefix/suffix products (DSA) · estimation + requirements (design)
**Language:** Java

## Topics covered
- Bucket sort for "top k frequent" (frequency used directly as an array index)
- Prefix and suffix products (no division)
- Back-of-envelope estimation (QPS, storage, bandwidth)
- Clarifying questions and requirements for "Design a chat app"
- Idempotent writes for "never lost, never duplicated" delivery

## Problems

### 1. Top K Frequent Elements (LeetCode 347), bucket sort version
**Idea:** the highest possible frequency is `n`, so make `n + 1` buckets indexed by frequency. `bucket[f]` holds the values that appear exactly `f` times. Walk from index `n` down and stop at `k` values.

```java
class Solution {
    public int[] topKFrequent(int[] nums, int k) {
        int n = nums.length;

        // 1. Count frequencies: time O(n), space O(u)  (u = unique values)
        Map<Integer, Integer> freq = new HashMap<>();
        for (int num : nums) freq.merge(num, 1, Integer::sum);

        // 2. Buckets indexed by frequency: space O(n) for n + 1 lists
        List<List<Integer>> buckets = new ArrayList<>();
        for (int i = 0; i <= n; i++) buckets.add(new ArrayList<>());

        // 3. Place each value in its frequency bucket: time O(u), space O(u) total
        for (Map.Entry<Integer, Integer> e : freq.entrySet()) {
            buckets.get(e.getValue()).add(e.getKey());
        }

        // 4. Walk high to low, stop at k: time O(n + k) worst case
        int[] result = new int[k];
        int j = 0;
        for (int i = n; i > 0; i--) {
            for (int num : buckets.get(i)) {
                result[j++] = num;
                if (j == k) return result;
            }
        }
        return result;
    }
}
```
**Complexity:** O(n) time, O(n) space.

**Learnings:**
- Space is **not** O(n²) just because lists are nested. Count the total items stored: n + 1 empty lists plus u values, so O(n).
- The early exit saves work on values, but not on empty buckets. Example: all-unique array with k = 1 still scans about n buckets, so the walk is O(n + k).
- The nested loop is not O(n²), because each value is visited once across all buckets.

**Heap vs bucket:**

| | Heap (size k) | Bucket sort |
|---|---|---|
| Time | O(n log k) | O(n) |
| Space | O(u + k) | O(n) |
| Pick when | k is much smaller than u, or memory matters | Best time wanted, frequencies bounded by n |

### 2. Product of Array Except Self (LeetCode 238)
**Idea:** left pass stores the product of everything to the left of `i`. Right pass multiplies in a running product of everything to the right. Use `rightProduct` *before* folding in `nums[i]`.

```java
class Solution {
    public int[] productExceptSelf(int[] nums) {
        int[] answer = new int[nums.length];

        // left pass: answer[i] = product of nums[0..i-1]
        answer[0] = 1;
        for (int i = 1; i < nums.length; i++) {
            answer[i] = answer[i - 1] * nums[i - 1];
        }

        // right pass: multiply in the product of nums[i+1..n-1]
        int rightProduct = 1;
        for (int i = nums.length - 2; i >= 0; i--) {
            rightProduct *= nums[i + 1];
            answer[i] *= rightProduct;
        }
        return answer;
    }
}
```
**Complexity:** O(n) time. O(1) extra space (O(n) including the required output array).

**Learnings:**
- Brute force is O(n²) time with O(1) extra space.
- Division breaks on zeros: with one zero, the zero's own index needs the product of the others and `nums[i] = 0` crashes the division. With two or more zeros, every answer is 0 but the divide still crashes.
- Pattern: "everything except me" → prefix and suffix. Same family as prefix sums and trapping rain water.

### 3. Valid Palindrome re-solve (LeetCode 125), blank editor
```java
class Solution {
    public boolean isPalindrome(String s) {
        int i = 0, j = s.length() - 1;
        while (i < j) {
            if (!Character.isLetterOrDigit(s.charAt(i))) { i++; continue; }
            if (!Character.isLetterOrDigit(s.charAt(j))) { j--; continue; }
            if (Character.toLowerCase(s.charAt(i)) != Character.toLowerCase(s.charAt(j))) return false;
            i++; j--;
        }
        return true;
    }
}
```
**Complexity:** O(n) time, O(1) space.
**Learning:** the redundant `isWhitespace` check and the `i`/`j` typo from Day 1 are gone. Reasoning came back without notes.

## System design

### Estimation: photo-sharing app (10M DAU, 2 uploads/day, ~2 MB each)
- Uploads: 20M/day. Write QPS: 20M / 100K s = **200 average, 400-600 peak**.
- Storage: 20M x 2 MB = **40 TB/day**, about **16 PB/year** (about 15 PB at 365 days).
- Ingress: 200 x 2 MB = about 400 MB/s average, 0.8-1.2 GB/s at peak.
- With 3x replication: about 45-50 PB/year raw, before thumbnails.
- Habit: convert to the biggest sensible unit (TB, PB).

**Design implications:**
- Photo bytes go to **object storage from the first upload**, not the app database. The database holds metadata only.
- Tiering by access pattern via lifecycle rules (infrequent-access, archive), not a fixed 7 days. CDN for hot reads.
- Clients upload **directly to object storage** with a **pre-signed URL** (multipart or resumable), so bytes bypass the app servers.
- Upload-complete event writes the metadata row and enqueues thumbnail/resize jobs.

### Chat app: clarifying questions and requirements
**Question feedback:**
- Don't ask "what is a chat app?" Ask scope instead: WhatsApp-like or Slack-like?
- Ask "can I assume authentication exists?" to skip login.
- Turn yes/no questions into design-driving ones: ordering per conversation or global? What delivery guarantee?
- Always ask the max group size (10 vs 100,000 are different designs).

**Interviewer answers:** 100M DAU, 40 messages/user/day, mostly 1:1, groups up to 500, order guaranteed within a conversation, never lost and never duplicated, history stored indefinitely.

**Functional:** phone-contact based, 1:1 and group chat (max 500), text/images/voice/emoji, indefinite history, delivered and read receipts.
**Non-functional:** 100M DAU x 40 msgs, ordering within a conversation, never lost/duplicated, offline messages stored and delivered on reconnect, end-to-end encryption, plus latency target (e.g. under ~500 ms to online users), availability (e.g. 99.99%), durability of acknowledged messages.
**E2EE trade-off:** the server only stores ciphertext, so no server-side search, and multi-device sync needs key handling.

**Numbers:** 4x10^9 messages/day, about **40K QPS average, 80-120K peak**. Text storage about 400 GB/day (about 150 TB/year). Group messages fan out to up to 500 recipients, so delivery rate is far above send rate.

### "Never lost, never duplicated"
- `sender + receiver + timestamp` is a weak ID: same-millisecond messages collide (would drop a real message), and phone clocks drift so retries may differ.
- Client generates a unique **message ID** (UUID/ULID) **once** and reuses it on every retry.
- Server write is **idempotent**: unique on `(conversation, sender, client_message_id)`. A repeat just returns the same ack.
- Server returns a **per-conversation sequence number** after durable storage. Order by that, not by client timestamps.
- Retries until ack = at-least-once (never lost); idempotent write = never duplicated.

## Still open
- Full chat design session: API and data model, architecture diagram, technology choices, deep dives (group fan-out, offline delivery, ordering, E2EE), bottlenecks at 10x

## Questions to revisit
- Why is the bucket walk O(n + k) and not O(k)?
- Why does `rightProduct` have to be used before folding in `nums[i]`?
- Why can't a timestamp-based message ID guarantee "never lost, never duplicated"?
