# Day 1 — 2026-09-29

**Week 1:** Arrays / hashing, two pointers (DSA) · answer framework + estimation (design, not started yet)
**Language:** Java

## Topics covered
- Complexity basics: always state time *and* space, and what `n` is
- Hashing pattern (trade space for time)
- Two pointers: opposite ends, sorted arrays
- Sort + fix one element + two pointers (3Sum)

## Problems

### 1. Two Sum (LeetCode 1)
**Idea:** map of value → index. For each element, check whether `target - num` is already in the map *before* inserting the current one.
**Complexity:** O(n) time, O(n) space.

```java
class Solution {
    public int[] twoSum(int[] nums, int target) {
        Map<Integer, Integer> seen = new HashMap<>();
        for (int i = 0; i < nums.length; i++) {
            Integer j = seen.get(target - nums[i]);
            if (j != null) return new int[]{j, i};
            seen.put(nums[i], i);
        }
        return new int[]{};
    }
}
```
**Learning:** check before insert, so the same element is never used twice. A single `get` avoids hashing twice.

### 2. Valid Palindrome (LeetCode 125)
**Idea:** two pointers from both ends, skip non-alphanumeric characters, compare lowercase.
**Complexity:** O(n) time, O(1) space.

```java
class Solution {
    public boolean isPalindrome(String s) {
        int i = 0, j = s.length() - 1;
        while (i < j) {
            while (i < j && !Character.isLetterOrDigit(s.charAt(i))) i++;
            while (i < j && !Character.isLetterOrDigit(s.charAt(j))) j--;
            if (Character.toLowerCase(s.charAt(i)) != Character.toLowerCase(s.charAt(j))) return false;
            i++; j--;
        }
        return true;
    }
}
```
**Learning:** `isWhitespace` was redundant, since whitespace is never a letter or digit. My original second check even used index `i` instead of `j`. Skip with inner loops and keep the `i < j` guards.

### 3. 3Sum (LeetCode 15)
**Idea:** sort; fix `nums[i]`; two pointers on the rest (starting at `i + 1`) looking for `-nums[i]`; skip duplicates at every level where a choice is made.
**Complexity:** O(n²) time, O(1) extra space (ignoring output and sort internals).

```java
class Solution {
    public List<List<Integer>> threeSum(int[] nums) {
        List<List<Integer>> result = new ArrayList<>();
        if (nums.length < 3) return result;
        Arrays.sort(nums);
        if (nums[0] > 0) return result;

        for (int i = 0; i < nums.length - 2; i++) {
            if (i > 0 && nums[i] == nums[i - 1]) continue;   // dedup first number
            twoSum(result, i + 1, -nums[i], nums);
        }
        return result;
    }

    private void twoSum(List<List<Integer>> result, int start, int target, int[] nums) {
        int i = start, j = nums.length - 1;
        while (i < j) {
            int sum = nums[i] + nums[j];
            if (sum == target) {
                result.add(Arrays.asList(-target, nums[i], nums[j]));
                i++; j--;
                while (i < j && nums[i] == nums[i - 1]) i++;  // dedup second number
                while (i < j && nums[j] == nums[j + 1]) j--;
            } else if (sum < target) i++;
            else j--;
        }
    }
}
```
**Bugs I hit (and fixes):**
- Found a triplet, moved only `i` → duplicates. Fix: move both pointers, then skip equal values.
- Skipped duplicates only for the first number. `[1,2,0,1,0,0,0,0]` gave `[[0,0,0],[0,0,0]]`. Fix: also skip duplicates for the second number.

**Learning:** deduplicate at *every* level where you choose a value. The third value is then determined.

### 4. Container With Most Water (LeetCode 11)
**Idea:** pointers at both ends; area = `min(h[l], h[r]) * (r - l)`. Move the **shorter** pointer.
**Why it works:** the shorter wall caps the height, and any other container using it has a smaller width. Its best case is already measured, so discard it. The taller wall might still pair with a taller one later.
**Complexity:** O(n) time, O(1) space.

```java
class Solution {
    public int maxArea(int[] height) {
        int i = 0, j = height.length - 1, max = 0;
        while (i < j) {
            int area = Math.min(height[i], height[j]) * (j - i);
            max = Math.max(max, area);
            if (height[i] < height[j]) i++; else j--;
        }
        return max;
    }
}
```
**Learning:** my first idea (scan, restart at a bigger number) throws away lines too early. Start with maximum width, then give up width only to find a taller limiting wall.

## Questions to revisit
- Why is moving the shorter pointer safe? (Say it in my own words, without notes.)
- Why does 3Sum need dedup at two levels?

## Cheat sheet
- Hash lookup: `map.get(k)` once instead of `containsKey` + `get`
- Compare ints safely: `Integer.compare(a, b)` instead of subtraction
- Always state: time, space, and what `n` is
