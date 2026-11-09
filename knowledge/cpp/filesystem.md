# std::filesystem cheat sheet (C++17)

Header `<filesystem>`, namespace `std::filesystem` (alias it: `namespace fs = std::filesystem;`). With GCC 9+ no extra link flag is needed.

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic file.cpp -o file && ./file
```

## Error handling: two overloads for most functions

| Style | Behavior |
|---|---|
| `fs::exists(p)` | Throws `fs::filesystem_error` on failure |
| `fs::exists(p, ec)` | Sets `std::error_code ec`, never throws |

Use the `ec` form in loops and anywhere failure is expected (permissions, races).

## Paths

`fs::path` is pure string manipulation. It does not touch the disk.

```cpp
#include <filesystem>
#include <iostream>

namespace fs = std::filesystem;

int main() {
  fs::path p = "/home/user/docs/report.final.txt";
  std::cout << "filename:    " << p.filename() << '\n';        // "report.final.txt"
  std::cout << "stem:        " << p.stem() << '\n';            // "report.final"
  std::cout << "extension:   " << p.extension() << '\n';       // ".txt"
  std::cout << "parent_path: " << p.parent_path() << '\n';     // "/home/user/docs"

  fs::path q = fs::path("/data") / "logs" / "app.log";         // operator/ joins with '/'
  std::cout << q << '\n';
  q.replace_extension(".gz");
  std::cout << q << '\n';

  // Gotcha: joining with an absolute path REPLACES the left side
  std::cout << (fs::path("/a/b") / "/etc/passwd") << '\n';     // "/etc/passwd"

  std::cout << "absolute? " << std::boolalpha << p.is_absolute() << '\n';
  std::cout << fs::path("a/./b/../c").lexically_normal() << '\n';  // "a/c"
  std::cout << "string(): " << p.string().size() << " chars\n";
}
```

## Query, create, and remove

```cpp
#include <filesystem>
#include <fstream>
#include <iostream>

namespace fs = std::filesystem;

int main() {
  const fs::path root = fs::temp_directory_path() / "cheatsheet_fs_demo";
  std::error_code ec;
  fs::remove_all(root, ec);                    // start clean

  fs::create_directories(root / "sub" / "deep");   // like mkdir -p
  std::ofstream(root / "a.txt") << "hello\n";
  std::ofstream(root / "sub" / "b.log") << "0123456789";

  std::cout << std::boolalpha;
  std::cout << "exists a.txt: " << fs::exists(root / "a.txt") << '\n';
  std::cout << "is_directory sub: " << fs::is_directory(root / "sub") << '\n';
  std::cout << "is_regular_file a.txt: " << fs::is_regular_file(root / "a.txt") << '\n';
  std::cout << "size b.log: " << fs::file_size(root / "sub" / "b.log") << '\n';

  // Non-throwing form
  auto n = fs::file_size(root / "missing.txt", ec);
  if (ec) std::cout << "error: " << ec.message() << '\n';
  (void)n;

  // create_directory returns false (no error) if it already exists
  std::cout << "created again: " << fs::create_directory(root / "sub") << '\n';

  fs::remove(root / "a.txt");                  // one file or empty dir
  std::cout << "removed count: " << fs::remove_all(root) << '\n';  // recursive
  std::cout << "gone: " << !fs::exists(root) << '\n';
}
```

## Iterate directories

`directory_iterator` lists one level. `recursive_directory_iterator` walks the whole tree. Order is unspecified, so sort if you need stable output.

```cpp
#include <algorithm>
#include <filesystem>
#include <fstream>
#include <iostream>
#include <string>
#include <vector>

namespace fs = std::filesystem;

int main() {
  const fs::path root = fs::temp_directory_path() / "cheatsheet_fs_walk";
  fs::remove_all(root);
  fs::create_directories(root / "src");
  std::ofstream(root / "main.cpp") << "int main(){}";
  std::ofstream(root / "src" / "util.cpp") << "// util";
  std::ofstream(root / "src" / "util.h") << "// header";
  std::ofstream(root / "README.md") << "# readme";

  std::vector<std::string> found;
  std::uintmax_t total = 0;
  for (const auto& entry : fs::recursive_directory_iterator(root)) {
    if (entry.is_regular_file()) {
      total += entry.file_size();
      if (entry.path().extension() == ".cpp")
        found.push_back(fs::relative(entry.path(), root).string());
    }
  }
  std::sort(found.begin(), found.end());
  for (const auto& f : found) std::cout << "cpp: " << f << '\n';
  std::cout << "total bytes: " << total << '\n';

  // Skip unreadable directories instead of throwing
  std::error_code ec;
  for (fs::recursive_directory_iterator it(root, fs::directory_options::skip_permission_denied, ec), end;
       it != end; it.increment(ec)) {
    if (ec) break;
  }
  fs::remove_all(root);
}
```

## Copy, rename, and metadata

```cpp
#include <chrono>
#include <filesystem>
#include <fstream>
#include <iostream>

