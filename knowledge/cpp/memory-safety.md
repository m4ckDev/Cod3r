# Common C++ memory-safety bugs and safe fixes

Each section shows the bug (as `text`, never compiled) and a fixed, runnable version. Fixed examples build clean with:

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic file.cpp -o file && ./file
```

## 1. Buffer overflow

```text
char buf[8];
strcpy(buf, "this string is too long");   // writes past the end: undefined behavior
int a[4]; a[4] = 1;                        // one past the end
```

Fix: use owning types with bounds, and bounded formatting.

```cpp
#include <array>
#include <cstdio>
#include <iostream>
#include <stdexcept>
#include <string>
#include <vector>

int main() {
  std::string s = "this string is as long as it needs to be";   // grows automatically
  std::cout << s.size() << '\n';

  char buf[8];
  int n = std::snprintf(buf, sizeof buf, "%s", "too long for buf");  // truncates, always terminates
  std::cout << "wanted " << n << " bytes, kept: " << buf << '\n';

  std::array<int, 4> a{};
  try {
    a.at(4) = 1;                                                 // checked access throws
  } catch (const std::out_of_range&) {
    std::cout << "caught out_of_range\n";
  }

  std::vector<int> v(4);
  for (std::size_t i = 0; i < v.size(); ++i) v[i] = static_cast<int>(i);  // loop bound from the container
}
```

## 2. Use after free and dangling references

```text
int* p = new int(5); delete p; *p = 6;          // use after free
const std::string& r = make_string()[0];         // (variants) reference into a temporary
int& f() { int x = 1; return x; }                // reference to a dead local
auto it = v.begin(); v.push_back(1); *it;        // iterator invalidated by reallocation
std::string_view sv = std::string("temp");       // view of a destroyed temporary
```

Fix: ownership types, return by value, and re-acquire iterators after mutation.

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <string_view>
#include <vector>

std::string make_name() { return "ada"; }       // return by value, never a reference to a local

int main() {
  auto p = std::make_unique<int>(5);            // freed automatically, no manual delete
  *p = 6;
  std::cout << *p << '\n';

  std::vector<int> v = {1, 2, 3};
  v.push_back(4);                               // may reallocate
  auto it = v.begin();                          // take the iterator AFTER mutating
  std::cout << *it << '\n';

  std::string owner = make_name();              // keep the owner alive
  std::string_view sv = owner;                  // view is safe while `owner` lives
  std::cout << sv << '\n';

  const std::string& r = make_name();           // OK: lifetime extended to r's scope
  std::cout << r << '\n';
}
```

Remember that lifetime extension only works when a temporary binds directly to a local reference. It does not work through a function that returns a reference into it.

## 3. Double free and leaks

```text
int* p = new int[10];
delete[] p;
delete[] p;                // double free
int* q = new int; if (cond) return;   // leak on early return
```

Fix: one owner, released exactly once. `unique_ptr` deletes in its destructor, even on exceptions.

```cpp
#include <iostream>
#include <memory>
#include <stdexcept>
#include <vector>

void work(bool fail) {
  auto buf = std::make_unique<int[]>(10);       // value-initialized, delete[] handled for you
  buf[0] = 42;
  if (fail) throw std::runtime_error("early exit");   // no leak
  std::cout << "buf[0]=" << buf[0] << '\n';
}

int main() {
  work(false);
  try { work(true); } catch (const std::exception& e) { std::cout << "caught: " << e.what() << '\n'; }

  std::vector<int> v(10);                       // usually better than any array
  v[0] = 1;
}
```

## 4. Uninitialized variables

```text
int x;
if (x > 0) { ... }          // reads an indeterminate value
```

Fix: always initialize. Brace initialization also rejects narrowing.

```cpp
#include <iostream>
#include <string>

struct Config {
  int retries = 3;                               // default member initializers
  bool verbose = false;
  std::string name{};
};

int main() {
  int x{};                                       // 0
  int y = 7;
  Config c;                                      // all members defined
  std::cout << x << ' ' << y << ' ' << c.retries << ' ' << c.verbose << '\n';
  // int bad{3.5};                               // compile error: narrowing, caught for free
}
```

## 5. Null pointer dereference

