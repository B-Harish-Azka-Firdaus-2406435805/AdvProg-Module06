# AdvProg Module 6 — Web Server Tutorial

A simple web server built in Rust, following Chapter 20 of *The Rust Programming Language*.

---

## Commit 1 Reflection Notes

### What `handle_connection` does

In Milestone 1, the server listens for incoming TCP connections on `127.0.0.1:7878`. When a browser connects, the `handle_connection` function is called with the raw `TcpStream`.

Inside `handle_connection`:

- A `BufReader` wraps the mutable stream reference. `BufReader` adds buffering on top of any type that implements `Read`, making it efficient to read line-by-line without issuing a system call per byte.
- `.lines()` returns an iterator of `Result<String>` — each item is one line of the incoming data.
- `.map(|result| result.unwrap())` unwraps each `Result`, panicking on I/O errors (acceptable for this tutorial server).
- `.take_while(|line| !line.is_empty())` stops collecting lines as soon as an empty line is encountered. An HTTP request ends its headers with a blank line (`\r\n\r\n`), so this correctly captures just the request header block.
- `.collect()` gathers the lines into a `Vec<String>`.

The result is a vector containing every header line of the HTTP request, which is then printed with the debug formatter `{:#?}` for human-readable output.

This revealed the exact HTTP request the browser sends — the request line (`GET / HTTP/1.1`), the `Host` header, `User-Agent`, `Accept`, and other standard headers. Understanding this raw format is the foundation for parsing and responding to requests in later milestones.

---

## Commit 2 Reflection Notes

### Returning HTML

In Milestone 2, `handle_connection` was extended to actually respond to the browser instead of just printing the request.

Key changes:

- `fs::read_to_string("hello.html")` reads the entire HTML file into a `String`. The path is relative to the directory where `cargo run` is executed, so `hello.html` must sit beside the binary's working directory.
- The response is built manually as a raw HTTP string. An HTTP response has the format: `status line \r\n headers \r\n\r\n body`. The `Content-Length` header tells the browser exactly how many bytes to expect in the body, preventing it from waiting indefinitely for more data.
- `stream.write_all(response.as_bytes())` writes the full response bytes to the TCP stream. `write_all` ensures the entire buffer is sent even if the OS sends it in chunks internally.

Without `Content-Length`, browsers may hang or display nothing because they don't know when the response body ends. This is why the header is essential even for a minimal server.

![Commit 2 screen capture](hello/assets/commit2.png)

---

## Commit 3 Reflection Notes

### Validating Request and Selectively Responding

In Milestone 3, the server learns to distinguish between valid and invalid routes instead of blindly serving `hello.html` for every request.

Key changes:

- Instead of collecting all headers, we now only read the **first line** of the request (`request_line`) using `.lines().next()`. This first line is the HTTP request line, e.g. `GET / HTTP/1.1`, which contains the method, path, and HTTP version — everything we need to route the request.
- A simple `if/else` checks whether the request line is exactly `"GET / HTTP/1.1"`. If it matches, we respond with `200 OK` and `hello.html`. Any other path falls through to `404 NOT FOUND` and `404.html`.
- The refactoring extracts `(status_line, filename)` as a tuple from the conditional, then reuses the same response-building and writing logic below. This avoids duplicating the `format!` and `write_all` calls for each branch — a clean separation between *deciding* the response and *sending* it.

The refactoring is important because without it, every new route would require copy-pasting the full response logic. By pulling just the varying parts (`status_line` and `filename`) into a tuple, the shared logic stays in one place and is easy to extend.

![Commit 3 screen capture](hello/assets/commit3.png)

---

## Commit 4 Reflection Notes

### Simulation of Slow Request

In Milestone 4, a `/sleep` route is added that calls `thread::sleep(Duration::from_secs(10))` before responding. This simulates a slow or expensive operation.

The critical problem this exposes: the server is **single-threaded**. The `for stream in listener.incoming()` loop processes one connection at a time — it calls `handle_connection`, blocks until it finishes, and only then picks up the next connection.

To observe this, open two browser tabs simultaneously:
1. Tab 1: `http://127.0.0.1:7878/sleep` — triggers the 10-second delay
2. Tab 2: `http://127.0.0.1:7878` — tries to load the normal page

Tab 2 is forced to wait the full 10 seconds even though it requested a fast page. In a real server with many users, one slow request would freeze everyone else. This motivates the move to a thread pool in Milestone 5.

---

## Commit 5 Reflection Notes

### Multithreaded Server using ThreadPool

In Milestone 5, the server becomes multithreaded by introducing a `ThreadPool` defined in `src/lib.rs`.

**How ThreadPool works:**

- `ThreadPool::new(size)` spawns `size` worker threads upfront and creates an `mpsc` channel. The channel's `Receiver` is wrapped in `Arc<Mutex<...>>` so it can be safely shared across threads — `Arc` allows multiple owners, `Mutex` ensures only one thread receives a job at a time.
- Each `Worker` runs a loop: it locks the receiver, waits for a `Job` message, executes it, then loops back. The lock is released after `.recv()` returns, so other workers can compete for the next job.
- `ThreadPool::execute` sends a closure (boxed as a `Job`) through the channel. Whichever worker is free picks it up immediately.
- The `Drop` implementation gracefully shuts down: it drops the `Sender` (closing the channel), which causes all workers' `.recv()` calls to return `Err`, breaking their loops. Then it joins each thread to wait for in-flight jobs to finish.

**Why this fixes the slow request problem:**  
Now each incoming connection is handed off to a worker thread via `pool.execute(|| { handle_connection(stream); })`. The main thread returns to `listener.incoming()` immediately. A `/sleep` request blocks its worker thread, but the other 3 workers remain free to handle new connections concurrently.

---

## Commit Bonus Reflection Notes

### `build` vs `new` — Function Improvement

The bonus adds a `ThreadPool::build(size: usize) -> Result<ThreadPool, PoolCreationError>` function as a safer alternative to `new`.

**The difference:**

| | `new` | `build` |
|---|---|---|
| Invalid input (`size == 0`) | Panics via `assert!` | Returns `Err(PoolCreationError(...))` |
| Return type | `ThreadPool` | `Result<ThreadPool, PoolCreationError>` |
| Caller error handling | None — crash is the only outcome | Caller decides: log, retry, exit gracefully |

**Why `build` is better in production code:**  
`new` by Rust convention implies the operation always succeeds. When it can fail (e.g. size of 0 is invalid, or resource allocation could fail), panicking is poor API design — it gives the caller no chance to recover or show a meaningful error message. `build` follows the pattern used by standard library types like `String::from_utf8` that return `Result` when construction can fail.

The `PoolCreationError` wrapper type (implementing `Display`) gives callers a descriptive error message and allows the error to propagate naturally through `?` in a `main` that returns `Result`.
