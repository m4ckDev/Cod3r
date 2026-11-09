# Modern C++17 basics cheat sheet

Covers RAII, smart pointers, `std::string`, and `std::vector`. Every example is a complete program. Build and run with:

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic file.cpp -o file && ./file
```

## RAII: tie a resource to an object's lifetime

Acquire in the constructor, release in the destructor. The destructor runs on normal exit, early `return`, and exceptions, so cleanup cannot be forgotten.

```cpp
#include <cstdio>
#include <iostream>
#include <stdexcept>

class File {
 public:
  File(const char* path, const char* mode) : f_(std::fopen(path, mode)) {
    if (!f_) throw std::runtime_error(std::string("cannot open ") + path);
  }
  ~File() { std::fclose(f_); }
  File(const File&) = delete;             // copying would double-close
  File& operator=(const File&) = delete;
  std::FILE* get() const { return f_; }

 private:
  std::FILE* f_;
};

int main() {
  try {
    File ok("/dev/null", "w");
    std::fputs("hello\n", ok.get());
    std::cout << "wrote, file closes at end of scope\n";
    File bad("/nonexistent/dir/x.txt", "r");  // throws
  } catch (const std::exception& e) {
    std::cout << "caught: " << e.what() << '\n';
  }
}
```

Rules of thumb:
- No raw `new`/`delete`, `fopen`/`fclose`, `lock`/`unlock` in application code. Wrap them.
- Rule of zero: if members are all RAII types (`string`, `vector`, `unique_ptr`), write no destructor or copy/move operations.
- Rule of five: if you write one of destructor / copy ctor / copy assign / move ctor / move assign, decide on all five (`= default` or `= delete` counts).

## Smart pointers

| Type | Ownership | Use when |
|---|---|---|
| `std::unique_ptr<T>` | Sole, movable, not copyable | Default choice. Zero overhead. |
| `std::shared_ptr<T>` | Shared, ref-counted | Several owners genuinely outlive each other |
| `std::weak_ptr<T>` | Non-owning observer of a `shared_ptr` | Break cycles, caches, back-pointers |
| raw `T*` / `T&` | None | Borrowing only, never owning |

```cpp
#include <iostream>
#include <memory>
#include <utility>

struct Widget {
  explicit Widget(int id) : id(id) { std::cout << "Widget " << id << " created\n"; }
  ~Widget() { std::cout << "Widget " << id << " destroyed\n"; }
  int id;
};

void borrow(const Widget& w) { std::cout << "borrowed " << w.id << '\n'; }
void take(std::unique_ptr<Widget> w) { std::cout << "took " << w->id << '\n'; }

int main() {
  auto u = std::make_unique<Widget>(1);   // prefer make_unique over new
  borrow(*u);                              // lend without transferring
  take(std::move(u));                      // ownership moves; Widget 1 dies here
  std::cout << "u is " << (u ? "set" : "empty") << '\n';

  auto s1 = std::make_shared<Widget>(2);
  {
    std::shared_ptr<Widget> s2 = s1;       // count = 2
    std::cout << "use_count " << s1.use_count() << '\n';
  }
  std::cout << "use_count " << s1.use_count() << '\n';

  std::weak_ptr<Widget> w = s1;            // does not keep it alive
  if (auto locked = w.lock()) std::cout << "still alive: " << locked->id << '\n';
  s1.reset();                              // last owner gone
  std::cout << "expired: " << std::boolalpha << w.expired() << '\n';
}
```

Custom deleter, for C handles (`unique_ptr` stores it in the type, so it costs nothing extra for a stateless function):

```cpp
#include <cstdio>
#include <memory>

struct FileCloser {
  void operator()(std::FILE* f) const { if (f) std::fclose(f); }
};

int main() {
  std::unique_ptr<std::FILE, FileCloser> f(std::fopen("/dev/null", "w"));
  if (!f) return 1;
  std::fputs("auto-closed\n", f.get());
}
```

Arrays: prefer `std::vector` or `std::array`. If you must, `std::unique_ptr<int[]> p(new int[n]);` uses `delete[]` correctly.

## std::string

```cpp
#include <iostream>
#include <sstream>
#include <string>
#include <string_view>
#include <vector>

// string_view: cheap read-only view. Never store one past its source's lifetime.
bool starts_with(std::string_view s, std::string_view prefix) {
  return s.substr(0, prefix.size()) == prefix;
}

