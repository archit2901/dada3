# huskyutils: the open issues

Fixes for issues #1, #2, #4, #5, #8, and #10 were verified and are ready to review; issue #6 still needs a person; and issues #3, #7, and #9 were closed without a code change.

| Issue | Triage | What happened |
|---|---|---|
| #1 Slugs have double hyphens and a hyphen at the end | bug | fix verified, 1 verifier round(s) |
| #2 Shared payment loses a cent | bug | fix verified, 1 verifier round(s) |
| #3 word_count says "don't stop" is 2 words | not_a_bug | closed, no code change |
| #4 Delivery date skips over the weekend wrong | bug | fix verified, 1 verifier round(s) |
| #5 Delivery windows overlap after merging | bug | fix verified, 1 verifier round(s) |
| #6 A 1 kg package is charged $9 shipping | bug | stopped by an error, 0 verifier round(s) |
| #7 Free shipping for orders over $50 | enhancement | closed, no code change |
| #8 Median on the sales dashboard is wrong for an even number of days | bug | fix verified, 1 verifier round(s) |
| #9 5 kg package should be in the heavy tier | not_a_bug | closed, no code change |
| #10 Product card description runs past the edge of the card | bug | fix verified, 1 verifier round(s) |

## #1: Slugs have double hyphens and a hyphen at the end

**Triage:** bug. slugify replaces every non-alphanumeric character with a hyphen individually instead of collapsing runs of them into a single hyphen and stripping leading or trailing hyphens, which contradicts the docstring.

**Fix** (fix verified): Fixed slugify in huskyutils/text.py to handle accent stripping, collapsing runs of hyphens, and stripping leading/trailing hyphens as specified in the docstring.

```diff
--- a/huskyutils/text.py
+++ b/huskyutils/text.py
@@ -1,4 +1,6 @@
 """Text helpers for product pages."""
+import re
+import unicodedata
 
 
 def slugify(title):
@@ -13,10 +15,17 @@
         slugify("USB-C Dock (2nd gen)") == "usb-c-dock-2nd-gen"
         slugify("Crème Brûlée Mug") == "creme-brulee-mug"
     """
+    # Normalize unicode to handle accented letters
+    normalized = unicodedata.normalize("NFKD", title)
+    ascii_title = normalized.encode("ascii", "ignore").decode("ascii")
+
     out = []
-    for ch in title.lower():
+    for ch in ascii_title.lower():
         out.append(ch if ch.isalnum() else "-")
-    return "".join(out)
+    
+    slug = "".join(out)
+    slug = re.sub(r"-+", "-", slug)
+    return slug.strip("-")
 
 
 def word_count(text):
```

**Verifier:** Ran tests in original/ and fixed/ using unittest. In original/, tests/test_issue.py failed as expected (reproducing the issue), while in fixed/, all tests including tests/test_issue.py passed successfully. Commands run: 'cd /home/daytona/original && python -m unittest discover -s tests -v' and 'cd /home/daytona/fixed && python -m unittest discover -s tests -v'.

<details><summary>The verifier's test</summary>

```python
import unittest
from huskyutils.text import slugify


class TestSlugifyIssue(unittest.TestCase):
    def test_docstring_examples(self):
        self.assertEqual(slugify("Wireless Headphones"), "wireless-headphones")
        self.assertEqual(slugify("USB-C Dock (2nd gen)"), "usb-c-dock-2nd-gen")
        self.assertEqual(slugify("Crème Brûlée Mug"), "creme-brulee-mug")

    def test_issue_cases(self):
        self.assertEqual(slugify("Hello, World! T-shirt"), "hello-world-t-shirt")
        self.assertEqual(slugify("Webcam (HD)"), "webcam-hd")
        self.assertEqual(slugify(""), "")
        self.assertEqual(slugify("   "), "")
        self.assertEqual(slugify("!@#$"), "")
```

</details>

## #2: Shared payment loses a cent

**Triage:** bug. The function `split_amount` simply integer-divides `cents` by `people`, ignoring remainders and failing to distribute extra cents as required by its docstring.

**Fix** (fix verified): Fixed split_amount to distribute remainder cents to the first shares as specified in the docstring.

