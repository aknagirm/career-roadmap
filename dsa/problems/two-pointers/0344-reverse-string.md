# 344. Reverse String

- **Difficulty:** Easy
- **Primary pattern:** Two Pointers
- **Variant:** Opposite-direction pointers
- **Status:** Solved
- **Language:** Python
- **LeetCode:** https://leetcode.com/problems/reverse-string/
- **First solved:** 2026-10-28
- **Next review:** 2026-11-04

---

## Problem in My Own Words

Reverse a list of characters in place.

The function must modify the original list and return nothing. It must use `O(1)` extra space.

### Example

```text
Input:  s = ["h", "e", "l", "l", "o"]
Output:     ["o", "l", "l", "e", "h"]
```

---

## Recognition Clues

- Values at opposite ends need to exchange positions.
- The input must be modified in place.
- The operation is symmetrical around the center.
- No additional list should be created.

These clues suggest **opposite-direction Two Pointers**.

```text
["h", "e", "l", "l", "o"]
   ↑                   ↑
 left                right
```

---

## My Solution

```python
class Solution:
    def reverseString(self, s: list[str]) -> None:
        """
        Do not return anything, modify s in-place instead.
        """
        left = 0
        right = len(s) - 1

        while left < right:
            s[left], s[right] = s[right], s[left]
            left += 1
            right -= 1
```

The solution is correct and directly implements the intended pattern.

---

## Approach

1. Place `left` at the beginning of the list.
2. Place `right` at the end of the list.
3. Swap the characters at `left` and `right`.
4. Move both pointers one position toward the center.
5. Stop when the pointers meet or cross.

### Pointer responsibilities

| Pointer | Responsibility |
|---|---|
| `left` | Identifies the next character from the beginning |
| `right` | Identifies its matching character from the end |

### Why `left < right` is used

- When `left == right`, both pointers identify the middle character. It is already in its final position.
- When `left > right`, every symmetrical pair has been processed.
- Using `<` avoids an unnecessary self-swap for an odd-length list.

---

## Detailed Dry Run

### Input

```text
s = ["h", "e", "l", "l", "o"]
```

### Initial state

```text
Index:  0    1    2    3    4
Value: [h,   e,   l,   l,   o]
        ↑                   ↑
       left               right
```

### Complete trace

| Step | `left` | `right` | Swap | Array afterward |
|---:|---:|---:|---|---|
| 1 | 0 | 4 | `h ↔ o` | `["o", "e", "l", "l", "h"]` |
| 2 | 1 | 3 | `e ↔ l` | `["o", "l", "l", "e", "h"]` |

After step 2:

```text
left = 2
right = 2
```

Both pointers identify the middle character. The condition `left < right` is false, so the loop ends.

### Final array

```text
["o", "l", "l", "e", "h"]
```

The function does not return this list. It modifies the original `s` list directly.

---

## Edge Cases

### Empty list

```text
[] → []
```

`right` becomes `-1`, so the loop does not run.

### One character

```text
["a"] → ["a"]
```

Both pointers start at index `0`, so the loop does not run.

### Two characters

```text
["a", "b"] → ["b", "a"]
```

The algorithm performs exactly one swap.

### Even number of characters

```text
["a", "b", "c", "d"] → ["d", "c", "b", "a"]
```

The pointers cross after every pair has been swapped.

---

## Complexity Analysis

### Time: `O(n)`

Each swap places two characters in their final positions. The loop runs approximately `n / 2` times, which simplifies to `O(n)`.

### Extra space: `O(1)`

The algorithm uses only two indices. Python's tuple swap uses constant additional space and does not create another list proportional to the input.

---

## Common Mistakes

- [ ] Returning a new reversed list instead of modifying `s`.
- [ ] Using slicing such as `s[::-1]`, which creates another list.
- [ ] Forgetting to move one of the pointers.
- [ ] Using `left <= right`, which works but performs an unnecessary middle self-swap.
- [ ] Returning `s` even though the method must return `None`.
- [ ] Reversing the order incorrectly by moving both pointers in the same direction.

---

## Review Questions

1. Why do the pointers start at opposite ends?
2. Why does one swap place two characters correctly?
3. Why does the loop stop at `left >= right`?
4. Why is the running time `O(n)` even though the loop runs about `n / 2` times?
5. Why does slicing fail the `O(1)` extra-space requirement?
6. What happens for an empty list or a one-character list?

---

## Review History

| Date | Result | Time taken | Needed a hint? | Notes |
|---|---|---:|---|---|
| 2026-10-28 | Solved | | No | Correct opposite-direction Two Pointers solution |
| 2026-11-04 | | | | First scheduled retry |
| | | | | |
