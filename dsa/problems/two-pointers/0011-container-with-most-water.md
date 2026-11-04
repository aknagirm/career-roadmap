# 11. Container With Most Water

- **Difficulty:** Medium
- **Primary pattern:** Two Pointers
- **Supporting strategy:** Greedy
- **Variant:** Opposite-direction pointers
- **Status:** Solved
- **Language:** Python
- **LeetCode:** https://leetcode.com/problems/container-with-most-water/
- **First solved:** 2026-10-28
- **Next review:** 2026-11-04

---

## Problem in My Own Words

Each array value represents the height of a vertical line. Choose two lines that form a container with the x-axis and hold the maximum possible amount of water.

For lines at indices `left` and `right`:

```text
width  = right - left
height = min(height[left], height[right])
area   = width * height
```

The shorter line limits the water level.

### Example

```text
height = [1, 8, 6, 2, 5, 4, 8, 3, 7]
```

The best pair is at indices `1` and `8`:

```text
width  = 8 - 1 = 7
height = min(8, 7) = 7
area   = 7 * 7 = 49
```

---

## Recognition Clues

- The answer uses a pair of positions.
- Distance between the positions affects the result.
- The values at both positions determine the usable height.
- Starting with maximum width lets us eliminate one endpoint after each comparison.

These clues suggest **opposite-direction Two Pointers**.

```text
[2, 9, 4, 7, 3, 8, 9, 5]
 ↑                    ↑
left                right
```

---

## Brute-Force Approach

Try every pair of lines and calculate its area.

```python
class Solution:
    def maxArea(self, height: list[int]) -> int:
        max_area = 0

        for left in range(len(height)):
            for right in range(left + 1, len(height)):
                width = right - left
                container_height = min(height[left], height[right])
                max_area = max(max_area, width * container_height)

        return max_area
```

### Complexity

- **Time:** `O(n²)` because every pair is examined.
- **Extra space:** `O(1)`.

The optimized solution must avoid examining every pair.

---

## My Solution

```python
class Solution:
    def maxArea(self, height: list[int]) -> int:
        left = 0
        right = len(height) - 1

        water = 0

        while left < right:
            if height[left] > height[right]:
                water = max((right - left) * height[right], water)
                right -= 1
            else:
                water = max((right - left) * height[left], water)
                left += 1

        return water
```

This solution is correct and optimal.

In each branch, it calculates the area using the shorter height and then moves the pointer at that shorter line.

When both lines have equal heights, the `else` branch moves `left`. Moving either pointer is valid in that case.

---

## Refined Version

This version separates area calculation from pointer movement:

```python
class Solution:
    def maxArea(self, height: list[int]) -> int:
        left = 0
        right = len(height) - 1
        max_area = 0

        while left < right:
            width = right - left
            container_height = min(height[left], height[right])
            current_area = width * container_height
            max_area = max(max_area, current_area)

            if height[left] <= height[right]:
                left += 1
            else:
                right -= 1

        return max_area
```

Both versions implement the same algorithm and have the same complexity.

---

## Why LeetCode Also Labels It Greedy

The labels describe different parts of the solution:

- **Two Pointers** describes how the search is organized.
- **Greedy** describes the irreversible decision made during each iteration.

The greedy decision is:

> After evaluating the current pair, permanently discard the shorter line.

### If the left line is shorter

Suppose:

```text
height[left] <= height[right]
```

The current area is limited by `height[left]`.

Keeping `left` and moving `right` inward cannot produce a larger area because:

- The width becomes smaller.
- The usable height remains at most `height[left]`.

Therefore, no remaining pair using the current `left` line can beat the pair just evaluated. It is safe to discard that line:

```python
left += 1
```

### If the right line is shorter

The same argument applies in the opposite direction. Keeping the short right line while moving `left` inward reduces the width without increasing the limiting height.

It is safe to discard the right line:

```python
right -= 1
```

### Why the taller line is not discarded

The taller line might later pair with another tall line and create a container whose increased usable height compensates for the reduced width.

The shorter line has no such possibility: it continues to limit every container that includes it.

### Equal heights

If both endpoints have equal heights, either one can be discarded. Keeping either endpoint while reducing the width cannot improve on the current pair using that same endpoint.

---

## Detailed Dry Run

### Input

```text
height = [2, 9, 4, 7, 3, 8, 9, 5]
```

### Complete trace

| Step | `left` | `right` | Heights | Width | Area | Maximum | Move |
|---:|---:|---:|---|---:|---:|---:|---|
| 1 | 0 | 7 | `2, 5` | 7 | 14 | 14 | `left` |
| 2 | 1 | 7 | `9, 5` | 6 | 30 | 30 | `right` |
| 3 | 1 | 6 | `9, 9` | 5 | **45** | **45** | `left` |
| 4 | 2 | 6 | `4, 9` | 4 | 16 | 45 | `left` |
| 5 | 3 | 6 | `7, 9` | 3 | 21 | 45 | `left` |
| 6 | 4 | 6 | `3, 9` | 2 | 6 | 45 | `left` |
| 7 | 5 | 6 | `8, 9` | 1 | 8 | 45 | `left` |

The pointers then meet, so the loop stops.

### Result

```text
max area = 45
```

The best pair uses indices `1` and `6`:

```text
width  = 6 - 1 = 5
height = min(9, 9) = 9
area   = 5 * 9 = 45
```

---

## Edge Cases

### Minimum valid input

```text
height = [1, 1]
area = 1 * 1 = 1
```

### Equal heights

```text
height = [5, 5, 5]
```

The outside pair produces the maximum area:

```text
width = 2
area = 2 * 5 = 10
```

### Tall lines close together

```text
height = [1, 10, 10, 1]
```

The inside pair has greater height but only width `1`. The algorithm still checks it while moving inward.

### Zero-height lines

```text
height = [0, 4, 0, 4, 0]
```

Zero-height endpoints produce zero area and are discarded as shorter lines.

---

## Complexity Analysis

### Time: `O(n)`

Exactly one pointer moves during every iteration. Both pointers move only inward, so there are at most `n - 1` iterations.

### Extra space: `O(1)`

The algorithm stores only the two pointers and a few numeric values.

---

## Common Mistakes

- [ ] Using the taller height instead of `min(height[left], height[right])`.
- [ ] Using `right - left + 1` for width. The correct width is `right - left`.
- [ ] Moving the taller line instead of the shorter line.
- [ ] Moving a pointer before calculating the current area.
- [ ] Returning the pair of indices instead of the maximum area.
- [ ] Updating the answer only after moving a pointer.
- [ ] Treating the problem as only Two Pointers and missing the Greedy elimination argument.

---

## Review Questions

1. Why do the pointers begin at opposite ends?
2. Why is the usable height the shorter of the two lines?
3. Why is the width `right - left`?
4. Why can the shorter line be permanently discarded?
5. Why might discarding the taller line lose the optimal answer?
6. Why is moving either pointer valid when the heights are equal?
7. Which part of the solution is Two Pointers, and which part is Greedy?
8. Why is the optimized running time `O(n)`?

---

## Review History

| Date | Result | Time taken | Needed a hint? | Notes |
|---|---|---:|---|---|
| 2026-10-28 | Solved | | No | Correct optimal solution; clarified its Greedy classification |
| 2026-11-04 | | | | First scheduled retry |
| | | | | |
