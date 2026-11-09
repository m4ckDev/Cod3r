# subprocess cheat sheet (Python 3.10+)

Run external programs. Use `subprocess.run()` for almost everything.

## The basic call

```python
import subprocess
import sys

result = subprocess.run(
    [sys.executable, "-c", "print('hi')"],   # list of args, no shell
    capture_output=True,
    text=True,           # str instead of bytes
    timeout=10,          # always set one
    check=False,         # True raises CalledProcessError on non-zero exit
)
print(result.returncode, repr(result.stdout), repr(result.stderr))
assert result.returncode == 0 and result.stdout == "hi\n"
```

`CompletedProcess` has `.args`, `.returncode`, `.stdout`, `.stderr`.

## Error handling

```python
import subprocess
import sys

def run(cmd: list[str], timeout: float = 10) -> str:
    try:
        proc = subprocess.run(
            cmd, capture_output=True, text=True, timeout=timeout, check=True
        )
    except FileNotFoundError:
        raise SystemExit(f"command not found: {cmd[0]}")
    except subprocess.TimeoutExpired:
        raise SystemExit(f"timed out after {timeout}s: {cmd}")
    except subprocess.CalledProcessError as e:
        raise SystemExit(f"exit {e.returncode}: {e.stderr.strip()}")
    return proc.stdout

print(run([sys.executable, "--version"]).strip())
try:
    run([sys.executable, "-c", "import sys; sys.exit(3)"])
except SystemExit as e:
    print("caught:", e)
try:
    run(["definitely-not-a-command"])
except SystemExit as e:
    print("caught:", e)
```

## Passing input and merging output

```python
import subprocess
import sys

# Feed stdin
r = subprocess.run(
    [sys.executable, "-c", "import sys; print(sys.stdin.read().upper())"],
    input="hello", capture_output=True, text=True, timeout=10,
)
print(r.stdout.strip())

# stderr merged into stdout
r = subprocess.run(
    [sys.executable, "-c", "import sys; print('out'); print('err', file=sys.stderr)"],
    stdout=subprocess.PIPE, stderr=subprocess.STDOUT, text=True, timeout=10,
)
print(r.stdout.splitlines())

# Throw output away
subprocess.run([sys.executable, "-c", "print('x')"], stdout=subprocess.DEVNULL, timeout=10)
```

## Environment and working directory

```python
import os
import subprocess
import sys
import tempfile

env = {**os.environ, "GREETING": "hello"}        # copy, then add
with tempfile.TemporaryDirectory() as tmp:
    r = subprocess.run(
        [sys.executable, "-c", "import os; print(os.environ['GREETING'], os.getcwd())"],
        env=env, cwd=tmp, capture_output=True, text=True, timeout=10,
    )
    print(r.stdout.strip())
```

## Pipes by hand (no shell)

```python
import subprocess
import sys

producer = subprocess.Popen(
    [sys.executable, "-c", "print('b\\na\\nc')"], stdout=subprocess.PIPE, text=True
)
consumer = subprocess.run(
    [sys.executable, "-c", "import sys; print(sorted(sys.stdin.read().split()))"],
    stdin=producer.stdout, capture_output=True, text=True, timeout=10,
)
producer.stdout.close()
producer.wait()
print(consumer.stdout.strip())
```

## Streaming output line by line

```python
import subprocess
import sys

code = "import time\nfor i in range(3):\n    print('tick', i, flush=True)\n    time.sleep(0.1)"
with subprocess.Popen(
    [sys.executable, "-c", code], stdout=subprocess.PIPE, text=True, bufsize=1
) as proc:
    for line in proc.stdout:
        print("got:", line.rstrip())
    print("exit:", proc.wait())
```

## Shell vs list: the security rule

```python
import shlex
import subprocess
import sys

user_value = "file.txt; echo INJECTED"

# BAD: shell=True with interpolated input runs the second command
#   subprocess.run(f"echo {user_value}", shell=True)

# GOOD: list form, the value is one literal argument
r = subprocess.run(["echo", user_value], capture_output=True, text=True, timeout=10)
print(r.stdout.strip())

# If you must use shell=True, quote the input
r = subprocess.run(f"echo {shlex.quote(user_value)}", shell=True,
                   capture_output=True, text=True, timeout=10)
print(r.stdout.strip())

# Split a trusted command string safely
print(shlex.split("ls -l '/tmp/my dir'"))
```

## OS notes

- Linux: `["ls", "-l"]`, `["ip", "addr"]`, `["systemctl", "status", "ssh"]`.
- Windows: `dir`, `echo` and other built-ins are shell commands, so you need `shell=True` or `["cmd", "/c", "dir"]`. Real executables like `ipconfig` and `ping` work in list form. PowerShell: `["powershell", "-NoProfile", "-Command", "Get-Process"]`.
- Use `shutil.which("nmap")` to check that a tool exists before running it.

## Common mistakes

| Mistake | Fix |
|---|---|
| `subprocess.run("ls -l /tmp")` fails with `FileNotFoundError`. The whole string is treated as a program name. | `["ls", "-l", "/tmp"]` or `shlex.split(...)`. |
| `shell=True` with user input is command injection. | List form, no shell. Validate input. |
| No `timeout`, so a hung child hangs your script. | Always set `timeout=`. Catch `TimeoutExpired`. |
| Ignoring failures. `run` does not raise by default. | `check=True`, or check `returncode`. |
| Getting bytes and calling string methods. | `text=True` (or `encoding="utf-8"`). |
| `stdout=PIPE` with `Popen`, never reading, then `wait()`. Deadlock when the pipe fills. | Use `communicate()` or `run(capture_output=True)`. |
| Passing a relative executable name and expecting the current dir to be searched. | Use a full path, or `./tool` on Linux. |
| `cmd = ["ls", "-l /tmp"]` as one arg. | One list item per argument: `["ls", "-l", "/tmp"]`. |
| `os.system()` or `os.popen()`. | Replace with `subprocess.run`. |
| Output buffered when streaming from a Python child. | Child needs `flush=True` or `python -u`. |
| Unhandled `FileNotFoundError` when the program is not installed. | Catch it, or check with `shutil.which`. |