namespace fs = std::filesystem;

int main() {
  const fs::path root = fs::temp_directory_path() / "cheatsheet_fs_copy";
  fs::remove_all(root);
  fs::create_directories(root / "dir");
  std::ofstream(root / "orig.txt") << "data";

  fs::copy_file(root / "orig.txt", root / "copy.txt");
  try {
    fs::copy_file(root / "orig.txt", root / "copy.txt");   // already exists -> throws
  } catch (const fs::filesystem_error& e) {
    std::cout << "copy failed as expected: " << e.code().message() << '\n';
  }
  fs::copy_file(root / "orig.txt", root / "copy.txt", fs::copy_options::overwrite_existing);

  fs::rename(root / "copy.txt", root / "dir" / "moved.txt");  // also moves across dirs
  std::cout << "moved exists: " << std::boolalpha << fs::exists(root / "dir" / "moved.txt") << '\n';

  fs::copy(root / "dir", root / "dir2", fs::copy_options::recursive);  // copy a tree

  auto t = fs::last_write_time(root / "orig.txt");
  auto age = fs::file_time_type::clock::now() - t;
  std::cout << "age < 1 minute: " << (age < std::chrono::minutes(1)) << '\n';

  fs::permissions(root / "orig.txt", fs::perms::owner_read | fs::perms::owner_write,
                  fs::perm_options::replace);                 // chmod 600
  auto perms = fs::status(root / "orig.txt").permissions();
  std::cout << "owner can write: " << ((perms & fs::perms::owner_write) != fs::perms::none) << '\n';

  std::cout << "cwd is absolute: " << fs::current_path().is_absolute() << '\n';
  fs::remove_all(root);
}
```

## Symlinks and path safety

```cpp
#include <filesystem>
#include <iostream>

namespace fs = std::filesystem;

// True if `candidate` resolves to somewhere inside `base` (guards against "../" tricks).
bool is_inside(const fs::path& base, const fs::path& candidate) {
  std::error_code ec;
  auto b = fs::weakly_canonical(base, ec);
  if (ec) return false;
  auto c = fs::weakly_canonical(candidate, ec);
  if (ec) return false;
  auto rel = c.lexically_relative(b);
  return !rel.empty() && *rel.begin() != "..";
}

int main() {
  const fs::path base = fs::temp_directory_path();
  std::cout << std::boolalpha;
  std::cout << is_inside(base, base / "ok.txt") << '\n';              // true
  std::cout << is_inside(base, base / ".." / "etc" / "passwd") << '\n';  // false
}
```

`fs::canonical` requires the path to exist and resolves symlinks. `weakly_canonical` tolerates a non-existent tail. `fs::is_symlink(p)` and `fs::read_symlink(p)` inspect links without following them.

## Common mistakes

| Mistake | Why it hurts | Fix |
|---|---|---|
| `fs::path("/a") / "/b"` | Right operand is absolute, so result is `/b` | Strip the leading `/` or use `relative_path()` |
| Using throwing overloads in a scan loop | One unreadable file aborts everything | Pass `std::error_code&` |
| `exists()` then act (check-then-use) | Race: file can change in between | Just try the operation and handle the error |
| Assuming iteration order | Order is unspecified | Collect and `std::sort` |
| Deleting while iterating | Invalidates the iterator | Collect paths first, delete afterward |
| `create_directory` for nested paths | Fails if parent is missing | `create_directories` |
| `copy_file` onto an existing file | Throws by default | `copy_options::overwrite_existing` |
| Joining untrusted input to a base dir | `../` or absolute path escapes | Check with `weakly_canonical` + `lexically_relative` |
| Printing `path` with `<<` and expecting no quotes | `operator<<` quotes the path | Use `p.string()` |
| `remove_all` on a computed path | Can delete far more than intended | Validate the path first, log what you delete |
| Comparing `file_time_type` to `system_clock` | Different clocks in C++17 | Compare against `file_time_type::clock::now()` |
