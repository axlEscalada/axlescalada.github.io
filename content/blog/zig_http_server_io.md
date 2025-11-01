+++
title = "Setting up an HTTP server in Zig 0.15"
date = 2025-11-01
+++

Zig 0.15 introduced a new IO system that represents the first step toward a complete redesign of async I/O in Zig. While the full async capabilities are still being developed, the changes to HTTP server setup give us a glimpse of where Zig is heading.

Here's a complete example of a minimal HTTP server using the new IO system:

```zig,linenos
const std = @import("std");

pub fn main() !void {
    const address = try std.net.Address.parseIp4("127.0.0.1", 8080);

    var server = try address.listen(std.net.Address.ListenOptions{});
    defer server.deinit();

    while (true) {
        try handleConnection(try server.accept());
    }
}

fn handleConnection(conn: std.net.Server.Connection) !void {
    defer conn.stream.close();

    var recv_buffer: [1024]u8 = undefined;
    var send_buffer: [1024]u8 = undefined;
    var conn_reader = conn.stream.reader(&recv_buffer);
    var conn_writer = conn.stream.writer(&send_buffer);

    var http_server = std.http.Server.init(conn_reader.interface(), &conn_writer.interface);
    var req = try http_server.receiveHead();
    try req.respond("hello world\n", std.http.Server.Request.RespondOptions{});
}
```

Compare this to how it was done before Zig 0.15:

```zig,linenos
fn handleConnection(conn: std.net.Server.Connection) !void {
    defer conn.stream.close();
    var buffer: [1024]u8 = undefined;
    var http_server = std.http.Server.init(conn, &buffer);
    var req = try http_server.receiveHead();
    try req.respond("hello world\n", std.http.Server.Request.RespondOptions{});
}
```

The old approach was simpler: pass the connection and a single buffer to <code class="highlight">std.http.Server.init</code>. The new approach requires separate buffers and explicit reader/writer creation. But this added complexity serves a purpose.

## Why the IO system matters

The changes in Zig 0.15 are the foundation for a much larger redesign. The goal is to make Zig code work optimally across different execution models, from single-threaded blocking I/O to event loops with thread pools to stackless coroutines without rewriting your code.

## The bigger picture

The full vision for Zig's async I/O involves passing an <code class="highlight">Io</code> interface as a parameter to your functions, similar to how you pass an <code class="highlight">Allocator</code>. This will allow the caller to decide the execution model: blocking, thread pool, green threads, or stackless coroutines and inject it into all dependencies.

The HTTP server changes in 0.15 are preparing the standard library for this future. By moving to explicit reader/writer interfaces now, the codebase can later be updated to work with the <code class="highlight">Io</code> interface without requiring massive rewrites.

## Running the server

To run this server, save the code to a file (e.g., <code class="highlight">server.zig</code>) and compile it with:

```bash,linenos
zig build-exe server.zig
./server
```

Test it by navigating to <code class="highlight">http://127.0.0.1:8080</code> or using curl:

```bash,linenos
curl http://127.0.0.1:8080
```

You should see "hello world" as the response.

## Migrating from pre-0.15 code

If you're updating code from before Zig 0.15:

**Old approach:**
```zig,linenos
var buffer: [1024]u8 = undefined;
var http_server = std.http.Server.init(conn, &buffer);
```

**New approach:**
```zig,linenos
var recv_buffer: [1024]u8 = undefined;
var send_buffer: [1024]u8 = undefined;
var conn_reader = conn.stream.reader(&recv_buffer);
var conn_writer = conn.stream.writer(&send_buffer);
var http_server = std.http.Server.init(conn_reader.interface(), &conn_writer.interface);
```

The new approach requires more explicit setup, but it's preparing your code for a future where you'll be able to switch execution models without rewriting anything. The same HTTP server code will work efficiently whether you're using blocking I/O, an event loop, or stackless coroutines.