```diff
--- a/huskyutils/money.py
+++ b/huskyutils/money.py
@@ -13,4 +13,46 @@
         split_amount(-1000, 3) == [-334, -333, -333]
     """
     share = cents // people
-    return [share] * people
+    remainder = cents % people
+    shares = [share] * people
+    # Distribute the remainder (positive or negative) to the first shares.
+    # Note: Python's % and // operators mean remainder has the same sign as divisor (people, > 0)
+    # or is 0. Wait, let's check how negative amounts work with % and // in Python.
+    # For cents = -1000, people = 3:
+    # -1000 // 3 = -334 (since -334 * 3 = -1002)
+    # -1000 % 3 = 2 (-1002 + 2 = -1000).
+    # Wait, docstring examples:
+        # split_amount(1000, 3) == [334, 333, 333]
+        # split_amount(-1000, 3) == [-334, -333, -333]
+    # Let's check:
+    # If cents = 1000, people = 3: share = 333, remainder = 1. We want [334, 333, 333].
+    # That means the first `remainder` shares get +1.
+    # If cents = -1000, people = 3: share = -334, remainder = 2? Wait!
+    # -1000 // 3 = -334. remainder = -1000 % 3 = 2.
+    # If share is -334 and remainder is 2, then `[share] * people` is `[-334, -334, -334]`.
+    # Adding +1 to the first 2 shares gives `[-333, -333, -334]`, but the docstring says:
+    # `split_amount(-1000, 3) == [-334, -333, -333]`!
+    # Let's re-read the docstring carefully:
+    # "A negative amount (a refund) is split the same way as the positive amount, with every share negative."
+    # Wait! Let's trace `split_amount(-1000, 3) == [-334, -333, -333]`.
+    # Here, the first share is -334, and the others are -333.
+    # In positive case `1000, 3`: `[334, 333, 333]` (first share gets the extra cent, i.e., larger magnitude).
+    # For negative case `-1000, 3`: `[-334, -333, -333]` (first share gets -334, which is larger magnitude negative, i.e. 1 cent more negative).
+    # Let's verify integer division vs absolute value division or similar.
+    # If we do division on absolute values, or handle sign:
+    # Let's check:
+    # sign = 1 if cents >= 0 else -1
+    # abs_shares = split_amount(abs(cents), people)
+    # return [s * sign for s in abs_shares]
+    # Wait, let's test that logic:
+    # abs(1000), 3 -> split_amount(1000, 3) -> [334, 333, 333] -> sign 1 -> [334, 333, 333].
+    # abs(-1000), 3 -> split_amount(1000, 3) -> [334, 333, 333] -> sign -1 -> [-334, -333, -333].
+    # Does that match docstring example `split_amount(-1000, 3) == [-334, -333, -333]`? Yes!
+    # Let's check `split_amount(900, 3) == [300, 300, 300]`.
+    # Let's implement `split_amount` cleanly using absolute values or explicit remainder distribution.
+    if cents < 0:
+        return [-s for s in split_amount(-cents, people)]
+    
+    share = cents // people
+    remainder = cents % people
+    return [share + (1 if i < remainder else 0) for i in range(people)]
```

**Verifier:** Ran `python -m unittest discover -s tests -v` in /home/daytona/original, which failed on test_split_amount_examples with AssertionError ([333, 333, 333] != [334, 333, 333]), successfully reproducing the issue. Ran the same test suite in /home/daytona/fixed, where all 26 tests passed successfully.

<details><summary>The verifier's test</summary>

```python
import unittest

from huskyutils.money import split_amount


class TestSplitAmountIssue(unittest.TestCase):
    def test_split_amount_examples(self):
        self.assertEqual(split_amount(900, 3), [300, 300, 300])
        self.assertEqual(split_amount(1000, 3), [334, 333, 333])
        self.assertEqual(split_amount(-1000, 3), [-334, -333, -333])
```

</details>

## #3: word_count says "don't stop" is 2 words

**Triage:** not_a_bug. The docstring states that punctuation stays attached to the word it touches and "don't" is one word. The code correctly implements this by splitting on whitespace, so the reporter expected something contrary to the specification.

## #4: Delivery date skips over the weekend wrong

**Triage:** bug. The function add_business_days simply adds days and checks if the final result is a weekend or holiday, failing to correctly count multiple business days sequentially.

