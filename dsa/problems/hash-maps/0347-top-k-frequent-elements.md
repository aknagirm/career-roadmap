# 347. Top K Frequent Elements

- **Difficulty:** Medium
- **Primary pattern:** Hash Map
- **Supporting technique:** Bucket Sort
- **Status:** Solved
- **Language:** Python
- **LeetCode:** https://leetcode.com/problems/top-k-frequent-elements/
- **First solved:** 2026-10-28
- **Next review:** 2026-11-04

---

## Problem in My Own Words

Given an integer array `nums` and an integer `k`, return the `k` values that occur most frequently.

LeetCode guarantees that the answer set is unique. The answer may be returned in any order.

### Example

```text
nums = [1, 1, 1, 2, 2, 3]
k = 2

Frequencies:
1 → 3
2 → 2
3 → 1

Result: [1, 2]
```

The solution must be faster than `O(n log n)`.

---

## Recognition Clues

- We need to count how often each number occurs.
- We need the values with the highest counts.
- A frequency is always an integer between `1` and `len(nums)`.
- The required complexity rules out sorting every distinct value by frequency.

This suggests combining:

```text
Hash Map:   number → frequency
Bucket list: frequency → numbers with that frequency
```

---

## First Sorting Attempt

```python
result = sorted(temp)[:k]
```

Sorting a dictionary directly sorts its keys, not its values.

For:

```python
temp = {
    10: 3,
    5: 2,
    7: 1,
}
```

this produces:

```python
sorted(temp) == [5, 7, 10]
```

That is numeric key order, not frequency order.

A corrected sorting version would be:

```python
result = sorted(
    temp,
    key=lambda number: temp[number],
    reverse=True,
)

return result[:k]
```

However, sorting `m` distinct values takes `O(m log m)`. Bucket Sort avoids that sorting step and satisfies the requested linear-time target.

---

## My Optimal Solution

```python
class Solution:
    def topKFrequent(self, nums: list[int], k: int) -> list[int]:
        temp = {}

        for n in nums:
            if n in temp:
                temp[n] += 1
            else:
                temp[n] = 1

        bucket = [[] for _ in range(len(nums) + 1)]

        for number, freq in temp.items():
            bucket[freq].append(number)

        result = []

        for i in range(len(bucket) - 1, -1, -1):
            if len(bucket[i]) > 0:
                for n in bucket[i]:
                    result.append(n)

                    if len(result) == k:
                        return result
```

This solution is correct and optimal.

---

## Refined Version

```python
class Solution:
    def topKFrequent(self, nums: list[int], k: int) -> list[int]:
        frequency = {}

        for number in nums:
            if number in frequency:
                frequency[number] += 1
            else:
                frequency[number] = 1

        buckets = [[] for _ in range(len(nums) + 1)]

        for number, count in frequency.items():
            buckets[count].append(number)

        result = []

        for count in range(len(buckets) - 1, 0, -1):
            for number in buckets[count]:
                result.append(number)

                if len(result) == k:
                    return result
```

### Refinements

- `frequency` and `buckets` describe the data more clearly than `temp` and `bucket`.
- The explicit empty-bucket check is removed because iterating over an empty list naturally performs zero iterations.
- The reverse range stops at frequency `1` because bucket `0` is never populated.
- Area outside the essential logic is reduced without changing the algorithm.

---

## Python Syntax Used

### Creating independent empty buckets

```python
buckets = [[] for _ in range(len(nums) + 1)]
```

This creates one separate empty list for each possible frequency.

The expanded version is:

```python
buckets = []

for index in range(len(nums) + 1):
    buckets.append([])
```

The underscore `_` conventionally means that the loop value is not needed.

Do not use:

```python
buckets = [[]] * (len(nums) + 1)
```

That creates several references to the same inner list. Appending to one bucket would appear in every bucket.

### Reading dictionary key-value pairs

```python
for number, count in frequency.items():
```

For:

```python
frequency = {1: 3, 2: 2, 3: 1}
```

`.items()` produces these pairs:

```text
number = 1, count = 3
number = 2, count = 2
number = 3, count = 1
```

### Reverse range

```python
range(len(buckets) - 1, 0, -1)
```

The arguments mean:

```text
start: final bucket index
stop:  0, excluded
step:  -1
```

For seven buckets, it generates:

```text
6, 5, 4, 3, 2, 1
```

---

## Detailed Dry Run

### Input

```text
nums = [5, 5, 5, 5, -2, -2, -2, 9, 9, 1, 7, 7, 7, 7, 7]
k = 3
```

### Step 1: Count frequencies

```python
frequency = {
    5: 4,
    -2: 3,
    9: 2,
    1: 1,
    7: 5,
}
```

