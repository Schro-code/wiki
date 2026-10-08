---
domain: system-design
tags: [system-design]
sources: []
updated: 2026-09-10
---

# File Descriptors

## What it is

A file descriptor (fd) is a number the OS gives your process as a reference to something the kernel is managing for you — a file, a socket, a pipe. You never touch the real resource; you use the number and the OS does the work.

## What problem it solves

Processes are isolated — they can't touch hardware, other processes' memory, or network interfaces directly. The OS is the only one allowed to do that. File descriptors are the contract: you ask the OS to create or open something, the OS does it and hands you back a number to refer to it later.

## Notes

### Why the server needs to ask the OS

A process cannot just "be reachable" on its own. Only the OS controls communication endpoints. The server has to tell the OS: "create a mailbox for me at this address, and let me know when someone arrives."

```
Server:  "OS, create me a socket at /tmp/app.sock"
OS:      creates it
OS:      "Done. Your reference number is fd=3"
Server:  saves fd=3, waits
```

The server isn't connecting to anyone. It's registering itself as reachable at an address.

### The front door vs. the private room

This is the key insight for understanding how multiple clients work:

- `/tmp/app.sock` (`fd=3`) is the **front door** — public, always open, any client can knock on it
- When a client knocks, `accept()` opens a **private room** — a new dedicated fd just for that client

```
/tmp/app.sock (fd=3)  ←── client A knocks  →  accept() → fd=4  (private, A only)
                      ←── client B knocks  →  accept() → fd=5  (private, B only)
                      ←── client C knocks  →  accept() → fd=6  (private, C only)
```

The front door (`fd=3`) never closes — it keeps receiving new clients. The private rooms (`fd=4`, `fd=5`, `fd=6`) are what consume fds, one per connected client.

### The full connection flow

```
SERVER                    OS                      CLIENT A
──────                    ──                      ────────
socket()         →   creates mailbox at
                     /tmp/app.sock, gives fd=3

listen(fd=3)     →   "watch for arrivals"

                                          connect("/tmp/app.sock")
                                          OS links them, queues it

accept(fd=3)     →   "give me next arrival"
                 ←   here's fd=4 (your private line to A)

write(fd=4, …)   →   copies into A's buffer  →  client reads it
read(fd=4)       ←   reads A's buffer        ←  client wrote it

close(fd=4)      →   private room closed, fd=4 freed
```

### The per-process fd table

The kernel keeps a table per process — an array where each index points to a kernel object:

```
Process fd table
─────────────────────────────────
fd 0  →  stdin  (keyboard)
fd 1  →  stdout (terminal)
fd 2  →  stderr (terminal)
fd 3  →  listening socket (/tmp/app.sock)
fd 4  →  private connection to client A
fd 5  →  private connection to client B
...
```

`0`, `1`, `2` are always pre-assigned. Every new thing you open or accept gets the next free slot.

The fd number (like `3`) is only meaningful inside that process's table. Two different processes can both have an `fd 3` pointing to completely different kernel objects.

### Everything is a file

Unix uses the same fd mechanism for every resource type:

| What you open | You get |
|---|---|
| A file on disk | fd |
| A TCP socket | fd |
| A Unix domain socket | fd |
| A pipe | fd |
| A device (`/dev/null`) | fd |

The same calls work on all of them: `read()`, `write()`, `close()`.

### Limits

The fd table has a size limit. This is why "number of connections" and "number of open fds" are the same question.

| Limit | Scope | Default | How to check |
|---|---|---|---|
| Soft fd limit | Per process | 1024 | `ulimit -n` |
| Hard fd limit | Per process ceiling | 4096–1M | `ulimit -Hn` |
| System-wide limit | All processes combined | Millions | `cat /proc/sys/fs/file-max` |

When the table is full, `accept()` fails and new connections are refused. Raise it:

```bash
ulimit -n 100000
```

The limit that matters is the number of private rooms (`fd=4`, `fd=5`...) — one per simultaneous client. The front door only uses one fd regardless of how many clients connect.

## Relations

- [[ipc-transports]] — every IPC mechanism (pipe, socket, Unix domain socket) is represented as an fd
- [[grpc]] — gRPC connections over TCP each consume an fd on both client and server

## Tags

#system-design