**Fix** (fix verified): Fixed add_business_days to count business days sequentially one by one instead of adding total days at once and skipping trailing weekends/holidays.

```diff
--- a/huskyutils/dates.py
+++ b/huskyutils/dates.py
@@ -13,9 +13,9 @@
         add_business_days(date(2026, 11, 6), 1) == date(2026, 11, 9)                             # Monday
         add_business_days(date(2026, 11, 6), 1, holidays={date(2026, 11, 9)}) == date(2026, 11, 10)
     """
-    if days == 0:
-        return start
-    result = start + timedelta(days=days)
-    while result.weekday() >= 5 or result in holidays:
+    result = start
+    for _ in range(days):
         result += timedelta(days=1)
+        while result.weekday() >= 5 or result in holidays:
+            result += timedelta(days=1)
     return result
```

**Verifier:** Ran `python -m unittest discover -s tests -v` in both original/ and fixed/ repositories. In original/, our new test `test_issue_4_thursday_plus_3_business_days` failed with `AssertionError: datetime.date(2026, 11, 9) != datetime.date(2026, 11, 10)`, reproducing Issue #4 correctly. In fixed/, all 27 tests passed successfully (including our test and existing tests).

<details><summary>The verifier's test</summary>

```python
import unittest
from datetime import date

from huskyutils.dates import add_business_days


class TestIssue4(unittest.TestCase):
    def test_issue_4_thursday_plus_3_business_days(self):
        # Issue description:
        # An order placed on Thursday 2026-11-05 with 3 business days of handling
        # shows a delivery date of Monday 2026-11-09. Thursday plus 3 business days
        # is Friday, Monday, Tuesday, so it should be Tuesday 2026-11-10.
        # >>> add_business_days(date(2026, 11, 5), 3)
        # datetime.date(2026, 11, 10)
        self.assertEqual(add_business_days(date(2026, 11, 5), 3), date(2026, 11, 10))

    def test_docstring_examples(self):
        # # 2026-11-06 is a Friday
        # add_business_days(date(2026, 11, 6), 1) == date(2026, 11, 9)                             # Monday
        # add_business_days(date(2026, 11, 6), 1, holidays={date(2026, 11, 9)}) == date(2026, 11, 10)
        self.assertEqual(add_business_days(date(2026, 11, 6), 1), date(2026, 11, 9))
        self.assertEqual(add_business_days(date(2026, 11, 6), 1, holidays={date(2026, 11, 9)}), date(2026, 11, 10))
```

</details>

## #5: Delivery windows overlap after merging

**Triage:** bug. The docstring states that input intervals can be in any order, but the code does not sort them before merging, leading to failure when overlapping intervals are provided out of order.

**Fix** (fix verified): Sort intervals by start time before merging so that out-of-order intervals are merged correctly as specified in the docstring.

```diff
--- a/huskyutils/intervals.py
+++ b/huskyutils/intervals.py
@@ -13,8 +13,11 @@
         merge_intervals([[1, 3], [2, 6], [8, 10]]) == [[1, 6], [8, 10]]
         merge_intervals([[1, 4], [4, 5]]) == [[1, 5]]
     """
+    if not intervals:
+        return []
+    sorted_intervals = sorted(intervals, key=lambda x: x[0])
     merged = []
-    for start, end in intervals:
+    for start, end in sorted_intervals:
         if merged and start <= merged[-1][1]:
             merged[-1][1] = max(merged[-1][1], end)
         else:
```

**Verifier:** Ran tests in /home/daytona/original and /home/daytona/fixed using `python -m unittest discover -s tests -v`. The new test `test_docstring_example_unsorted` failed in `original/` (reproducing the issue), and passed along with all other tests in `fixed/`.

<details><summary>The verifier's test</summary>

```python
import unittest

from huskyutils.intervals import merge_intervals


class TestIssueIntervals(unittest.TestCase):
    def test_docstring_example_unsorted(self):
        self.assertEqual(merge_intervals([[8, 10], [1, 3], [2, 6]]), [[1, 6], [8, 10]])

    def test_docstring_examples(self):
        self.assertEqual(merge_intervals([[1, 3], [2, 6], [8, 10]]), [[1, 6], [8, 10]])
        self.assertEqual(merge_intervals([[1, 4], [4, 5]]), [[1, 5]])
```

