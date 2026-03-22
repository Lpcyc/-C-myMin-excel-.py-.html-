# C++ Bug 追踪记录

记录 C++ 开发过程中遇到并解决的 Bug，包含根本原因分析与修复说明。

---

## Bug 记录模板

新增 Bug 时请使用以下模板填写。

```
### BUG-XXX: <简短标题>

| 字段     | 内容 |
|----------|------|
| 日期     | YYYY-MM-DD |
| 严重程度 | 严重 / 高 / 中 / 低 |
| 状态     | 待处理 / 处理中 / 已解决 |
| 模块     | <模块或文件名> |
| 报告人   | <姓名或账号> |

**问题描述**
简明描述出现的异常行为。

**最小复现代码**
```cpp
// 触发 Bug 的最小代码片段
```

**根本原因**
说明 Bug 产生的原因。

**修复方案**
描述为解决问题所做的修改。

**经验教训**
避免同类 Bug 的关键要点。
```

---

## Bug 记录

---

### BUG-001: map 默认值陷阱

| 字段     | 内容 |
|----------|------|
| 日期     | 2026-03-20 |
| 严重程度 | 中 |
| 状态     | 已解决 |
| 模块     | 频率统计 / 查找逻辑 |
| 报告人   | Lpcyc |

**问题描述**
在使用 `std::map` 检查某个键是否存在时，若通过下标运算符 `[]` 查询一个不存在的键，map 会悄无声息地将该键以默认值（整型为 0）插入容器。这会导致 map 的大小被错误地撑大，并在后续代码中产生"键已存在"的错误判断。

**最小复现代码**
```cpp
#include <iostream>
#include <map>

int main() {
    std::map<int, int> freq;
    freq[1] = 3;
    freq[2] = 5;

    // BUG：用 [] 查询不存在的键，会将该键以值 0 插入 map
    if (freq[3] == 0) {
        std::cout << "键 3 不存在\n"; // 会打印，但键 3 已经被插入 map 了！
    }

    std::cout << "Map 大小：" << freq.size() << "\n"; // 打印 3，预期为 2
    return 0;
}
```

**根本原因**
`std::map::operator[]` 的规范行为是"插入或赋值"：若键不存在，会先默认构造一个值并将键值对插入容器，再返回该值的引用。因此任何通过 `[]` 进行的读操作都会改变容器状态。

**修复方案**
使用 `std::map::count()` 或 `std::map::find()` 进行无副作用的存在性查询。

```cpp
// 方案 A —— count（对 std::map 返回 0 或 1）
if (freq.count(3) == 0) {
    std::cout << "键 3 不存在\n"; // map 大小保持为 2
}

// 方案 B —— find
auto it = freq.find(3);
if (it == freq.end()) {
    std::cout << "键 3 不存在\n";
} else {
    std::cout << "值：" << it->second << "\n";
}

// 方案 C —— C++20 contains()
if (!freq.contains(3)) {
    std::cout << "键 3 不存在\n";
}
```

**经验教训**
- 不要用 `map[key]` 做纯粹的读取或存在性检查，它有插入副作用。
- 需要同时检查存在性并获取值时，优先使用 `find()`。
- 开启 AddressSanitizer 或在单元测试中加入容器大小断言，以便及早发现容器被意外撑大的问题。

---

### BUG-002: 大数模拟边界情况

| 字段     | 内容 |
|----------|------|
| 日期     | 2026-03-21 |
| 严重程度 | 高 |
| 状态     | 已解决 |
| 模块     | 大整数运算（数组 / 字符串模拟） |
| 报告人   | Lpcyc |

**问题描述**
手写的大整数实现（用 `std::vector<int>` 或 `std::string` 存储各位数字）在以下几个边界情况下产生了错误结果或崩溃：

1. 两数相加，其中一个操作数为 `"0"` 时，返回 `"00"` 而非 `"0"`。
2. 任意数乘以 `"0"` 时，返回全为零的长字符串而非 `"0"`。
3. 两个相等的数相减，返回 `""` （空字符串）而非 `"0"`。
4. 进位运算结束后，前导零未被去除。

**最小复现代码**
```cpp
#include <iostream>
#include <string>
#include <algorithm>

// 简化版大数加法（有 Bug 的版本）
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
    return result; // BUG："0" + "0" 返回 "00"，而非 "0"
}

int main() {
    std::cout << addBig("0", "0") << "\n"; // 打印 "00"，预期为 "0"
    return 0;
}
```

**根本原因**

| 序号 | 边界情况 | 根本原因 |
|------|----------|----------|
| 1 | `"0" + "0"` → `"00"` | 构造完成后未去除前导零 |
| 2 | `n × "0"` → `"000…0"` | 缺少对零操作数的提前退出判断 |
| 3 | `n − n` → `""` | 循环结束时，结果为空时未补充 `'0'` |
| 4 | 一般性前导零问题 | 返回前未调用 `stripLeadingZeros` 辅助函数 |

**修复方案**
```cpp
#include <iostream>
#include <string>
#include <algorithm>

// 去除前导零；至少保留一位数字
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
    if (result.empty()) result = "0";          // 修复边界情况 3
    std::reverse(result.begin(), result.end());
    return stripLeadingZeros(result);           // 修复边界情况 1 & 4
}

std::string multiplyBig(const std::string& a, const std::string& b) {
    if (a == "0" || b == "0") return "0";      // 修复边界情况 2
    // … 正常乘法逻辑 …
    std::string result = "0";
    // （具体实现略）
    return stripLeadingZeros(result);
}

int main() {
    std::cout << addBig("0", "0")       << "\n"; // "0"    ✓
    std::cout << addBig("999", "1")     << "\n"; // "1000" ✓
    std::cout << addBig("123", "456")   << "\n"; // "579"  ✓
    return 0;
}
```

**经验教训**
- 在进入主循环**之前**，始终将零操作数作为特殊情况单独处理。
- 每次返回大数结果前，无条件调用 `stripLeadingZeros` 辅助函数。
- 对空结果字符串做保护，默认返回 `"0"`。
- 编写专项单元测试，覆盖：`0+0`、`0×n`、`n−n`、单位数操作数，以及两个长度不同的数相运算等情况。

---

*最后更新：2026-03-22*
