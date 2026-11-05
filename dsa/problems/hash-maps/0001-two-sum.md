# 1. Two Sum

- **Difficulty:** Easy
- **Primary pattern:** Hash Map
- **Status:** Solved
- **Language:** Python
- **LeetCode:** https://leetcode.com/problems/two-sum/
- **First solved:** 2026-10-28
- **Next review:** 2026-11-04

---

## Problem in My Own Words

Given an unsorted integer array `nums` and an integer `target`, find two different elements whose sum equals `target`.

Return their zero-based indices. The problem guarantees exactly one valid answer, and the same element cannot be used twice.

### Test case

```text
nums = [3, 8, -4, 6, 12, 3]
target = 6
```

Expected indices:

```text
[0, 5]
```

The two values are both `3`, but they occur at different indices:

```text
nums[0] + nums[5] = 3 + 3 = 6
```

---

## Recognition Clues

- The input is not sorted.
- The problem asks whether a matching value has already been encountered.
- For each number, the required partner can be calculated directly.
- Original indices must be preserved.
- A fast value-to-index lookup is needed.

These clues suggest a **Hash Map**.

```text
current number + complement = target
complement = target - current number
```

---

## Brute-Force Approach

Try every pair of indices.

```python
class Solution:
    def twoSum(self, nums: list[int], target: int) -> list[int]:
        for first in range(len(nums)):
            for second in range(first + 1, len(nums)):
                if nums[first] + nums[second] == target:
                    return [first, second]

        raise ValueError("No solution found")
```

### Complexity

- **Time:** `O(n²)` because up to every pair is checked.
- **Extra space:** `O(1)`.

The repeated search for a matching value is the part a Hash Map removes.

---

## My Solution

```python
class Solution:
    def twoSum(self, nums: list[int], target: int) -> list[int]:
        temp = {}

        for i, n in enumerate(nums):
            remaining_number = target - n

            if remaining_number in temp:
                return [i, temp[remaining_number]]
            else:
                temp[n] = i
```

This is the intended one-pass Hash Map solution.

The dictionary stores:

```text
number → index where that number was seen
```

The lookup occurs before insertion, which prevents the current element from matching with itself.

The returned indices may appear as `[5, 0]`. LeetCode permits either order.

---

## Refined Version

```python
class Solution:
    def twoSum(self, nums: list[int], target: int) -> list[int]:
        seen = {}

        for index, number in enumerate(nums):
            complement = target - number

            if complement in seen:
                return [seen[complement], index]

            seen[number] = index

        raise ValueError("No solution found")
```

### Refinements

- `seen` describes what the dictionary contains more clearly than `temp`.
- `complement` is the conventional term for the required partner value.
- Returning `[seen[complement], index]` places the earlier index first.
- The `else` is unnecessary because a successful branch returns immediately.
- The final `raise` makes the general function complete, although LeetCode guarantees a solution.

---

## Detailed Dry Run

### Input

```text
nums = [3, 8, -4, 6, 12, 3]
target = 6
```

### Complete trace

| Step | Index | Number | Complement | `seen` before | Decision |
|---:|---:|---:|---:|---|---|
| 1 | 0 | 3 | 3 | `{}` | Not found; store `3: 0` |
| 2 | 1 | 8 | -2 | `{3: 0}` | Not found; store `8: 1` |
| 3 | 2 | -4 | 10 | `{3: 0, 8: 1}` | Not found; store `-4: 2` |
| 4 | 3 | 6 | 0 | `{3: 0, 8: 1, -4: 2}` | Not found; store `6: 3` |
| 5 | 4 | 12 | -6 | `{3: 0, 8: 1, -4: 2, 6: 3}` | Not found; store `12: 4` |
| 6 | 5 | 3 | 3 | `{3: 0, 8: 1, -4: 2, 6: 3, 12: 4}` | Found at index `0`; return `[0, 5]` |

### Result

```text
[0, 5]
```

The first `3` was stored before the second `3` was examined. Therefore, two distinct indices are returned.

---

## Why Check Before Inserting?

Consider:

```text
nums = [3, 3]
target = 6
```

At index `0`, the required complement is also `3`. If the current value were inserted before checking, the algorithm could incorrectly match index `0` with itself.

The correct order is:

```text
1. Calculate complement
2. Search previously seen values
3. Return if found
4. Otherwise store the current value
```

This guarantees that every dictionary entry comes from an earlier index.

---

## Why Two Pointers Is Not the Best Choice Here

For pair-sum problems, opposite-direction Two Pointers relies on sorted order.

This input is unsorted, so a sum being too high or too low does not tell us which pointer to move.

Sorting is possible, but preserving original indices requires storing `(value, original_index)` pairs:

- Sorting takes `O(n log n)` time.
- Two Pointers then takes `O(n)` time.
- The Hash Map approach takes `O(n)` average time directly.

```text
Unsorted Two Sum → Hash Map
Sorted Two Sum   → Two Pointers
```

Sorting is not required for every Two Pointers problem. It is required here because the pointer movement would depend on whether the sum is too small or too large.

---

## Edge Cases

### Duplicate values form the answer

```text
nums = [3, 3]
target = 6
answer = [0, 1]
```

Checking before insertion handles this correctly.

### Negative numbers

```text
nums = [-5, -2, -9, -1]
target = -11
answer = [1, 2]
```

Dictionary lookup works the same way for negative values.

### Zero values

```text
nums = [0, 4, 3, 0]
target = 0
answer = [0, 3]
```

The second zero finds the first zero in the dictionary.

### Answer appears early

```text
nums = [2, 7, 11, 15]
target = 9
answer = [0, 1]
```

The function returns immediately after finding the pair.

---

## Complexity Analysis

### Time: `O(n)` average

The loop visits each number once. Python dictionary lookup and insertion take average `O(1)` time.

### Extra space: `O(n)`

If the matching pair occurs near the end, the dictionary may contain almost every input value.

---

## Common Mistakes

- [ ] Searching after inserting and accidentally using one element twice.
- [ ] Storing only values when the problem requires indices.
- [ ] Returning the values instead of their indices.
- [ ] Using two nested loops and producing `O(n²)` time.
- [ ] Sorting without preserving original indices.
- [ ] Returning one-based indices; this problem requires zero-based indices.
- [ ] Expecting a specific index order even though LeetCode allows either order.

---

## Review Questions

1. What does the dictionary map from and to?
2. How is the complement calculated?
3. Why must lookup happen before insertion?
4. How does the algorithm handle duplicate values such as `[3, 3]`?
5. Why is the average running time `O(n)`?
6. Why does the solution require `O(n)` extra space?
7. Why is Two Pointers not directly applicable to the unsorted input?
8. What additional work would be needed to sort while preserving indices?

---

## Review History

| Date | Result | Time taken | Needed a hint? | Notes |
|---|---|---:|---|---|
| 2026-10-28 | Solved | | No | Recognized why unsorted input favors a Hash Map |
| 2026-11-04 | | | | First scheduled retry |
| | | | | |
