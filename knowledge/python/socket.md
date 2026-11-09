# socket cheat sheet (Python 3.10+)

Low level networking from the standard library. Examples here only talk to `127.0.0.1`. Only connect to or scan systems you own or have written permission to test.

## Quick reference

| Call | Purpose |
|---|---|
| `socket.socket(AF_INET, SOCK_STREAM)` | TCP over IPv4 |
| `socket.socket(AF_INET, SOCK_DGRAM)` | UDP over IPv4 |
| `s.settimeout(sec)` | Timeout for connect/recv/accept |
| `s.connect((host, port))` | Client connect |
| `s.bind((host, port))`, `s.listen()`, `s.accept()` | Server side |
| `s.sendall(data)` | Send all bytes (prefer over `send`) |
| `s.recv(n)` | Read up to n bytes, `b""` means peer closed |
| `socket.create_connection((h, p), timeout=3)` | Connect with DNS and IPv4/6 handled |
| `socket.gethostbyname(name)` | DNS lookup, IPv4 |
| `socket.getaddrinfo(...)` | Full lookup, IPv4 and IPv6 |

## TCP echo server and client (one script)

Port `0` asks the OS for a free port.

```python
import socket
import threading

def echo_server(ready: threading.Event, port_holder: list[int]) -> None:
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as srv:
        srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        srv.bind(("127.0.0.1", 0))
        srv.listen()
        port_holder.append(srv.getsockname()[1])
        ready.set()
        conn, addr = srv.accept()
        with conn:
            while data := conn.recv(1024):      # b"" means client closed
                conn.sendall(data.upper())

ready = threading.Event()
port: list[int] = []
t = threading.Thread(target=echo_server, args=(ready, port), daemon=True)
t.start()
ready.wait(5)

with socket.create_connection(("127.0.0.1", port[0]), timeout=3) as c:
    c.sendall(b"hello")
    reply = c.recv(1024)
    print(reply)
    assert reply == b"HELLO"
t.join(2)
```

## Message framing (TCP is a stream, not messages)

`recv(n)` can return fewer bytes than sent, or two messages glued together. Use a length prefix or a delimiter.

```python
import socket
import struct
import threading

def recv_exact(sock: socket.socket, n: int) -> bytes:
    buf = bytearray()
    while len(buf) < n:
        chunk = sock.recv(n - len(buf))
        if not chunk:
            raise ConnectionError("peer closed mid-message")
        buf.extend(chunk)
    return bytes(buf)

def send_msg(sock: socket.socket, payload: bytes) -> None:
    sock.sendall(struct.pack("!I", len(payload)) + payload)   # 4-byte big-endian length

def recv_msg(sock: socket.socket) -> bytes:
    (length,) = struct.unpack("!I", recv_exact(sock, 4))
    if length > 1_000_000:                                    # sanity limit
        raise ValueError("message too large")
    return recv_exact(sock, length)

a, b = socket.socketpair()          # connected pair, no ports needed
threading.Thread(target=lambda: (send_msg(a, b"one"), send_msg(a, b"two")), daemon=True).start()
print(recv_msg(b), recv_msg(b))
a.close(); b.close()
```

## TCP port check with proper errors

```python
import socket
import threading

def check_port(host: str, port: int, timeout: float = 1.0) -> str:
    try:
        with socket.create_connection((host, port), timeout=timeout):
            return "open"
    except ConnectionRefusedError:
        return "closed"
    except (TimeoutError, socket.timeout):
        return "filtered/timeout"
    except socket.gaierror:
        return "dns failure"
    except OSError as e:
        return f"error: {e.strerror}"

with socket.socket() as srv:                       # temporary local listener
    srv.bind(("127.0.0.1", 0))
    srv.listen()
    open_port = srv.getsockname()[1]
    print(open_port, check_port("127.0.0.1", open_port))

with socket.socket() as tmp:                        # grab a free port, then release it
    tmp.bind(("127.0.0.1", 0))
    closed_port = tmp.getsockname()[1]
print(closed_port, check_port("127.0.0.1", closed_port))
print(check_port("no-such-host.invalid", 80))
```

## UDP

```python
import socket

server = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
server.bind(("127.0.0.1", 0))
server.settimeout(2)
addr = server.getsockname()

client = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
client.sendto(b"ping", addr)                 # no connect, no handshake

data, sender = server.recvfrom(1024)
server.sendto(b"pong", sender)
client.settimeout(2)
print(data, client.recvfrom(1024)[0])
server.close(); client.close()
```

## DNS

```python
import socket

print(socket.gethostbyname("localhost"))
for family, type_, proto, canon, sockaddr in socket.getaddrinfo(
    "localhost", 80, proto=socket.IPPROTO_TCP
):
    print(family.name, sockaddr)
print(socket.gethostname())
```

## Common mistakes

| Mistake | Fix |
|---|---|
| No timeout, so `connect` or `recv` can hang for minutes. | `settimeout()` or `create_connection(..., timeout=)`. |
| Sending `str`. Raises `TypeError`. | Encode: `s.sendall("text".encode())`. Decode on receive. |
| Assuming one `send` equals one `recv`. | Frame messages (length prefix or delimiter). Loop until complete. |
| `send()` instead of `sendall()`. It may send only part. | Use `sendall()`. |
| Ignoring `recv` returning `b""`. Infinite loop. | Treat it as "peer closed" and break. |
| `OSError: Address already in use` on restart. | `setsockopt(SOL_SOCKET, SO_REUSEADDR, 1)` before `bind`. |
| Binding to `0.0.0.0` for a local test. It exposes the service to your network. | Bind `127.0.0.1` unless remote access is needed. |
| Not closing sockets. | Use `with socket...` blocks. |
| Catching only `socket.error`. | It is an alias of `OSError`. Catch `ConnectionRefusedError`, `TimeoutError`, `socket.gaierror` first. |
| Fixed port in tests, collisions with other programs. | Bind port `0`, read `getsockname()[1]`. |
| Trusting data from the network (huge lengths, bad encodings). | Cap sizes, validate, decode with `errors="replace"` when logging. |
| Blocking `accept()` serving one client at a time. | Use `threading`, `selectors`, or `asyncio` (`asyncio.start_server`). |
