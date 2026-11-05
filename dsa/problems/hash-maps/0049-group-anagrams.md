# 49. Group Anagrams

- **Difficulty:** Medium
- **Primary pattern:** Hash Map
- **Supporting technique:** Sorting
- **Status:** Solved
- **Language:** Python
- **LeetCode:** https://leetcode.com/problems/group-anagrams/
- **First solved:** 2026-10-28
- **Next review:** 2026-11-04

---

## Problem in My Own Words

Given a list of strings, group together strings that are anagrams of each other.

Anagrams contain exactly the same characters with exactly the same frequencies, but the characters may appear in a different order.

### Example

```text
Input:
["eat", "tea", "tan", "ate", "nat", "bat"]

Possible output:
[
  ["eat", "tea", "ate"],
  ["tan", "nat"],
  ["bat"]
]
```

The order of the groups and the order of strings inside each group do not matter.

---

## Recognition Clues

- Several items need to be grouped by a shared property.
- Anagrams need a common representation regardless of character order.
- A value must map to a collection of matching strings.

This suggests a Hash Map:

```text
canonical anagram key → list of original strings
```

Sorting an anagram always produces the same canonical key:

```text
"eat" → "aet"
"tea" → "aet"
"ate" → "aet"
```

---

## My Solution

```python
class Solution:
    def groupAnagrams(self, strs: list[str]) -> list[list[str]]:
        tempDict = {}

        for i, s in enumerate(strs):
            newArr = list(s)
            newArr.sort()
            newStr = "".join(newArr)

            if newStr in tempDict:
                tempDict[newStr].append(s)
            else:
                tempDict[newStr] = [s]

        return list(tempDict.values())
```

This solution is correct.

For each string, it:

1. Converts the string to a character list.
2. Sorts the characters.
3. Joins them into a canonical string key.
4. Appends the original string to the matching Hash Map group.
5. Returns all grouped values from the Hash Map.

---

## Refined Version

```python
class Solution:
    def groupAnagrams(self, strs: list[str]) -> list[list[str]]:
        groups = {}

        for word in strs:
            key = "".join(sorted(word))

            if key not in groups:
                groups[key] = []

            groups[key].append(word)

        return list(groups.values())
```

### Refinements

- `i` is removed because the index is not used.
- `word`, `key`, and `groups` describe their roles more clearly.
- `sorted(word)` returns the sorted characters directly.
- Group creation and appending are separated, so appending appears only once.

A shorter version can use `setdefault`:

```python
class Solution:
    def groupAnagrams(self, strs: list[str]) -> list[list[str]]:
        groups = {}

        for word in strs:
            key = "".join(sorted(word))
            groups.setdefault(key, []).append(word)

        return list(groups.values())
```

The explicit version is often easier to explain in an interview.

---

## Detailed Dry Run

### Input

```text
strs = ["eat", "tea", "tan", "ate", "nat", "bat"]
```

### Complete trace

| Step | Original string | Sorted key | Hash Map after insertion |
|---:|---|---|---|
| 1 | `"eat"` | `"aet"` | `{"aet": ["eat"]}` |
| 2 | `"tea"` | `"aet"` | `{"aet": ["eat", "tea"]}` |
| 3 | `"tan"` | `"ant"` | `{"aet": ["eat", "tea"], "ant": ["tan"]}` |
| 4 | `"ate"` | `"aet"` | `{"aet": ["eat", "tea", "ate"], "ant": ["tan"]}` |
| 5 | `"nat"` | `"ant"` | `{"aet": ["eat", "tea", "ate"], "ant": ["tan", "nat"]}` |
| 6 | `"bat"` | `"abt"` | `{"aet": ["eat", "tea", "ate"], "ant": ["tan", "nat"], "abt": ["bat"]}` |

### Returned value

```text
[
  ["eat", "tea", "ate"],
  ["tan", "nat"],
  ["bat"]
]
```

Each Hash Map value is one anagram group.

---

## Complexity Analysis

Let:

- `n` be the number of strings.
- `k` be the maximum string length.

### Time: `O(n × k log k)`

Sorting one string takes `O(k log k)`. Doing that for `n` strings takes:

```text
O(n × k log k)
```

Building keys and inserting them into the dictionary does not exceed the sorting cost.

### Extra space: `O(n × k)`

The Hash Map stores all original strings in groups and up to one sorted key per distinct group. Temporary sorted character lists also use space proportional to string length.

---

## Frequency-Signature Alternative

LeetCode specifies lowercase English letters. Instead of sorting, count the frequency of each of the 26 letters and use the immutable count tuple as the Hash Map key.

```python
class Solution:
    def groupAnagrams(self, strs: list[str]) -> list[list[str]]:
        groups = {}

        for word in strs:
            counts = [0] * 26

            for char in word:
                index = ord(char) - ord("a")
                counts[index] += 1

            key = tuple(counts)

            if key not in groups:
                groups[key] = []

            groups[key].append(word)

        return list(groups.values())
```

### Complexity of frequency signatures

- **Time:** `O(n × k)`
- **Extra space:** `O(n × k)` for the output and groups; each key has a fixed 26 counts.

The frequency approach is asymptotically faster, but the sorting-key solution is shorter, generalizes naturally to more character sets, and is often the clearest interview solution unless optimization is requested.

### Why the key must be a tuple

Python dictionary keys must be hashable. A list can change and is not hashable:

```python
counts = [0] * 26       # cannot be a dictionary key
key = tuple(counts)     # immutable and hashable
```

---

## Edge Cases

### One string

```text
["abc"] → [["abc"]]
```

### Empty string

```text
[""] → [[""]]
```

The sorted key for an empty string is also `""`.

### Multiple empty strings

```text
["", ""] → [["", ""]]
```

Both use the same empty key.

### No anagrams

```text
["a", "b", "c"] → [["a"], ["b"], ["c"]]
```

### All strings are anagrams

```text
["abc", "bca", "cab"] → [["abc", "bca", "cab"]]
```

### Duplicate strings

```text
["ab", "ab"] → [["ab", "ab"]]
```

Both copies must remain in the result.

---

## Common Mistakes

- [ ] Using the original string as the key instead of a canonical representation.
- [ ] Sorting the input list rather than sorting the characters of each string.
- [ ] Returning the entire dictionary instead of `list(groups.values())`.
- [ ] Forgetting to initialize a list for a new key.
- [ ] Appending the sorted key instead of the original string.
- [ ] Using a mutable frequency list as a dictionary key instead of converting it to a tuple.
- [ ] Claiming the sorting solution takes `O(n × k)` rather than `O(n × k log k)`.
- [ ] Assuming the output groups must appear in one specific order.

---

## Review Questions

1. What property should be used as the Hash Map key?
2. Why do all anagrams produce the same sorted key?
3. What does each Hash Map value contain?
4. Why is the sorting solution `O(n × k log k)`?
5. How does the frequency-signature version avoid sorting?
6. Why must a frequency list be converted to a tuple before using it as a key?
7. Which approach would you choose when clarity matters most?
8. Which approach would you choose when strings are long and restricted to lowercase English letters?

---

## Review History

| Date | Result | Time taken | Needed a hint? | Notes |
|---|---|---:|---|---|
| 2026-10-28 | Solved | | No | Used a sorted string as the Hash Map grouping key |
| 2026-11-04 | | | | First scheduled retry |
| | | | | |