int main() {
  std::string s = "hello";
  s += ", world";
  s.append("!");
  std::cout << s << " len=" << s.size() << '\n';

  auto pos = s.find("world");
  if (pos != std::string::npos) std::cout << "found at " << pos << " -> " << s.substr(pos, 5) << '\n';

  s.replace(0, 5, "HELLO");
  s.insert(s.size(), "?");
  std::cout << s << '\n';

  // Number conversions throw on bad input
  std::cout << std::stoi("42") << ' ' << std::to_string(3.5) << '\n';
  try {
    std::stoi("abc");
  } catch (const std::invalid_argument&) {
    std::cout << "stoi: invalid_argument\n";
  }

  // Split on a delimiter
  std::vector<std::string> parts;
  std::istringstream in("a,b,,c");
  for (std::string tok; std::getline(in, tok, ',');) parts.push_back(tok);
  std::cout << parts.size() << " parts\n";

  std::cout << std::boolalpha << starts_with("filesystem", "file") << '\n';
  const char* c = s.c_str();  // valid until s is modified or destroyed
  std::cout << c << '\n';
}
```

Notes:
- `s[i]` is unchecked, `s.at(i)` throws `std::out_of_range`.
- `==`, `<`, `+` work on `std::string`. Never compare `char*` with `==` (that compares addresses).
- `std::string` is not null-safe: `std::string(nullptr)` is undefined behavior.

## std::vector

```cpp
#include <algorithm>
#include <iostream>
#include <stdexcept>
#include <string>
#include <vector>

int main() {
  std::vector<int> v = {5, 3, 8, 1};
  v.push_back(9);
  v.emplace_back(2);                        // constructs in place
  std::sort(v.begin(), v.end());
  for (int x : v) std::cout << x << ' ';
  std::cout << "\n";

  // Erase-remove idiom (C++17 has no std::erase_if)
  v.erase(std::remove_if(v.begin(), v.end(), [](int x) { return x % 2 == 0; }), v.end());
  std::cout << "odds: " << v.size() << '\n';

  auto it = std::find(v.begin(), v.end(), 5);
  if (it != v.end()) std::cout << "5 at index " << (it - v.begin()) << '\n';

  std::vector<std::string> names;
  names.reserve(100);                       // avoid reallocations when size is known
  names.push_back("ada");
  std::cout << "size " << names.size() << " cap>=100 " << (names.capacity() >= 100) << '\n';

  try {
    std::cout << v.at(99) << '\n';          // checked access
  } catch (const std::out_of_range& e) {
    std::cout << "out_of_range caught\n";
  }

  std::vector<std::vector<int>> grid(3, std::vector<int>(4, 0));
  grid[1][2] = 7;
  std::cout << "grid[1][2]=" << grid[1][2] << '\n';
}
```

Notes:
- Pass by `const std::vector<T>&` to read, by value then `std::move` to sink.
- `push_back`/`insert`/`reserve` can reallocate: all iterators, pointers, and references into the vector become invalid.
- Loop with `for (const auto& x : v)` to avoid copying large elements.
- Use `std::size_t` (or `v.size()`'s type) for indices, not `int`, to avoid sign-compare warnings.

## Common mistakes

| Mistake | Why it hurts | Fix |
|---|---|---|
| `new`/`delete` by hand | Leaks on early return or exception | `make_unique` / `make_shared` / containers |
| Copying a `unique_ptr` | Compile error: sole ownership | `std::move`, or pass `*p` / `p.get()` to borrow |
| `shared_ptr` cycles (A holds B holds A) | Never freed | Make the back-link a `weak_ptr` |
| Two `shared_ptr`s built from the same raw pointer | Double free | Create once with `make_shared`; copy the `shared_ptr` |
| Using an object after `std::move` | Valid but unspecified state | Reassign or don't touch it |
| `string_view` over a temporary string | Dangles after the full expression | Keep the owning `std::string` alive |
| `v[i]` with bad `i` | Silent UB | `v.at(i)` or check `i < v.size()` |
| Holding `v[0]` / iterator across `push_back` | Dangling after reallocation | Re-fetch, or `reserve` first |
| `erase` inside a range-for | Invalidates the loop iterator | Erase-remove idiom |
| `for (int i = 0; i < v.size(); ++i)` | Signed/unsigned warning | `std::size_t i`, or range-for |
| `const char* p = (a + b).c_str();` | Temporary dies at `;` | Store the `std::string` in a variable |
| Forgetting `explicit` on single-arg constructors | Surprise implicit conversions | Mark them `explicit` |
