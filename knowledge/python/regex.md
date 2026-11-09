# regex cheat sheet (Python 3.10+, `re` module)

## Syntax quick reference

| Pattern | Meaning |
|---|---|
| `.` | Any character except newline |
| `\d` `\w` `\s` | Digit, word char (letters, digits, `_`), whitespace |
| `\D` `\W` `\S` | Opposites |
| `^` `$` | Start, end of string (of each line with `re.M`) |
| `\b` | Word boundary |
| `[abc]` `[^abc]` `[a-z]` | Character class, negated class, range |
| `*` `+` `?` | 0 or more, 1 or more, 0 or 1 |
| `{n}` `{n,m}` | Exactly n, between n and m |
| `*?` `+?` | Lazy (shortest) versions |
| `(...)` `(?:...)` | Capturing group, non-capturing group |
| `(?P<name>...)` | Named group |
| `a|b` | Alternation |
| `(?=...)` `(?!...)` | Lookahead, negative lookahead |
| `(?<=...)` `(?<!...)` | Lookbehind (fixed width only) |

Always use raw strings: `r"\d+"`.

## The main functions

```python
import re

text = "Order 66 shipped on 2026-11-09, order 7 pending."

print(re.search(r"\d+", text).group())          # first match anywhere: '66'
print(re.match(r"Order", text) is not None)     # must match at start: True
print(re.fullmatch(r"\d+", "123") is not None)  # whole string: True
print(re.findall(r"\d+", text))                 # ['66', '2026', '11', '09', '7']
print(re.sub(r"\d+", "#", text))                # replace
print(re.split(r"[,.]\s*", text))               # split on pattern

for m in re.finditer(r"\d+", text):             # match objects with positions
    print(m.group(), m.start(), m.end())
```

`search` returns `None` when nothing matches, so check before calling `.group()`.

## Groups and named groups

```python
import re

m = re.search(r"(?P<year>\d{4})-(?P<month>\d{2})-(?P<day>\d{2})", "on 2026-11-09 ok")
print(m.group("year"), m.group(2), m.groups())
print(m.groupdict())
assert m["year"] == "2026"

# Reuse a group in the replacement
print(re.sub(r"(\w+)@(\w+)\.com", r"\2: \1", "ada@example.com"))
print(re.sub(r"(?P<u>\w+)@", r"\g<u> at ", "ada@x"))

# Function as replacement
print(re.sub(r"\d+", lambda m: str(int(m.group()) * 2), "3 apples, 10 pears"))
```

## Compile and flags

```python
import re

LOG_LINE = re.compile(
    r"""
    ^(?P<ip>\d{1,3}(?:\.\d{1,3}){3})   # client ip
    \s+-\s+\S+\s+
    \[(?P<ts>[^\]]+)\]                  # timestamp
    \s+"(?P<method>[A-Z]+)\s+(?P<path>\S+)
    """,
    re.VERBOSE,
)
line = '203.0.113.9 - - [09/Nov/2026:10:00:01 +0000] "GET /login HTTP/1.1" 401 512'
m = LOG_LINE.match(line)
print(m.groupdict())
```

| Flag | Short | Effect |
|---|---|---|
| `re.IGNORECASE` | `re.I` | Case insensitive |
| `re.MULTILINE` | `re.M` | `^`/`$` match at each line |
| `re.DOTALL` | `re.S` | `.` also matches newline |
| `re.VERBOSE` | `re.X` | Whitespace and comments allowed in the pattern |

## Greedy vs lazy, lookarounds

```python
import re

html = "<b>one</b> and <b>two</b>"
print(re.findall(r"<b>.*</b>", html))     # greedy: one big match
print(re.findall(r"<b>.*?</b>", html))    # lazy: two matches

print(re.findall(r"\d+(?= USD)", "10 USD, 20 EUR, 30 USD"))   # ['10', '30']
print(re.findall(r"(?<=\$)\d+", "cost $15 and $40"))           # ['15', '40']
print(re.findall(r"\b(?!the\b)\w{3}\b", "the cat and the dog"))
```

## Practical patterns

```python
import re

samples = {
    "ipv4": (r"\b(?:\d{1,3}\.){3}\d{1,3}\b", "hosts 10.0.0.5 and 192.168.1.20"),
    "email": (r"[\w.+-]+@[\w-]+(?:\.[\w-]+)+", "mail ada@example.com, bob@mail.co.uk"),
    "hex": (r"\b[0-9a-fA-F]{32}\b", "md5 d41d8cd98f00b204e9800998ecf8427e end"),
    "port": (r"(?<=:)\d{2,5}\b", "listening on 0.0.0.0:8080"),
}
for name, (pattern, text) in samples.items():
    print(name, re.findall(pattern, text))

# Validate octets after matching (regex alone cannot check ranges well)
import ipaddress
for candidate in re.findall(samples["ipv4"][0], "1.2.3.4 999.1.1.1"):
    try:
        ipaddress.ip_address(candidate)
        print("valid", candidate)
    except ValueError:
        print("invalid", candidate)
```

## Common mistakes

| Mistake | Fix |
|---|---|
| Non-raw strings: `"\d"` or `"\b"` (backspace). | `r"\d"`, `r"\b"`. |
| `re.match` when you meant `search`. `match` anchors at the start only. | `search` for anywhere, `fullmatch` for the whole string. |
| `.group()` on `None`, giving `AttributeError`. | `if m := re.search(...):` then use `m`. |
| Greedy `.*` swallowing too much. | Use `.*?` or a negated class like `[^"]*`. |
| Unescaped special characters: `re.search("1.5", "135")` matches. | `re.escape(user_text)` for literal text. |
| `findall` with groups returns only the group contents. | Use `(?:...)` for non-capturing, or `finditer`. |
| Parsing HTML, JSON, or nested structures with regex. | Use `html.parser`, `json`, or a real parser. |
| Validating emails or IPs with one giant regex. | Loose regex first, then `ipaddress` or `email.utils`. |
| Catastrophic backtracking: `(a+)+$` on a long non-matching string. | Avoid nested quantifiers. Never run untrusted patterns on untrusted input without limits. |
| Using `$` to mean "end of string". It also matches before a trailing newline. | Use `\Z`, or `fullmatch`. |
| Compiling the same pattern inside a loop. | `re.compile` once. (The `re` module caches, but a compiled name documents intent.) |
| Lookbehind with variable width: `(?<=a+)`. | Python requires fixed width. Use a capture group instead. |
