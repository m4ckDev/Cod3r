# POSIX sockets on Linux cheat sheet (C++17)

BSD sockets API for TCP. All examples bind and connect to `127.0.0.1` only, and each is a self-contained program that runs in a single process (the kernel queues a connection in the listen backlog, so the client can `connect()` before the server calls `accept()`).

```bash
g++ -std=c++17 -Wall -Wextra -Wpedantic file.cpp -o file && ./file
```

## Call sequence

| Server | Client |
|---|---|
| `socket()` | `socket()` |
| `bind()` to address:port | |
| `listen()` | |
| `accept()` returns a new fd | `connect()` |
| `recv()` / `send()` | `send()` / `recv()` |
| `close()` | `close()` |

Port `0` in `bind()` asks the kernel for a free port. Read it back with `getsockname()`.

## RAII wrapper for file descriptors

Every example reuses this. It guarantees `close()` on all paths.

```cpp
#include <unistd.h>

class Fd {
 public:
  Fd() = default;
  explicit Fd(int fd) : fd_(fd) {}
  ~Fd() { if (fd_ >= 0) ::close(fd_); }
  Fd(const Fd&) = delete;
  Fd& operator=(const Fd&) = delete;
  Fd(Fd&& o) noexcept : fd_(o.fd_) { o.fd_ = -1; }
  Fd& operator=(Fd&& o) noexcept {
    if (this != &o) { if (fd_ >= 0) ::close(fd_); fd_ = o.fd_; o.fd_ = -1; }
    return *this;
  }
  int get() const { return fd_; }
  bool valid() const { return fd_ >= 0; }
 private:
  int fd_ = -1;
};

int main() { Fd f; return f.valid() ? 1 : 0; }
```

## TCP echo over loopback

```cpp
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>

#include <cerrno>
#include <cstdint>
#include <cstring>
#include <iostream>
#include <stdexcept>
#include <string>

class Fd {
 public:
  explicit Fd(int fd) : fd_(fd) { if (fd < 0) throw std::runtime_error(std::strerror(errno)); }
  ~Fd() { ::close(fd_); }
  Fd(const Fd&) = delete;
  Fd& operator=(const Fd&) = delete;
  int get() const { return fd_; }
 private:
  int fd_;
};

sockaddr_in loopback(std::uint16_t port) {
  sockaddr_in a{};                       // zero-initialize: no stray bytes
  a.sin_family = AF_INET;
  a.sin_port = htons(port);              // host -> network byte order
  inet_pton(AF_INET, "127.0.0.1", &a.sin_addr);
  return a;
}

// send() may write fewer bytes than asked; loop until done.
void send_all(int fd, const char* data, std::size_t len) {
  while (len > 0) {
    ssize_t n = ::send(fd, data, len, MSG_NOSIGNAL);   // no SIGPIPE if peer closed
    if (n < 0) {
      if (errno == EINTR) continue;
      throw std::runtime_error(std::string("send: ") + std::strerror(errno));
    }
    data += n;
    len -= static_cast<std::size_t>(n);
  }
}

int main() {
  Fd server(::socket(AF_INET, SOCK_STREAM, 0));
  int yes = 1;
  setsockopt(server.get(), SOL_SOCKET, SO_REUSEADDR, &yes, sizeof yes);

  sockaddr_in addr = loopback(0);        // port 0 = pick any free port
  if (bind(server.get(), reinterpret_cast<sockaddr*>(&addr), sizeof addr) < 0)
    throw std::runtime_error(std::strerror(errno));
  if (listen(server.get(), 8) < 0) throw std::runtime_error(std::strerror(errno));

  socklen_t len = sizeof addr;
  getsockname(server.get(), reinterpret_cast<sockaddr*>(&addr), &len);
  std::uint16_t port = ntohs(addr.sin_port);
  std::cout << "listening on 127.0.0.1:" << port << '\n';

  Fd client(::socket(AF_INET, SOCK_STREAM, 0));
  sockaddr_in dst = loopback(port);
  if (connect(client.get(), reinterpret_cast<sockaddr*>(&dst), sizeof dst) < 0)
    throw std::runtime_error(std::strerror(errno));

  Fd conn(::accept(server.get(), nullptr, nullptr));   // server side of the connection

  const std::string msg = "hello over loopback";
  send_all(client.get(), msg.data(), msg.size());

  char buf[256];
  ssize_t n = ::recv(conn.get(), buf, sizeof buf, 0);  // 0 = peer closed, <0 = error
  if (n < 0) throw std::runtime_error(std::strerror(errno));
  std::string got(buf, static_cast<std::size_t>(n));
  std::cout << "server got: " << got << '\n';

  send_all(conn.get(), got.data(), got.size());        // echo back
  n = ::recv(client.get(), buf, sizeof buf, 0);
  std::cout << "client got: " << std::string(buf, static_cast<std::size_t>(n)) << '\n';
}
```

## Reading a full message (TCP is a byte stream)

