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
