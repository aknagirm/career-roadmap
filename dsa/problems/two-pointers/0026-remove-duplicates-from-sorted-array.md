# 26. Remove Duplicates from Sorted Array

- **Difficulty:** Easy
- **Primary pattern:** Two Pointers
- **Variant:** Same-direction read/write pointers
- **Supporting idea:** Sorted input
- **Status:** Solved
- **Language:** Python
- **LeetCode:** https://leetcode.com/problems/remove-duplicates-from-sorted-array/
- **First solved:** 2026-10-28
- **Next review:** 2026-11-04

---

## Problem in My Own Words

Given an integer array `nums` sorted in nondecreasing order, remove duplicate values **in place** so that each unique value appears once at the beginning of the array.

Return `k`, the number of unique values.

After the function finishes:

```text
nums[0:k]
```

must contain the unique values in their original sorted order. Values after index `k - 1` do not matter.

### Example

```text
Input:  nums = [0, 0, 1, 1, 2]
Output: k = 3

Relevant part of nums: [0, 1, 2]
```

---

## Recognition Clues

- The input is **sorted**, so equal values are adjacent.
- The result must be written into the same array.
- Only a prefix of the final array must be valid.
- One pointer can inspect values while another builds the valid prefix.

These clues suggest the **same-direction Two Pointers** pattern.

```text
write: last position in the unique prefix
read:  candidate currently being inspected
```

---

## First Approach: Remove Duplicates with `pop()`

```python
class Solution:
    def removeDuplicates(self, nums: list[int]) -> int:
        idx1 = 1

        while idx1 < len(nums):
            if nums[idx1 - 1] == nums[idx1]:
                nums.pop(idx1)
            else:
                idx1 += 1

        return len(nums)
```

### Why this is logically correct

When a duplicate is removed, the following value shifts into the current index. The pointer correctly stays in place so that the shifted value is checked next.

### Why this is inefficient

Removing from the middle of a Python list requires all later elements to shift left.

For an input containing many duplicates:

```text
[1, 1, 1, 1, 1]
```

repeated calls to `pop(1)` perform approximately:

```text
(n - 1) + (n - 2) + ... + 1
= n(n - 1) / 2
= O(n²)
```

### Complexity of the `pop()` approach

- **Time:** `O(n²)` in the worst case
- **Extra space:** `O(1)`

### Lesson

A single visible loop does not guarantee `O(n)`. The complexity of operations inside the loop must also be counted.

---

## Optimized Insight

Do not physically remove elements. Build the unique result at the beginning of the existing array by overwriting positions that are no longer needed.

Use two pointers:

- `write` is the index of the **last unique value** stored so far.
- `read` scans the rest of the array looking for the next unique value.

Initial state for a nonempty array:

```python
write = 0
read = 1
```

At each step:

1. Compare `nums[read]` with `nums[write]`.
2. If they are equal, `nums[read]` is a duplicate, so skip it.
3. If they differ, `nums[read]` is a new unique value.
4. Store it at `write + 1` and advance `write`.

---

## My Two Pointers Solution

```python
class Solution:
    def removeDuplicates(self, nums: list[int]) -> int:
        write = 0
        read = 1

        while read < len(nums):
            if nums[write] != nums[read]:
                nums[write + 1], nums[read] = nums[read], nums[write + 1]
                write += 1

            read += 1

        return write + 1
```

This solution is correct for LeetCode 26.

### Why the swap works

When a new unique value is found, the swap places it directly after the current unique prefix. The value moved to `read` is irrelevant because `read` advances and that position will not be examined again.

However, preserving the overwritten value is unnecessary because only the first `k` positions matter.

---

## Refined Solution

```python
class Solution:
    def removeDuplicates(self, nums: list[int]) -> int:
        write = 0
        read = 1

        while read < len(nums):
            if nums[write] != nums[read]:
                nums[write + 1] = nums[read]
                write += 1

            read += 1

        return write + 1
```

### Why assignment is better than swapping

This:

```python
nums[write + 1] = nums[read]
```

states the actual goal: place the new unique value after the unique prefix.

A swap does extra conceptual work by preserving a value that the problem does not require.

---

## Detailed Dry Run

### Input

```text
nums = [0, 1, 1, 2, 2, 2, 3, 3, 3, 3, 4]
```

### Initial state

```text
Index:  0  1  2  3  4  5  6  7  8  9  10
Value: [0, 1, 1, 2, 2, 2, 3, 3, 3, 3, 4]
         ↑  ↑
         W  R

write = 0
read  = 1
unique prefix = [0]
```

### Complete trace