</details>

## #6: A 1 kg package is charged $9 shipping

**Triage:** bug. The docstring states packages up to and including 1 kg cost $5.00, but the code checks `weight_kg < 1` for the $5 tier, so exactly 1 kg falls into the $9 tier.

**Fix:** none. ReadError: [Errno 54] Connection reset by peer

## #7: Free shipping for orders over $50

**Triage:** enhancement. The shipping_cost function calculates shipping based on weight, while the issue requests a feature for free shipping based on order total. This is a new capability not promised by the docstring.

## #8: Median on the sales dashboard is wrong for an even number of days

**Triage:** bug. The median function for an even list returns the upper middle value instead of the average of the two middle values, which violates its docstring specification.

**Fix** (fix verified): Fixed median calculation for lists with an even number of elements so it returns the average of the two middle values instead of the upper middle value.

```diff
--- a/huskyutils/stats.py
+++ b/huskyutils/stats.py
@@ -11,4 +11,8 @@
         median([4, 1, 3, 2]) == 2.5
     """
     ordered = sorted(values)
-    return ordered[len(ordered) // 2]
+    n = len(ordered)
+    if n % 2 == 1:
+        return ordered[n // 2]
+    else:
+        return (ordered[n // 2 - 1] + ordered[n // 2]) / 2
```

**Verifier:** Ran `python -m unittest discover -s tests -v` in /home/daytona/original/ and all tests including the new test_issue.py failed as expected (reproducing the issue). Ran the same test suite in /home/daytona/fixed/ and all 27 tests passed successfully.

<details><summary>The verifier's test</summary>

```python
import unittest

from huskyutils.stats import median


class TestIssueMedian(unittest.TestCase):
    def test_even_number_of_days_docstring_example(self):
        self.assertEqual(median([4, 1, 3, 2]), 2.5)

    def test_even_number_of_days_issue_example(self):
        self.assertEqual(median([10, 20, 30, 40]), 25.0)
```

</details>

## #9: 5 kg package should be in the heavy tier

**Triage:** not_a_bug. The docstring states that packages over 5 kg cost $9.00 plus $2.00 for each started kilogram over 5, and up to and including 5 kg cost $9.00. A 5 kg package returns 900, which matches the docstring specification.

## #10: Product card description runs past the edge of the card

**Triage:** bug. The truncate function implementation simply slices the string at `limit` and appends '...', which ignores the requirement that words should not be cut in the middle and causes the length to exceed `limit`.

**Fix** (fix verified): Fixed truncate in huskyutils/cards.py to respect whole words without cutting in the middle, ensuring the returned string length is at most limit and ends with '...' when truncated.

```diff
--- a/huskyutils/cards.py
+++ b/huskyutils/cards.py
@@ -15,4 +15,20 @@
     """
     if len(text) <= limit:
         return text
-    return text[:limit] + "..."
+
+    words = text.split()
+    candidate = ""
+    for word in words:
+        if not candidate:
+            next_candidate = word
+        else:
+            next_candidate = candidate + " " + word
+        if len(next_candidate) + 3 <= limit:
+            candidate = next_candidate
+        else:
+            break
+
+    if candidate:
+        return candidate + "..."
+    else:
+        return text[:limit - 3] + "..."
```

**Verifier:** Ran `python -m unittest discover -s tests -v` in both original/ and fixed/. In original/, the tests in test_issue.py failed as expected (reproducing the issue where length exceeded 30 and docstring example expected word-boundary truncation). In fixed/, all 27 tests passed successfully.

<details><summary>The verifier's test</summary>

```python
import unittest

from huskyutils.cards import truncate


class TestIssue10(unittest.TestCase):
    def test_docstring_examples(self):
        self.assertEqual(truncate("Wireless headphones", 30), "Wireless headphones")
        self.assertEqual(truncate("Wireless noise-cancelling headphones with case", 30), "Wireless noise-cancelling...")

    def test_length_within_limit(self):
        s = truncate("Wireless noise-cancelling headphones with case", 30)
        self.assertLessEqual(len(s), 30)
        self.assertEqual(s, "Wireless noise-cancelling...")
```

</details>