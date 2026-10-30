# 167. Two Sum II — Input Array Is Sorted

- **Difficulty:** Medium
- **Primary pattern:** Two Pointers
- **Supporting idea:** Sorted input
- **Status:** Solved
- **Language:** Python
- **LeetCode:** https://leetcode.com/problems/two-sum-ii-input-array-is-sorted/
- **First solved:** 2026-10-28
- **Next review:** 2026-11-04

---

## Problem in My Own Words

Given an array of integers sorted in nondecreasing order, find two different elements whose sum equals `target`.

Return their positions using **1-based indexing**. The problem guarantees exactly one solution, and the same element cannot be used twice.

### Example

```text
numbers = [2, 7, 11, 15]
target = 9

Output: [1, 2]
```

The values `2` and `7` add up to `9`. Their zero-based indices are `0` and `1`, so the required one-based answer is `[1, 2]`.

---

## Recognition Clues

- The input array is **sorted**.
- The problem asks for a **pair** of values.
- Comparing the current sum with the target tells us whether we need a larger or smaller value.
- The required result can be found without checking every pair.

These clues suggest the **opposite-direction Two Pointers** pattern.

```text
[2, 7, 11, 15]
 ↑           ↑
left       right
```

---

## Brute-Force Approach

Check every possible pair with two nested loops.

```python
def two_sum_brute_force(numbers: list[int], target: int) -> list[int]:
    for first in range(len(numbers)):
        for second in range(first + 1, len(numbers)):
            if numbers[first] + numbers[second] == target:
                return [first + 1, second + 1]

    raise ValueError("No solution found")
```

### Brute-force complexity

- **Time:** `O(n²)` because up to every pair is checked.
- **Extra space:** `O(1)` because no collection grows with the input.

### What is wasteful?

The brute-force solution ignores the sorted order. After calculating a sum, it does not use whether that sum is too small or too large to eliminate other pairs.

---

## Optimized Insight

Start with pointers at the two ends:

- `left` points to the smallest available value.
- `right` points to the largest available value.

Calculate:

```text
sum = numbers[left] + numbers[right]
```

Then make one of three decisions:

| Comparison | Meaning | Action |
|---|---|---|
| `sum == target` | The required pair was found | Return the indices |
| `sum < target` | A larger sum is needed | Move `left` right |
| `sum > target` | A smaller sum is needed | Move `right` left |

Each comparison safely removes one value from consideration.

---

## Why Each Pointer Movement Is Safe

### When the sum is too small

Suppose:

```text
numbers[left] + numbers[right] < target
```

The current `right` value is the largest available value. Pairing `numbers[left]` with any value to the left of `right` would produce an equal or smaller sum.

Therefore, `numbers[left]` cannot be part of the solution within the remaining range. We safely discard it:

```python
left += 1
```

### When the sum is too large

Suppose:

```text
numbers[left] + numbers[right] > target
```

The current `left` value is the smallest available value. Pairing `numbers[right]` with any value to the right of `left` would produce an equal or larger sum.

Therefore, `numbers[right]` cannot be part of the solution within the remaining range. We safely discard it:

```python
right -= 1
```

---

## Invariant

At the beginning of every loop iteration:

> If a valid pair has not already been found, it remains inside the inclusive range from `left` to `right`.

Moving a pointer discards only a value that cannot participate in a valid remaining pair. Therefore, the invariant continues to hold.

---

## Dry Run

### Input

```text
numbers = [1, 2, 7, 11, 12, 15, 20]
target = 19
```

```text
Index (0-based):  0  1  2   3   4   5   6
Value:            1  2  7  11  12  15  20
                  ↑                     ↑
                 left                 right
```

| Step | `left` | `right` | Values | Sum | Comparison | Action |
|---:|---:|---:|---|---:|---|---|
| 1 | 0 | 6 | `1 + 20` | 21 | Too large | Move `right` left |
| 2 | 0 | 5 | `1 + 15` | 16 | Too small | Move `left` right |
| 3 | 1 | 5 | `2 + 15` | 17 | Too small | Move `left` right |
| 4 | 2 | 5 | `7 + 15` | 22 | Too large | Move `right` left |
| 5 | 2 | 4 | `7 + 12` | 19 | Equal | Return `[3, 5]` |

