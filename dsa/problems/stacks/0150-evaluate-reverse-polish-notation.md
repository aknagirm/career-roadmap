# 150. Evaluate Reverse Polish Notation

- **Difficulty:** Medium
- **Primary pattern:** Stack
- **Status:** Solved
- **Language:** Python
- **LeetCode:** https://leetcode.com/problems/evaluate-reverse-polish-notation/
- **First solved:** 2026-10-28
- **Next review:** 2026-11-04

---

## Problem in My Own Words

Evaluate an arithmetic expression written in Reverse Polish Notation (RPN), also called postfix notation.

Valid operators are:

```text
+  -  *  /
```

Each operator applies to the two most recent values. Division between integers must truncate toward zero.

### Infix compared with postfix

```text
Infix:   (2 + 1) * 3
Postfix:  2 1 + 3 *
```

RPN does not need parentheses because each operator appears after its operands.

---

## Recognition Clues

- Numbers must be saved until an operator appears.
- Every operator uses the two most recently produced values.
- An operation's result may become an operand for a later operation.
- This follows Last In, First Out order.

These clues indicate a **Stack**:

```text
Number   → push
Operator → pop right, pop left, calculate, push result
```

---

## My Solution

```python
class Solution:
    def evalRPN(self, tokens: list[str]) -> int:
        stack = []

        for n in tokens:
            if n == "/" or n == "*" or n == "+" or n == "-":
                l1 = stack.pop()
                l2 = stack.pop()

                if n == "/":
                    stack.append(int(l2 / l1))
                if n == "*":
                    stack.append(l2 * l1)
                if n == "+":
                    stack.append(l2 + l1)
                if n == "-":
                    stack.append(l2 - l1)
            else:
                stack.append(int(n))

        return stack[0]
```

This solution is correct.

The first popped value, `l1`, is the right operand. The second popped value, `l2`, is the left operand. Therefore, subtraction and division correctly use:

```python
l2 - l1
l2 / l1
```

A valid RPN expression leaves exactly one value in the stack, so `stack[0]` returns the correct result.

---

## Refined Version

```python
class Solution:
    def evalRPN(self, tokens: list[str]) -> int:
        stack = []

        for token in tokens:
            if token in {"+", "-", "*", "/"}:
                right = stack.pop()
                left = stack.pop()

                if token == "+":
                    stack.append(left + right)
                elif token == "-":
                    stack.append(left - right)
                elif token == "*":
                    stack.append(left * right)
                else:
                    stack.append(int(left / right))
            else:
                stack.append(int(token))

        return stack[-1]
```

### Refinements

- `token` describes the loop value more clearly.
- Set membership replaces four `or` comparisons.
- `left` and `right` make operand order explicit.
- `elif` communicates that exactly one operator branch can execute.
- `stack[-1]` communicates that the final answer is on top of the stack.

---

## Operand Order

Operand order matters for subtraction and division.

For:

```text
tokens = ["10", "4", "-"]
```

the stack before `-` is:

```text
[10, 4]
```

Pop in this order:

```python
right = stack.pop()  # 4
left = stack.pop()   # 10
```

Then evaluate:

```text
left - right
10 - 4 = 6
```

Reversing the operands would incorrectly calculate `4 - 10`.

Addition and multiplication can hide this bug because their operand order does not change the result.

---

## Division Toward Zero

Truncating toward zero means removing the fractional part and moving the result closer to zero:

| Division | Decimal result | Required integer |
|---|---:|---:|
| `7 / 2` | `3.5` | `3` |
| `-7 / 2` | `-3.5` | `-3` |
| `7 / -2` | `-3.5` | `-3` |
| `-7 / -2` | `3.5` | `3` |

Python's `int()` truncates toward zero:

```python
int(-7 / 2)  # -3
```

Do not use floor division for this problem:

```python
-7 // 2  # -4
```

`//` rounds downward toward negative infinity, which differs from the problem requirement for negative results.

---

## Detailed Dry Run

### Input

```text
tokens = ["2", "1", "+", "3", "*"]
```

| Step | Token | Action | Stack afterward |
|---:|---|---|---|
| 1 | `"2"` | Push `2` | `[2]` |
| 2 | `"1"` | Push `1` | `[2, 1]` |
| 3 | `"+"` | Pop `1` and `2`; push `2 + 1` | `[3]` |
| 4 | `"3"` | Push `3` | `[3, 3]` |
| 5 | `"*"` | Pop `3` and `3`; push `3 * 3` | `[9]` |

Result:

```python
9
```

---

## Division Dry Run

### Input

```text
tokens = ["4", "13", "5", "/", "+"]
```

| Step | Token | Action | Stack afterward |
|---:|---|---|---|
| 1 | `"4"` | Push `4` | `[4]` |
| 2 | `"13"` | Push `13` | `[4, 13]` |
| 3 | `"5"` | Push `5` | `[4, 13, 5]` |
| 4 | `"/"` | Calculate `int(13 / 5) = 2` | `[4, 2]` |
| 5 | `"+"` | Calculate `4 + 2 = 6` | `[6]` |

Result:

```python
6
```

---

## Negative Number Tokens

A negative number such as `"-11"` is an operand, not the subtraction operator `"-"`.

Explicitly checking the four operator strings handles this correctly:

```python
if token in {"+", "-", "*", "/"}:
    # operator
else:
    stack.append(int(token))
```

Avoid relying only on `isdigit()`:

```python
"-11".isdigit()  # False
```

---

## Complexity Analysis

### Time: `O(n)`

Every token is processed exactly once. Stack pushes and pops from the end of a Python list take `O(1)` time.

### Extra space: `O(n)`

The stack can hold many operands before operators reduce them.

---

## Common Mistakes

- [ ] Popping the left operand before the right operand and then reversing subtraction or division.
- [ ] Using Python's `//` operator for negative division.
- [ ] Treating a negative number token such as `"-11"` as an operator.
- [ ] Calculating a result without pushing it back onto the stack.
- [ ] Returning before all tokens have been processed.
- [ ] Returning the original tokens rather than the final stack value.
- [ ] Assuming the first and second popped values have the same operand roles.

---

## Review Questions

1. What is the difference between infix and postfix notation?
2. What happens when the current token is a number?
3. What happens when the current token is an operator?
4. Why is the first popped value the right operand?
5. Which operations reveal reversed operand order?
6. Why is `int(left / right)` used instead of `left // right`?
7. Why does a valid expression leave one value in the stack?
8. What are the time and space complexities?

---

## Review History

| Date | Result | Time taken | Needed a hint? | Notes |
|---|---|---:|---|---|
| 2026-10-28 | Solved | | Yes | Correctly handled operand order and division toward zero |
| 2026-11-04 | | | | First scheduled retry |
| | | | | |