```text
Widget* w = find("x");
w->run();                  // crashes if find() returned nullptr
```

Fix: return a type that forces handling, and check before use.

```cpp
#include <iostream>
#include <map>
#include <optional>
#include <string>

std::optional<int> lookup(const std::map<std::string, int>& m, const std::string& key) {
  auto it = m.find(key);
  if (it == m.end()) return std::nullopt;
  return it->second;
}

int main() {
  std::map<std::string, int> ports = {{"http", 80}, {"https", 443}};
  if (auto p = lookup(ports, "https")) std::cout << "https=" << *p << '\n';
  std::cout << "ftp=" << lookup(ports, "ftp").value_or(-1) << '\n';

  const int* raw = nullptr;
  if (raw != nullptr) std::cout << *raw << '\n';  // always guard raw pointers
  std::cout << "guarded\n";
}
```

## 6. Use after move

```text
std::string a = "data";
std::string b = std::move(a);
std::cout << a;            // valid but unspecified (often empty)
```

Fix: treat a moved-from object as dead unless you reassign it.

```cpp
#include <iostream>
#include <string>
#include <utility>
#include <vector>

int main() {
  std::string a = "data";
  std::vector<std::string> sink;
  sink.push_back(std::move(a));                  // a is now unspecified
  a = "fresh";                                   // reassigning makes it usable again
  std::cout << a << ' ' << sink[0] << '\n';
}
```

## 7. Signed overflow and bad conversions

```text
int big = INT_MAX; big + 1;           // signed overflow is UB
size_t n = v.size() - 1;              // wraps to huge when v is empty
for (unsigned i = 10; i >= 0; --i)    // infinite loop: unsigned is never < 0
```

Fix: check before arithmetic, and use the right loop idiom.

```cpp
#include <climits>
#include <cstddef>
#include <iostream>
#include <vector>

bool add_would_overflow(int a, int b) { return (b > 0 && a > INT_MAX - b) || (b < 0 && a < INT_MIN - b); }

int main() {
  std::cout << std::boolalpha << add_would_overflow(INT_MAX, 1) << '\n';

  std::vector<int> v;
  if (!v.empty()) std::cout << v[v.size() - 1] << '\n';          // guard before size() - 1

  for (std::size_t i = 3; i-- > 0;) std::cout << i << ' ';        // countdown with unsigned
  std::cout << '\n';
}
```

## Catch bugs automatically with sanitizers

```bash
g++ -std=c++17 -g -fsanitize=address,undefined -fno-omit-frame-pointer file.cpp -o file && ./file
```

| Tool | Flag | Detects |
|---|---|---|
| AddressSanitizer | `-fsanitize=address` | Overflow, use-after-free, double free, leaks |
| UBSan | `-fsanitize=undefined` | Signed overflow, bad shifts, null deref, misaligned access |
| ThreadSanitizer | `-fsanitize=thread` | Data races (cannot combine with ASan) |
| Valgrind | `valgrind ./file` | Uninitialized reads, leaks, no recompile needed |

Run tests under ASan + UBSan in CI. Sanitizer builds are slower and use more memory, so keep them separate from release builds.

## Common mistakes

| Mistake | Why it hurts | Fix |
|---|---|---|
| `strcpy` / `sprintf` / `gets` | Unbounded writes | `std::string`, `snprintf`, `std::getline` |
| `v[i]` with unchecked `i` | Out-of-bounds read or write | `at()`, or compare to `size()` |
| Returning a reference to a local | Dangling | Return by value |
| Keeping iterators across `push_back` / `erase` | Invalidated | Re-fetch after mutating |
| `string_view` / `c_str()` of a temporary | Dangles at the end of the statement | Store the `std::string` first |
| Owning raw pointers | Leaks, double free | `unique_ptr`, containers |
| Uninitialized locals and members | Indeterminate values | `{}` / default member initializers |
| `delete` vs `delete[]` mismatch | UB | Never pair them by hand, use `make_unique<T[]>` or `vector` |
| Ignoring compiler warnings | Misses real bugs | `-Wall -Wextra -Wpedantic`, treat warnings as errors in CI |
| Never running sanitizers | Bugs stay silent | Test with `-fsanitize=address,undefined` |
