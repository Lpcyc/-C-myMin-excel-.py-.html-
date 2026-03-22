# C++ Bug Tracking Log

A running record of bugs encountered and resolved during C++ development, along with root‑cause analysis and fix notes.

---

## Bug Record Template

Use the following template when logging a new bug.

```
### BUG-XXX: <Short Title>

| Field        | Detail |
|--------------|--------|
| Date         | YYYY-MM-DD |
| Severity     | Critical / High / Medium / Low |
| Status       | Open / In Progress / Resolved |
| Component    | <module or file name> |
| Reporter     | <name or handle> |

**Description**
A concise description of the unexpected behaviour.

**Minimal Reproducer**
```cpp
// smallest code snippet that triggers the bug
```

**Root Cause**
Explanation of why the bug occurs.

**Fix**
What was changed to resolve the issue.

**Lessons Learned**
Key takeaway to prevent similar bugs in the future.
```

---

## Bug Records

---

### BUG-001: Map Default Value Trap

| Field        | Detail |
|--------------|--------|
| Date         | 2026-03-20 |
| Severity     | Medium |
| Status       | Resolved |
| Component    | Frequency counting / lookup logic |
| Reporter     | Lpcyc |

**Description**
When checking whether a key exists in a `std::map`, using the subscript operator `[]` to query the value silently inserts a default‑constructed entry (0 for integers) into the map if the key is absent. This inflates the map's size and causes incorrect "key exists" assumptions later in the code.

**Minimal Reproducer**
```cpp
#include <iostream>
#include <map>

int main() {
    std::map<int, int> freq;
    freq[1] = 3;
    freq[2] = 5;

    // BUG: querying a missing key with [] inserts it with value 0
    if (freq[3] == 0) {
        std::cout << "Key 3 not found\n"; // prints, but key 3 is now IN the map!
    }

    std::cout << "Map size: " << freq.size() << "\n"; // prints 3, expected 2
    return 0;
}
```

**Root Cause**
`std::map::operator[]` is specified to perform an *insert‑or‑assign* operation: if the key does not exist it default‑constructs a value and inserts the new pair before returning a reference to it. Any read through `[]` therefore mutates the container.

**Fix**
Use `std::map::count()` or `std::map::find()` to query existence without side effects.

```cpp
// Option A – count (returns 0 or 1 for std::map)
if (freq.count(3) == 0) {
    std::cout << "Key 3 not found\n"; // map size stays 2
}

// Option B – find
auto it = freq.find(3);
if (it == freq.end()) {
    std::cout << "Key 3 not found\n";
} else {
    std::cout << "Value: " << it->second << "\n";
}

// Option C – C++20 contains()
if (!freq.contains(3)) {
    std::cout << "Key 3 not found\n";
}
```

**Lessons Learned**
- Never use `map[key]` purely for a read/existence check; it has an insertion side‑effect.
- Prefer `find()` when you need both existence check and value access in one step.
- Enable address‑sanitizer or add size‑assertion unit tests to catch unexpected container growth early.

---

### BUG-002: Big Number Simulation Edge Cases

| Field        | Detail |
|--------------|--------|
| Date         | 2026-03-21 |
| Severity     | High |
| Status       | Resolved |
| Component    | Big integer arithmetic (array / string simulation) |
| Reporter     | Lpcyc |

**Description**
A hand‑rolled big‑integer implementation (storing digits in a `std::vector<int>` or `std::string`) produced wrong results or crashed on several edge cases:

1. Adding two numbers where one operand is `"0"` returned `"00"` instead of `"0"`.
2. Multiplying any number by `"0"` returned the full‑length zero‑padded string instead of `"0"`.
3. Subtraction of equal numbers returned `""` (empty string) instead of `"0"`.
4. Leading zeros were not stripped after carry propagation.

**Minimal Reproducer**
```cpp
#include <iostream>
#include <string>
#include <algorithm>

// Simplified big-number addition (buggy version)
std::string addBig(const std::string& a, const std::string& b) {
    std::string result;
    int carry = 0, i = a.size() - 1, j = b.size() - 1;
    while (i >= 0 || j >= 0 || carry) {
        int sum = carry;
        if (i >= 0) sum += a[i--] - '0';
        if (j >= 0) sum += b[j--] - '0';
        carry = sum / 10;
        result += char('0' + sum % 10);
    }
    std::reverse(result.begin(), result.end());
    return result; // BUG: returns "0" + "0" = "00", not "0"
}

int main() {
    std::cout << addBig("0", "0") << "\n"; // prints "00", expected "0"
    return 0;
}
```

**Root Cause**

| # | Edge Case | Root Cause |
|---|-----------|------------|
| 1 | `"0" + "0"` → `"00"` | No leading‑zero strip after construction |
| 2 | `n × "0"` → `"000…0"` | Early‑exit check for zero operand missing |
| 3 | `n − n` → `""` | Loop exits without pushing a `'0'` when result is empty |
| 4 | General leading zeros | `stripLeadingZeros` helper not called before returning |

**Fix**
```cpp
#include <iostream>
#include <string>
#include <algorithm>

// Strip leading zeros; always keep at least one digit
std::string stripLeadingZeros(const std::string& s) {
    size_t start = s.find_first_not_of('0');
    return (start == std::string::npos) ? "0" : s.substr(start);
}

std::string addBig(const std::string& a, const std::string& b) {
    std::string result;
    int carry = 0, i = (int)a.size() - 1, j = (int)b.size() - 1;
    while (i >= 0 || j >= 0 || carry) {
        int sum = carry;
        if (i >= 0) sum += a[i--] - '0';
        if (j >= 0) sum += b[j--] - '0';
        carry = sum / 10;
        result += char('0' + sum % 10);
    }
    if (result.empty()) result = "0";          // FIX edge case 3
    std::reverse(result.begin(), result.end());
    return stripLeadingZeros(result);           // FIX edge cases 1 & 4
}

std::string multiplyBig(const std::string& a, const std::string& b) {
    if (a == "0" || b == "0") return "0";      // FIX edge case 2
    // … normal multiplication logic …
    std::string result = "0";
    // (implementation omitted for brevity)
    return stripLeadingZeros(result);
}

int main() {
    std::cout << addBig("0", "0")       << "\n"; // "0"   ✓
    std::cout << addBig("999", "1")     << "\n"; // "1000" ✓
    std::cout << addBig("123", "456")   << "\n"; // "579"  ✓
    return 0;
}
```

**Lessons Learned**
- Always handle zero operands as a special case **before** entering the main loop.
- Call a `stripLeadingZeros` helper unconditionally before returning any big‑number result.
- Guard against an empty result string by defaulting to `"0"`.
- Write a dedicated unit‑test suite that covers: `0+0`, `0×n`, `n−n`, single‑digit operands, and numbers with different lengths.

---

*Last updated: 2026-03-22*