| Step | `read` | `write` before | Compare | Decision | `write` after | Unique prefix |
|---:|---:|---:|---|---|---:|---|
| 1 | 1 | 0 | `0 != 1` | Write `1` at index 1 | 1 | `[0, 1]` |
| 2 | 2 | 1 | `1 == 1` | Duplicate; skip | 1 | `[0, 1]` |
| 3 | 3 | 1 | `1 != 2` | Write `2` at index 2 | 2 | `[0, 1, 2]` |
| 4 | 4 | 2 | `2 == 2` | Duplicate; skip | 2 | `[0, 1, 2]` |
| 5 | 5 | 2 | `2 == 2` | Duplicate; skip | 2 | `[0, 1, 2]` |
| 6 | 6 | 2 | `2 != 3` | Write `3` at index 3 | 3 | `[0, 1, 2, 3]` |
| 7 | 7 | 3 | `3 == 3` | Duplicate; skip | 3 | `[0, 1, 2, 3]` |
| 8 | 8 | 3 | `3 == 3` | Duplicate; skip | 3 | `[0, 1, 2, 3]` |
| 9 | 9 | 3 | `3 == 3` | Duplicate; skip | 3 | `[0, 1, 2, 3]` |
| 10 | 10 | 3 | `3 != 4` | Write `4` at index 4 | 4 | `[0, 1, 2, 3, 4]` |

### Important array changes

#### Step 1: discover `1`

`write + 1` and `read` are both index `1`, so assigning the value changes nothing visibly:

```text
[0, 1, 1, 2, 2, 2, 3, 3, 3, 3, 4]
 └──┘
 unique prefix
```

#### Step 3: discover `2`

Copy `nums[3]` to `nums[2]`:

```text
Before: [0, 1, 1, 2, 2, 2, 3, 3, 3, 3, 4]
After:  [0, 1, 2, 2, 2, 2, 3, 3, 3, 3, 4]
         └─────┘
       unique prefix
```

#### Step 6: discover `3`

Copy `nums[6]` to `nums[3]`:

```text
Before: [0, 1, 2, 2, 2, 2, 3, 3, 3, 3, 4]
After:  [0, 1, 2, 3, 2, 2, 3, 3, 3, 3, 4]
         └────────┘
        unique prefix
```

#### Step 10: discover `4`

Copy `nums[10]` to `nums[4]`:

```text
Before: [0, 1, 2, 3, 2, 2, 3, 3, 3, 3, 4]
After:  [0, 1, 2, 3, 4, 2, 3, 3, 3, 3, 4]
         └───────────┘
          unique prefix
```

### Final state

```text
write = 4
k = write + 1 = 5
```

The function returns:

```python
5
```

The final array may look like:

```text
[0, 1, 2, 3, 4, 2, 3, 3, 3, 3, 4]
 └─────────────┘
  first k values
```

Only this portion matters:

```python
nums[:5] == [0, 1, 2, 3, 4]
```

Values after the first `k` positions are intentionally left unspecified.

---

## Edge Cases

### One element

```text
nums = [7]
```

The loop does not run. `write` remains `0`, so the function returns `1`.

### All values are the same

```text
nums = [2, 2, 2, 2]
```

Every value after the first is skipped. The function returns `1`, and the valid prefix is `[2]`.

### No duplicates

```text
nums = [1, 2, 3, 4]
```

Every candidate is new. Each assignment writes a value to the position it already occupies. The function returns `4`.

### Empty input outside LeetCode's constraints

LeetCode guarantees at least one element. A general-purpose version could handle an empty list explicitly:

```python
if not nums:
    return 0
```

---

## Complexity Analysis

### Time: `O(n)`

`read` moves from index `1` to the final index exactly once. Every iteration performs constant-time comparison, assignment, and pointer updates.

### Extra space: `O(1)`

The algorithm uses only the two integer pointers. It does not allocate another collection whose size depends on the input.

---

## How This Differs from Move Zeroes

Both problems use same-direction read/write pointers, but the pointer meanings differ slightly.

### Remove Duplicates

- `write` identifies the **last valid unique value**.
- A candidate is accepted when it differs from `nums[write]`.
- The answer is `write + 1`.

### Move Zeroes

- `write` identifies the **next position for a nonzero value**.
- A candidate is accepted when it is nonzero.
- The function modifies the entire array and returns nothing.

The shared high-level pattern is:

```text
read scans every candidate
write builds the valid prefix
```

---

## Mistakes or Confusion to Watch For

- [ ] Returning `write` instead of `write + 1`.
- [ ] Starting `read` at `0` and comparing the first element with itself unnecessarily.
- [ ] Calling `pop()` repeatedly and creating `O(n²)` behavior.
- [ ] Comparing against `nums[read - 1]` after overwriting values without checking the invariant.
- [ ] Believing the unused suffix must be cleaned up.
- [ ] Swapping when simple assignment communicates the intention more clearly.
- [ ] Forgetting that the algorithm depends on sorted input.

---

## Review Questions

Answer these without looking at the implementation:

1. What does `write` represent?
2. What does `read` represent?
3. What invariant does the unique prefix maintain?
4. Why can duplicates be detected by comparing with the last unique value?
5. Why is assignment sufficient instead of swapping?
6. Why do values after index `k - 1` not matter?
7. Why does the function return `write + 1`?
8. Why is the `pop()` approach `O(n²)` in the worst case?
9. What changes would be required if empty input were allowed?
10. How are the pointer responsibilities different in Move Zeroes?

---

## Review History

| Date | Result | Time taken | Needed a hint? | Notes |
|---|---|---:|---|---|
| 2026-10-28 | Solved | | Yes | First used `pop()`; later derived the read/write solution independently |
| 2026-11-04 | | | | First scheduled retry |
| | | | | |
