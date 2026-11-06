# 20. Valid Parentheses

- **Difficulty:** Easy
- **Primary pattern:** Stack
- **Status:** Solved
- **Language:** Python
- **LeetCode:** https://leetcode.com/problems/valid-parentheses/
- **First solved:** 2026-10-28
- **Next review:** 2026-11-04

---

## Problem in My Own Words

Given a string containing only these bracket characters:

```text
( ) { } [ ]
```

return `True` when every opening bracket:

- Has the correct closing bracket
- Closes in the correct order

Otherwise, return `False`.

### Examples

```text
"()[]{}" → True
"(]"     → False
"([)]"   → False
"{[]}"   → True
```

---

## Recognition Clues

- Closing symbols must match opening symbols.
- The most recently opened unmatched bracket must close first.
- Processing follows **Last In, First Out** order.

These clues indicate a **Stack**.

```text
Opening bracket → push onto stack
Closing bracket → compare with stack top, then pop
```

---

## My Solution

```python
class Solution:
    def isValid(self, s: str) -> bool:
        stack = []
        temp_map = {"(": ")", "{": "}", "[": "]"}

        for i, n in enumerate(s):
            if n == "(" or n == "{" or n == "[":
                stack.append(n)
            else:
                if len(stack) == 0:
                    return False
                else:
                    if n == temp_map[stack[len(stack) - 1]]:
                        stack.pop()
                    else:
                        return False

        return not stack
```

This solution is correct.

The map stores:

```text
opening bracket → expected closing bracket
```

Before reading the stack top, the code checks whether the stack is empty. This avoids an invalid index access and correctly rejects a closing bracket with no matching opening bracket.

The final line:

```python
return not stack
```

returns `True` only when no unmatched opening brackets remain.

---

## Refined Version

```python
class Solution:
    def isValid(self, s: str) -> bool:
        stack = []
        pairs = {
            "(": ")",
            "{": "}",
            "[": "]",
        }

        for bracket in s:
            if bracket in pairs:
                stack.append(bracket)
            else:
                if not stack:
                    return False

                opening = stack[-1]

                if pairs[opening] != bracket:
                    return False

                stack.pop()

        return not stack
```

### Refinements

- The unused index from `enumerate()` is removed.
- `bracket`, `pairs`, and `opening` describe their values more clearly.
- `bracket in pairs` replaces three separate opening-bracket comparisons.
- `not stack` replaces `len(stack) == 0`.
- `stack[-1]` accesses the final item directly.
- The extra `else` nesting is removed where an earlier branch already returns.

---

## Detailed Dry Run: Valid Input

### Input

```text
s = "{[()]}"
```

| Step | Character | Action | Stack afterward |
|---:|---|---|---|
| 1 | `{` | Opening bracket; push | `["{"]` |
| 2 | `[` | Opening bracket; push | `["{", "["]` |
| 3 | `(` | Opening bracket; push | `["{", "[", "("]` |
| 4 | `)` | Matches `(`; pop | `["{", "["]` |
| 5 | `]` | Matches `[`; pop | `["{"]` |
| 6 | `}` | Matches `{`; pop | `[]` |

The stack is empty, so:

```python
return not stack  # True
```

---

## Detailed Dry Run: Invalid Order

### Input

```text
s = "([)]"
```

| Step | Character | Action | Stack afterward |
|---:|---|---|---|
| 1 | `(` | Push | `["("]` |
| 2 | `[` | Push | `["(", "["]` |
| 3 | `)` | Top `[` expects `]`; mismatch | Return `False` |

Although the string contains matching quantities of each bracket type, their nesting order is invalid.

---

## Detailed Dry Run: Closing Bracket First

### Input

```text
s = "]"
```

The first character is a closing bracket while the stack is empty:

```python
if not stack:
    return False
```

This check must happen before accessing `stack[-1]`.

---

## Why the Final Empty-Stack Check Matters

Consider:

```text
s = "(("
```

Both characters are pushed, and no mismatch occurs during the loop:

```text
stack = ["(", "("]
```

The string is still invalid because the opening brackets were never closed.

```python
return not stack
```

returns `False` when the stack contains any unmatched opening brackets.

---

## Python Stack Operations

Python lists support stack behavior:

```python
stack.append(value)  # push onto top
stack[-1]            # inspect top without removing it
stack.pop()          # remove and return top
```

All three operations take `O(1)` time at the end of a list.

Do not use `pop(0)` for stack behavior. Removing index `0` shifts the remaining list elements and takes `O(n)` time.

---

## Edge Cases

### One unmatched opening bracket

```text
"(" → False
```

### One unmatched closing bracket

```text
")" → False
```

### Correct adjacent pairs

```text
"()[]{}" → True
```

### Correct nested pairs

```text
"{[()]}" → True
```

### Incorrect bracket type

```text
"(]" → False
```

### Incorrect nesting order

```text
"([)]" → False
```

---

## Complexity Analysis

### Time: `O(n)`

Every character is examined once. Each opening bracket is pushed once and each matched opening bracket is popped once.

### Extra space: `O(n)`

In the worst case, every character is an opening bracket:

```text
"(([[{{"
```

The stack then contains all `n` characters.

---

## Common Mistakes

- [ ] Checking only whether opening and closing bracket counts are equal.
- [ ] Ignoring nesting order.
- [ ] Accessing `stack[-1]` before checking whether the stack is empty.
- [ ] Popping an opening bracket before confirming the closing type matches.
- [ ] Returning `True` without checking for unmatched opening brackets at the end.
- [ ] Using queue behavior instead of Last In, First Out behavior.
- [ ] Using an unused index from `enumerate()` when iterating directly is enough.

---

## Review Questions

1. Why does this problem require Last In, First Out order?
2. What does the map store?
3. Why must the stack be checked before reading its top?
4. Why is `([)]` invalid even though every bracket type has a matching count?
5. Why does the function return `not stack` after the loop?
6. What are the time complexities of `append()`, `stack[-1]`, and `pop()`?
7. What is the worst-case stack size?

---

## Review History

| Date | Result | Time taken | Needed a hint? | Notes |
|---|---|---:|---|---|
| 2026-10-28 | Solved | | No | Correctly matched each closing bracket against the most recent opening bracket |
| 2026-11-04 | | | | First scheduled retry |
| | | | | |