One `send()` does not equal one `recv()`. TCP preserves order but not message boundaries. Frame messages with a length prefix or a delimiter, and loop on `recv()`.

```cpp
#include <arpa/inet.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <unistd.h>

#include <cerrno>
#include <cstdint>
#include <cstring>
#include <iostream>
#include <stdexcept>
#include <string>

// Read exactly `len` bytes or throw. Returns early only on EOF/error.
void recv_exact(int fd, char* buf, std::size_t len) {
  while (len > 0) {
    ssize_t n = ::recv(fd, buf, len, 0);
    if (n == 0) throw std::runtime_error("peer closed early");
    if (n < 0) {
      if (errno == EINTR) continue;
      throw std::runtime_error(std::strerror(errno));
    }
    buf += n;
    len -= static_cast<std::size_t>(n);
  }
}

int main() {
  int sv[2];                                            // connected pair, no ports needed
  if (socketpair(AF_UNIX, SOCK_STREAM, 0, sv) < 0) return 1;

  // Sender: 4-byte big-endian length, then payload
  std::string payload = "length-prefixed message";
  std::uint32_t be = htonl(static_cast<std::uint32_t>(payload.size()));
  ::send(sv[0], &be, sizeof be, MSG_NOSIGNAL);
  ::send(sv[0], payload.data(), payload.size(), MSG_NOSIGNAL);

  // Receiver
  std::uint32_t rbe = 0;
  recv_exact(sv[1], reinterpret_cast<char*>(&rbe), sizeof rbe);
  std::uint32_t n = ntohl(rbe);
  if (n > 1024 * 1024) throw std::runtime_error("message too large");  // never trust the peer
  std::string out(n, '\0');
  recv_exact(sv[1], &out[0], n);
  std::cout << "received " << n << " bytes: " << out << '\n';

  ::close(sv[0]);
  ::close(sv[1]);
}
```

## poll() with a timeout

```cpp
#include <arpa/inet.h>
#include <netinet/in.h>
#include <poll.h>
#include <sys/socket.h>
#include <unistd.h>

#include <iostream>

int main() {
  int s = ::socket(AF_INET, SOCK_STREAM, 0);
  sockaddr_in a{};
  a.sin_family = AF_INET;
  inet_pton(AF_INET, "127.0.0.1", &a.sin_addr);          // port 0 -> any free port
  bind(s, reinterpret_cast<sockaddr*>(&a), sizeof a);
  listen(s, 4);

  pollfd p{};
  p.fd = s;
  p.events = POLLIN;                                     // readable = a connection is waiting
  int r = poll(&p, 1, 200);                              // wait up to 200 ms
  std::cout << "poll returned " << r << (r == 0 ? " (timeout, nobody connected)\n" : "\n");
  ::close(s);
}
```

For many connections, put every fd in the `pollfd` array and handle those with `revents & POLLIN`. `epoll` scales better beyond a few thousand fds.

## Useful options

```text
setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &one, sizeof one);   // restart server without TIME_WAIT bind errors
setsockopt(fd, SOL_SOCKET, SO_RCVTIMEO, &timeval, sizeof tv); // recv() fails with EAGAIN after timeout
setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &one, sizeof one);   // disable Nagle for small latency-sensitive writes
```

Binding to `127.0.0.1` keeps a service reachable only from the local machine. `0.0.0.0` exposes it on every interface, so use it deliberately.

## Common mistakes

| Mistake | Why it hurts | Fix |
|---|---|---|
| Ignoring the return value of `send`/`recv` | Short writes and errors lost | Check every call, loop until all bytes move |
| Assuming one `recv` = one message | TCP has no message boundaries | Length prefix or delimiter, `recv_exact` |
| Forgetting `htons`/`htonl` | Wrong port or length on little-endian hosts | Convert on write, `ntohs`/`ntohl` on read |
| Not zeroing `sockaddr_in` | Garbage in padding or `sin_zero` | `sockaddr_in a{};` |
| `recv` returns 0 treated as error or ignored | 0 means orderly peer shutdown | Handle EOF explicitly |
| SIGPIPE kills the process on write to closed peer | Default signal action terminates | `MSG_NOSIGNAL`, or `signal(SIGPIPE, SIG_IGN)` |
| Fixed-size buffer plus `buf[n] = 0` when `n == sizeof buf` | Off-by-one overflow | Read `sizeof buf - 1`, or use `std::string(buf, n)` |
| Trusting a length field from the network | Huge allocation, DoS | Cap sizes before allocating |
| Leaking fds on error paths | Runs out of descriptors | RAII wrapper (`Fd`) |
| `bind` fails with `EADDRINUSE` after restart | Old socket in TIME_WAIT | `SO_REUSEADDR` before `bind` |
| Not retrying on `EINTR` | Spurious failures when signals arrive | `if (errno == EINTR) continue;` |
| Casting `sockaddr_in*` to `sockaddr*` with a C cast | Hides mistakes | `reinterpret_cast<sockaddr*>(&addr)` |