The matching values are `7` and `12`. Their zero-based indices are `2` and `4`, so the required one-based result is `[3, 5]`.

---

## My Original Solution

```python
class Solution:
    def twoSum(self, numbers: list[int], target: int) -> list[int]:
        sIdx = 0
        lIdx = len(numbers) - 1

        while sIdx < lIdx:
            total = numbers[sIdx] + numbers[lIdx]

            if total == target:
                return [sIdx + 1, lIdx + 1]
            elif total > target:
                lIdx -= 1
            elif total < target:
                sIdx += 1
```

The algorithm is correct. It moves the appropriate pointer based on whether the sum is too large or too small and returns the required one-based indices.

---

## Refined Solution

```python
class Solution:
    def twoSum(self, numbers: list[int], target: int) -> list[int]:
        left = 0
        right = len(numbers) - 1

        while left < right:
            total = numbers[left] + numbers[right]

            if total == target:
                return [left + 1, right + 1]

            if total > target:
                right -= 1
            else:
                left += 1

        raise ValueError("No solution found")
```

### Refinements

- `left` and `right` are conventional names that make the pattern immediately recognizable.
- The final `elif total < target` becomes `else` because equality and the greater-than case have already been handled.
- The final `raise` makes the return behavior explicit, although LeetCode guarantees that a solution exists.

---

## Complexity Analysis

### Time: `O(n)`

Each iteration moves either `left` or `right` inward. Neither pointer moves backward, and each pointer can move at most `n - 1` times.

The total work is therefore linear, even though the algorithm uses two pointers.

### Extra space: `O(1)`

The algorithm stores only:

- `left`
- `right`
- `total`

The amount of additional memory does not grow with the input size.

---

## Why This Is Not Binary Search

Both algorithms can use variables named `left` and `right`, but their strategies differ.

### This Two Pointers solution

- Examines values at both ends.
- Moves one pointer by one position.
- Uses the pair's sum to decide which value to discard.

### Binary Search

- Examines the middle value.
- Discards half of the search range after each comparison.
- Usually searches for one target value or a boundary.

This problem could be solved by running binary search for each element's complement, but that would take `O(n log n)`. Two Pointers achieves `O(n)`.

---

## What I Learned

- A sorted array provides information after every comparison.
- If the sum is too small, moving the right pointer left would only make it smaller. The left pointer must move right.
- If the sum is too large, moving the left pointer right would only make it larger. The right pointer must move left.
- Two index variables do not automatically make an algorithm Two Pointers. The pattern depends on coordinated pointer movement that eliminates candidates.
- LeetCode 167 requires **1-based indices**.

---

## Mistakes or Confusion to Watch For

- [ ] Returning zero-based indices instead of one-based indices.
- [ ] Moving the wrong pointer when the sum is too small or too large.
- [ ] Using `left <= right`, which could allow the same element to be used twice.
- [ ] Using a Hash Map and missing the opportunity to exploit sorted input.
- [ ] Calling the solution Binary Search merely because it uses `left` and `right`.

---

## Review Questions

Before retrying the problem, answer these without reading the solution:

1. Which clues identify this as a Two Pointers problem?
2. Why do the pointers begin at opposite ends?
3. Why is it safe to move `left` when the sum is too small?
4. Why is it safe to move `right` when the sum is too large?
5. Why is the running time `O(n)` rather than `O(n²)`?
6. Why does the loop use `left < right`?
7. Why are `1` values added to the returned indices?

---

## Review History

| Date | Result | Time taken | Needed a hint? | Notes |
|---|---|---:|---|---|
| 2026-10-28 | Solved | | | Initial solution was correct |
| 2026-11-04 | | | | First scheduled retry |
| | | | | |
