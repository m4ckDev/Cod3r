# logging cheat sheet (Python 3.10+)

Use `logging` instead of `print` for anything beyond a throwaway script. It gives levels, timestamps, file output, and easy on/off control.

## Levels

| Level | Value | Use for |
|---|---|---|
| `DEBUG` | 10 | Detail for diagnosing problems |
| `INFO` | 20 | Normal progress ("started scan") |
| `WARNING` | 30 | Something odd, program continues (default threshold) |
| `ERROR` | 40 | An operation failed |
| `CRITICAL` | 50 | The program cannot continue |

## Quick start

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)-8s %(name)s: %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)
log = logging.getLogger(__name__)

log.debug("hidden, below INFO")
log.info("starting, user=%s", "ada")        # lazy %-style formatting
log.warning("disk at %d%%", 91)
log.error("failed to open %s", "/nope")
```

## Log to file and console, different levels

```python
import logging
import sys
import tempfile
from pathlib import Path

def setup_logging(logfile: Path, verbose: bool = False) -> None:
    root = logging.getLogger()
    root.setLevel(logging.DEBUG)
    root.handlers.clear()                        # avoid duplicate handlers on re-run

    fmt = logging.Formatter("%(asctime)s %(levelname)s %(name)s: %(message)s")

    console = logging.StreamHandler(sys.stderr)
    console.setLevel(logging.DEBUG if verbose else logging.INFO)
    console.setFormatter(fmt)

    file_handler = logging.FileHandler(logfile, encoding="utf-8")
    file_handler.setLevel(logging.DEBUG)
    file_handler.setFormatter(fmt)

    root.addHandler(console)
    root.addHandler(file_handler)

with tempfile.TemporaryDirectory() as tmp:
    path = Path(tmp) / "app.log"
    setup_logging(path, verbose=False)
    log = logging.getLogger("demo")
    log.debug("only in the file")
    log.info("in both")
    logging.shutdown()                           # flush and close handlers
    print(path.read_text(encoding="utf-8"))
```

## Logging exceptions

```python
import logging

logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")
log = logging.getLogger(__name__)

try:
    1 / 0
except ZeroDivisionError:
    log.exception("math failed")        # ERROR + full traceback, use inside except
try:
    int("abc")
except ValueError as exc:
    log.error("bad input: %s", exc, exc_info=True)   # same idea, any level
```

## Rotating files

```python
import logging
import tempfile
from logging.handlers import RotatingFileHandler
from pathlib import Path

with tempfile.TemporaryDirectory() as tmp:
    handler = RotatingFileHandler(
        Path(tmp) / "app.log", maxBytes=200, backupCount=2, encoding="utf-8"
    )
    log = logging.getLogger("rotating")
    log.setLevel(logging.INFO)
    log.addHandler(handler)
    for i in range(30):
        log.info("line number %d", i)
    handler.close()
    print(sorted(p.name for p in Path(tmp).iterdir()))   # app.log, app.log.1, app.log.2
```
(`TimedRotatingFileHandler(when="midnight")` rotates by time instead.)

## Wiring to argparse

```python
import argparse
import logging

parser = argparse.ArgumentParser()
parser.add_argument("-v", "--verbose", action="count", default=0)
args = parser.parse_args(["-vv"])

level = {0: logging.WARNING, 1: logging.INFO}.get(args.verbose, logging.DEBUG)
logging.basicConfig(level=level, format="%(levelname)s: %(message)s", force=True)
logging.getLogger("tool").debug("visible because -vv")
```

## Libraries vs applications

- Applications configure logging once, in `main()`.
- Libraries only call `logging.getLogger(__name__)` and never configure handlers.
- Child loggers (`app.net`) propagate to parents (`app`) unless `propagate = False`.
- Quiet a noisy dependency: `logging.getLogger("urllib3").setLevel(logging.WARNING)`.

## Common mistakes

| Mistake | Fix |
|---|---|
| `log.info(f"user={user}")` formats even when disabled, and breaks structured grouping. | `log.info("user=%s", user)`. |
| `basicConfig()` does nothing the second time, since the root logger already has handlers. | Call once at startup, or pass `force=True` (3.8+). |
| Using `logging.info()` (root) everywhere. | One logger per module: `logging.getLogger(__name__)`. |
| Duplicate lines, from adding a handler on every function call. | Configure once. Check `logger.handlers` before adding. |
| `log.error(e)` loses the traceback. | `log.exception(...)` inside `except`. |
| Logging secrets: passwords, tokens, full auth headers. | Redact before logging. Log IDs, not credentials. |
| Logging untrusted input directly. Newlines can forge fake log lines. | Log with `%r` so control characters are escaped: `log.info("user=%r", name)`. |
| Log file empty when the script exits abruptly. | Call `logging.shutdown()` or use `try/finally`. |
| Default level is `WARNING`, so `info()` shows nothing. | Set `level=logging.INFO` in `basicConfig`. |
| Naive local timestamps in multi-machine logs. | Use UTC: `logging.Formatter.converter = time.gmtime`. |
