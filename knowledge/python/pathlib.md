# pathlib cheat sheet (Python 3.10+)

Object-oriented paths. Works on Windows and Linux with the same code. Prefer it over `os.path` and string concatenation.

## Building paths

```python
from pathlib import Path

home = Path.home()
p = home / "projects" / "cod3r" / "notes.txt"   # / joins, no manual separators
print(p)
print(Path.cwd(), Path("~/x").expanduser())

# Pieces
f = Path("/var/log/syslog.1.gz")
print(f.name)      # syslog.1.gz
print(f.stem)      # syslog.1
print(f.suffix)    # .gz
print(f.suffixes)  # ['.1', '.gz']
print(f.parent)    # /var/log
print(f.parts)     # ('/', 'var', 'log', 'syslog.1.gz')
assert f.suffix == ".gz" and f.parent == Path("/var/log")
```

## Changing parts (returns a new Path)

```python
from pathlib import Path

p = Path("report.txt")
print(p.with_suffix(".csv"))        # report.csv
print(p.with_name("summary.txt"))   # summary.txt
print(p.with_stem("final"))         # final.txt  (3.9+)
print(Path("a/b/c.txt").relative_to("a"))   # b/c.txt
print(Path("a/../b").resolve())     # absolute, symlinks and .. resolved
```

## Reading, writing, checking

```python
import tempfile
from pathlib import Path

with tempfile.TemporaryDirectory() as tmp:
    base = Path(tmp)
    note = base / "note.txt"

    note.write_text("hello\nworld\n", encoding="utf-8")
    print(note.read_text(encoding="utf-8").splitlines())

    note.write_bytes(b"\x00\x01")
    print(note.read_bytes())

    with note.open("a", encoding="utf-8") as fh:   # streaming / appending
        fh.write("more")

    print(note.exists(), note.is_file(), note.is_dir())
    print(note.stat().st_size, "bytes")

    sub = base / "a" / "b"
    sub.mkdir(parents=True, exist_ok=True)         # like mkdir -p
    note.rename(sub / "moved.txt")                 # move
    (sub / "moved.txt").unlink()                   # delete file
    (sub / "missing.txt").unlink(missing_ok=True)  # no error if absent
    sub.rmdir()                                    # only if empty
```

## Listing and searching

```python
import tempfile
from pathlib import Path

with tempfile.TemporaryDirectory() as tmp:
    root = Path(tmp)
    (root / "src" / "pkg").mkdir(parents=True)
    (root / "src" / "a.py").write_text("x = 1\n")
    (root / "src" / "pkg" / "b.py").write_text("y = 2\n")
    (root / "README.md").write_text("# hi\n")

    print(sorted(p.name for p in root.iterdir()))           # one level
    print(sorted(p.name for p in root.glob("*.md")))        # pattern, one level
    print(sorted(p.name for p in root.rglob("*.py")))       # recursive
    print(sorted(str(p.relative_to(root)) for p in root.glob("**/*") if p.is_file()))

    # Python 3.12+: Path.walk() like os.walk
    import sys
    if sys.version_info >= (3, 12):
        for dirpath, dirnames, filenames in root.walk():
            print(dirpath.name, filenames)
```

## Safe path handling

```python
from pathlib import Path

def safe_join(base: Path, user_value: str) -> Path:
    """Reject paths that escape base (path traversal)."""
    base = base.resolve()
    target = (base / user_value).resolve()
    if not target.is_relative_to(base):      # 3.9+
        raise ValueError(f"path escapes base: {user_value}")
    return target

base = Path("/tmp/uploads")
print(safe_join(base, "ok/file.txt"))
try:
    safe_join(base, "../../etc/passwd")
except ValueError as e:
    print("blocked:", e)
```

## Windows vs Linux

```python
from pathlib import Path, PurePosixPath, PureWindowsPath

w = PureWindowsPath(r"C:\Users\ada\notes.txt")   # works on any OS, no disk access
print(w.drive, w.parts, w.suffix)
print(PurePosixPath("/etc/hosts").as_posix())
print(w.as_posix())                              # C:/Users/ada/notes.txt
```

## Common mistakes

| Mistake | Fix |
|---|---|
| `Path("a") + "b"` raises `TypeError`. | Use `Path("a") / "b"`. |
| `"C:\new\table"` in a normal string, where `\n` and `\t` are escapes. | Use raw strings `r"C:\new\table"` or forward slashes. |
| Forgetting encoding in `read_text()`. Default is platform dependent (cp1252 on Windows). | Always pass `encoding="utf-8"`. |
| Passing a `Path` to code that wants `str` (some old libs). | `str(path)` or `os.fspath(path)`. |
| `mkdir()` raising `FileExistsError` or `FileNotFoundError`. | `mkdir(parents=True, exist_ok=True)`. |
| `rglob` through huge trees or symlink loops, or reading big files with `read_text()`. | Filter patterns narrowly. Stream big files with `open()`. |
| `startswith` on strings to check containment. `/tmp/ab` matches `/tmp/a`. | `resolve()` both and use `is_relative_to()`. |
| `Path.rename` across drives or filesystems fails. | Use `shutil.move(src, dst)`. |
| `with_suffix` on `archive.tar.gz` replaces only `.gz`. | Strip with `p.name.removesuffix(".tar.gz")` (3.9+). |
| `resolve()` assumed to verify existence. | It does not (non strict). Use `exists()` or `resolve(strict=True)`. |