### Step 2: Create buckets

There are 15 input values, so 16 buckets are created for indices `0` through `15`.

Only these buckets receive values:

```text
buckets[1] = [1]
buckets[2] = [9]
buckets[3] = [-2]
buckets[4] = [5]
buckets[5] = [7]
```

All other buckets remain empty.

### Step 3: Scan backward

| Bucket/frequency | Contents | Action | Result |
|---:|---|---|---|
| 15 down to 6 | Empty | Skip naturally | `[]` |
| 5 | `[7]` | Append `7` | `[7]` |
| 4 | `[5]` | Append `5` | `[7, 5]` |
| 3 | `[-2]` | Append `-2`; length is now `k` | `[7, 5, -2]` |

The function returns immediately:

```python
[7, 5, -2]
```

---

## Why the Nested Loops Are Still O(n)

The reverse scan contains nested loops:

```python
for count in range(len(buckets) - 1, 0, -1):
    for number in buckets[count]:
        ...
```

This is not `O(n²)` because the inner loop does not process `n` values for every bucket.

Each distinct number appears in exactly one bucket. If there are `m` distinct numbers:

```text
len(buckets[0])
+ len(buckets[1])
+ ...
+ len(buckets[n])
= m
```

Therefore:

- The outer loop checks at most `n` bucket positions.
- All inner-loop executions combined process at most `m` numbers.
- Since `m <= n`, the scan is `O(n + m) = O(n)`.

Nested loops are quadratic only when the inner loop performs linear work for every outer iteration.

---

## Tie at the Kth Boundary

Consider:

```text
nums = [5, 5, 5, 5, -2, -2, -2, 9, 9, 1, 1, 7, 7, 7, 7, 7]
k = 4
```

Frequencies are:

```text
7  → 5
5  → 4
-2 → 3
9  → 2
1  → 2
```

The fourth result could be either `9` or `1`. That produces two valid answer sets, so this input violates LeetCode's guarantee that the answer is unique.

The Bucket Sort code can still return one of those valid custom-test results. LeetCode's official tests avoid ambiguity at the `k`th cutoff.

Ties are allowed when the cutoff does not split the tied group.

---

## Edge Cases

### One distinct value

```text
nums = [4, 4, 4]
k = 1
result = [4]
```

### Every value appears once

```text
nums = [3, 1, 2]
k = 3
```

Every value occupies bucket `1`, and all three are returned. Their order may vary.

### Negative values

```text
nums = [-1, -1, -2]
k = 1
result = [-1]
```

Negative dictionary keys require no special handling.

### `k` equals the number of distinct values

Every distinct number is returned.

---

## Complexity Analysis

Let `n` be the input length and `m` the number of distinct values.

### Time: `O(n)`

- Count frequencies: `O(n)`
- Create `n + 1` buckets: `O(n)`
- Place distinct values into buckets: `O(m)`
- Scan buckets and their combined contents: `O(n + m)`

Since `m <= n`, the total simplifies to `O(n)`.

### Extra space: `O(n)`

The frequency dictionary, bucket list, and result collectively use space proportional to the input size.

---

## Common Mistakes

- [ ] Sorting the dictionary directly and therefore sorting numeric keys.
- [ ] Returning the `k` numerically largest values instead of the `k` most frequent values.
- [ ] Creating only `len(nums)` buckets and forgetting that frequency `len(nums)` is possible.
- [ ] Using `[[]] * size` and accidentally sharing one inner list across all buckets.
- [ ] Scanning buckets from low frequency to high frequency.
- [ ] Waiting until the end to stop instead of returning after collecting `k` values.
- [ ] Assuming the nested loops automatically make the algorithm `O(n²)`.
- [ ] Requiring one specific output order when tied values are all valid.

---

## Review Questions

1. What does the first Hash Map map from and to?
2. Why are `len(nums) + 1` buckets needed?
3. What does a bucket index represent?
4. Why does each distinct number appear in exactly one bucket?
5. Why are the nested loops collectively `O(n)` rather than `O(n²)`?
6. Why does the bucket scan proceed backward?
7. Why is the empty-bucket condition unnecessary?
8. What is wrong with `[[]] * bucket_count`?
9. When can ties make a custom test violate LeetCode's unique-answer guarantee?
10. How does Bucket Sort improve on sorting by frequency?

---

## Review History

| Date | Result | Time taken | Needed a hint? | Notes |
|---|---|---:|---|---|
| 2026-10-28 | Solved | | Yes | Frequency map was correct; learned Bucket Sort to meet the linear-time requirement |
| 2026-11-04 | | | | First scheduled retry |
| | | | | |
