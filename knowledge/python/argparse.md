# argparse cheat sheet (Python 3.10+)

Standard library CLI parsing. Gives you `--help`, type conversion, and error messages for free.

## Minimal parser

Passing a list to `parse_args()` makes examples testable. In a real script, call `parse_args()` with no argument so it reads `sys.argv`.

```python
import argparse

parser = argparse.ArgumentParser(description="Greet someone")
parser.add_argument("name")                          # positional, required
parser.add_argument("-c", "--count", type=int, default=1)
parser.add_argument("-v", "--verbose", action="store_true")

args = parser.parse_args(["Ada", "-c", "2", "-v"])
print(args.name, args.count, args.verbose)
assert (args.name, args.count, args.verbose) == ("Ada", 2, True)
```

## Common argument options

```python
import argparse

p = argparse.ArgumentParser()
p.add_argument("--port", type=int, default=8080, help="TCP port (default: %(default)s)")
p.add_argument("--mode", choices=["fast", "slow"], default="fast")
p.add_argument("--tag", action="append", default=[], help="repeatable")
p.add_argument("files", nargs="*", help="zero or more files")
p.add_argument("--level", action="count", default=0, help="-l, -ll, ...")
p.add_argument("--debug", action=argparse.BooleanOptionalAction, default=False)  # --debug / --no-debug

a = p.parse_args(["--tag", "a", "--tag", "b", "--no-debug", "x.txt", "y.txt"])
print(a)
assert a.tag == ["a", "b"] and a.debug is False and a.files == ["x.txt", "y.txt"]
```

## Subcommands (like `git commit`)

```python
import argparse

parser = argparse.ArgumentParser(prog="tool")
sub = parser.add_subparsers(dest="command", required=True)

scan = sub.add_parser("scan", help="scan a host")
scan.add_argument("host")
scan.add_argument("--ports", default="1-1024")

hash_ = sub.add_parser("hash", help="hash a file")
hash_.add_argument("path")

args = parser.parse_args(["scan", "127.0.0.1", "--ports", "20-80"])
print(args.command, args.host, args.ports)
assert args.command == "scan"
```

## Custom type validators

Raise `argparse.ArgumentTypeError` and argparse prints a clean usage error.

```python
import argparse
import sys
from pathlib import Path

def existing_file(value: str) -> Path:
    path = Path(value)
    if not path.is_file():
        raise argparse.ArgumentTypeError(f"not a file: {value}")
    return path

def port(value: str) -> int:
    n = int(value)
    if not 1 <= n <= 65535:
        raise argparse.ArgumentTypeError(f"port out of range: {n}")
    return n

p = argparse.ArgumentParser(exit_on_error=False)
p.add_argument("--port", type=port)
p.add_argument("--file", type=existing_file)
print(p.parse_args(["--port", "443", "--file", sys.executable]))
try:
    p.parse_args(["--port", "99999"])
except argparse.ArgumentError as e:
    print("caught:", e)
```

## Mutually exclusive groups and mains

```python
import argparse
import sys

def main(argv: list[str] | None = None) -> int:
    parser = argparse.ArgumentParser()
    group = parser.add_mutually_exclusive_group(required=True)
    group.add_argument("--encode", action="store_true")
    group.add_argument("--decode", action="store_true")
    args = parser.parse_args(argv)
    print("encode" if args.encode else "decode")
    return 0

if __name__ == "__main__":
    sys.exit(main(["--encode"]))   # real script: sys.exit(main())
```

## Common mistakes

| Mistake | Fix |
|---|---|
| `type=bool` for flags. `bool("False")` is `True`. | Use `action="store_true"` or `BooleanOptionalAction`. |
| `action="append"` with a non-empty default: user values are added after the default items. | Use `default=None` and fall back in code if you want user values to replace the default. |
| Using `args.some-flag`. | Dashes become underscores: `--some-flag` is `args.some_flag`. |
| Forgetting `dest="command"` on subparsers, so you cannot tell which ran. | `add_subparsers(dest="command", required=True)`. |
| `nargs="?"` positional followed by other positionals. | Ambiguous. Use options (`--name`) for optional values. |
| Calling `parse_args()` at import time. | Put it inside `main()` so the module is importable and testable. |
| A value starting with `-` (like `-5`) parsed as a flag. | Use `--num=-5`, or `--` before positionals. |
| `type=open`. File handles never get closed. | Use `type=Path`, then `with path.open()`. Or `argparse.FileType` only for tiny scripts. |
| Calling `sys.exit` inside helpers. | argparse exits on bad input with code 2. Catch `SystemExit` in tests. |

## Handy extras

- `parser.print_help()` prints the help text.
- `parse_known_args()` returns `(namespace, leftover_list)`, good for wrappers.
- `formatter_class=argparse.ArgumentDefaultsHelpFormatter` appends defaults to help text.
- `vars(args)` converts the namespace to a dict.
- `parser.error("message")` prints usage and exits with code 2.
- Environment variable fallback: `default=os.environ.get("API_TOKEN")`. Never hardcode secrets.
