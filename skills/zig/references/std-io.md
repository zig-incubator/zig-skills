# std.Io - I/O Interface Reference (0.16.0)

Zig 0.16.0 introduces a complete rewrite of the I/O system with async/await support, concurrent operations, and a unified interface across all I/O operations.

## Table of Contents
- [Overview](#overview)
- [Setting Up I/O](#setting-up-io)
- [I/O Implementations](#io-implementations)
- [Async/Await](#asyncawait)
- [Concurrent Operations](#concurrent-operations)
- [File System Operations](#file-system-operations)
- [Networking](#networking)
- [Time and Sleeping](#time-and-sleeping)
- [Synchronization Primitives](#synchronization-primitives)
- [Migration from 0.15.x](#migration-from-015x)

## Overview

The new `std.Io` interface abstracts all I/O operations and concurrency:
- File system
- Networking
- Processes
- Time and sleeping
- Randomness
- Async, await, concurrent, and cancel
- Concurrent queues
- Wait groups and select
- Mutexes, futexes, events, and conditions
- Memory mapped files

## Setting Up I/O

### Basic Setup (Threaded)
```zig
const std = @import("std");
const Io = std.Io;

pub fn main() !void {
    // Set up allocator
    var debug_allocator: std.heap.DebugAllocator(.{}) = .init;
    defer std.debug.assert(debug_allocator.deinit() == .ok);
    const gpa = debug_allocator.allocator();

    // Set up I/O implementation
    var threaded: std.Io.Threaded = .init(gpa);
    defer threaded.deinit();
    const io = threaded.io();

    // Use I/O operations
    try doWork(io);
}

fn doWork(io: Io) !void {
    std.debug.print("working\n", .{});
    io.sleep(.fromSeconds(1), .awake) catch {};
}
```

### "Juicy Main" Pattern
The recommended pattern for Zig applications:

```zig
pub fn main() !void {
    var debug_allocator: std.heap.DebugAllocator(.{}) = .init;
    defer std.debug.assert(debug_allocator.deinit() == .ok);
    const gpa = debug_allocator.allocator();

    var threaded: std.Io.Threaded = .init(gpa);
    defer threaded.deinit();
    const io = threaded.io();

    return juicyMain(gpa, io);
}

fn juicyMain(gpa: std.mem.Allocator, io: Io) !void {
    // Application logic here
}
```

## I/O Implementations

### std.Io.Threaded
Thread-based I/O implementation. Works on all platforms.

```zig
var threaded: std.Io.Threaded = .init(gpa);
defer threaded.deinit();
const io = threaded.io();

// Optionally limit concurrency
threaded.cpu_count = 1;  // Single thread
```

### std.Io.Evented (Platform-Specific)
Event-driven I/O using platform-specific mechanisms:

```zig
pub const Evented = if (fiber.supported) switch (builtin.os.tag) {
    .linux => Uring,           // io_uring
    .dragonfly, .freebsd, .netbsd, .openbsd => Kqueue,
    .driverkit, .ios, .maccatalyst, .macos, .tvos, .visionos, .watchos => Dispatch,
    else => void,
} else void;
```

### std.Io.Uring (Linux)
Linux io_uring implementation for high-performance async I/O.

### std.Io.Kqueue (BSD/macOS)
Kqueue-based event system for BSD and macOS.

### std.Io.Dispatch (Apple platforms)
Grand Central Dispatch-based implementation for Apple platforms.

## Async/Await

### Basic Async
Decouple function calling from function returning:

```zig
fn juicyMain(gpa: Allocator, io: Io) !void {
    var future = io.async(doWork, .{io});
    try future.await(io);
}

fn doWork(io: Io) !void {
    std.debug.print("working\n", .{});
    io.sleep(.fromSeconds(1), .awake) catch {};
}
```

### Multiple Concurrent Tasks
```zig
fn juicyMain(gpa: Allocator, io: Io) !void {
    var a = io.async(doWork, .{ io, "task A" });
    var b = io.async(doWork, .{ io, "task B" });

    try a.await(io);
    try b.await(io);
}

fn doWork(io: Io, name: []const u8) !void {
    std.debug.print("working on {s}\n", .{name});
    io.sleep(.fromSeconds(1), .awake) catch {};
}
```

### Error Handling with Cancellation
Use `defer` with `cancel` for proper resource cleanup:

```zig
fn juicyMain(gpa: Allocator, io: Io) !void {
    var a = io.async(doWork, .{ gpa, io, "task A" });
    defer a.cancel(io) catch {};

    var b = io.async(doWork, .{ gpa, io, "task B" });
    defer b.cancel(io) catch {};

    try a.await(io);
    try b.await(io);
}
```

### Resource Management
Handle allocated return values properly:

```zig
fn juicyMain(gpa: Allocator, io: Io) !void {
    var a = io.async(allocString, .{ gpa, io, "task A" });
    defer if (a.cancel(io)) |s| gpa.free(s) else |_| {};

    var b = io.async(allocString, .{ gpa, io, "task B" });
    defer if (b.cancel(io)) |s| gpa.free(s) else |_| {};

    const a_string = try a.await(io);
    const b_string = try b.await(io);
    std.debug.print("finished {s}\n", .{a_string});
    std.debug.print("finished {s}\n", .{b_string});
}

fn allocString(gpa: Allocator, io: Io, name: []const u8) ![]u8 {
    const copied = try gpa.dupe(u8, name);
    std.debug.print("working {s}\n", .{copied});
    io.sleep(.fromSeconds(1), .awake) catch {};
    return copied;
}
```

## Concurrent Operations

### Asynchrony vs Concurrency
- **Asynchrony** (`io.async`): Decouple call from return, but may not provide parallelism
- **Concurrency** (`io.concurrent`): Explicitly requests parallel execution

```zig
// WRONG: async may deadlock with single thread
var task1 = io.async(producer, .{io, &queue, "data"});
var task2 = io.async(consumer, .{io, &queue});

// CORRECT: concurrent ensures parallelism
var task1 = try io.concurrent(producer, .{io, &queue, "data"});
var task2 = try io.concurrent(consumer, .{io, &queue});
```

### Concurrency with Error Handling
```zig
fn doConcurrent(io: Io) !void {
    var queue: Io.Queue([]const u8) = .init(&.{});

    var producer_task = try io.concurrent(producer, .{ io, &queue, "message" });
    defer producer_task.cancel(io) catch {};

    var consumer_task = try io.concurrent(consumer, .{ io, &queue });
    defer _ = consumer_task.cancel(io) catch {};

    const result = try consumer_task.await(io);
    std.debug.print("received: {s}\n", .{result});
}

fn producer(io: Io, queue: *Io.Queue([]const u8), msg: []const u8) !void {
    try queue.putOne(io, msg);
}

fn consumer(io: Io, queue: *Io.Queue([]const u8)) ![]const u8 {
    return queue.getOne(io);
}
```

## File System Operations

### File Reading
```zig
fn readFile(io: Io, gpa: std.mem.Allocator, path: []const u8) ![]u8 {
    const file = try std.fs.cwd().openFile(path, .{});
    defer file.close();

    // New allocation function in 0.16.0
    return file.readToEndAlloc(gpa, 1024 * 1024);
}
```

### Directory Operations
```zig
fn listDirectory(io: Io) !void {
    var dir = try std.fs.cwd().openDir("src", .{ .iterate = true });
    defer dir.close();

    var iter = dir.iterate();
    while (try iter.next()) |entry| {
        std.debug.print("{s}\n", .{entry.name});
    }
}
```

### File.Stat with Optional access_time
```zig
const stat = try file.stat();
std.debug.print("size: {}\n", .{stat.size});

// access_time is now optional
if (stat.access_time) |atime| {
    std.debug.print("access time: {}\n", .{atime});
}
```

## Networking

### TCP Connection
```zig
fn connectToServer(io: Io, gpa: std.mem.Allocator) !void {
    const addr = try std.net.Address.resolveIp("127.0.0.1", 8080);
    const stream = try std.net.tcpConnectToAddress(addr);
    defer stream.close();

    // Use I/O operations with the stream
}
```

## Time and Sleeping

### Basic Sleep
```zig
fn sleepExample(io: Io) void {
    // Sleep for 1 second
    io.sleep(.fromSeconds(1), .awake) catch {};

    // Sleep for 500 milliseconds
    io.sleep(.fromMillis(500), .awake) catch {};
}
```

### Timestamps
```zig
fn timestampExample(io: Io) !void {
    const timestamp = io.now();
    std.debug.print("timestamp: {}\n", .{timestamp});

    const duration = std.Io.Duration.fromSeconds(10);
    // ... work ...
    const elapsed = std.Io.Timestamp.since(io.now(), timestamp);
}
```

## Synchronization Primitives

### Queue
Unbuffered queue for producer/consumer patterns:

```zig
var queue: Io.Queue(Message) = .init(&.{});

// Producer
try queue.putOne(io, message);

// Consumer
const msg = try queue.getOne(io);
```

### RwLock
```zig
var lock: Io.RwLock = .{};

// Read lock
lock.lockShared();
defer lock.unlockShared();

// Write lock
lock.lock();
defer lock.unlock();
```

### Semaphore
```zig
var sem: Io.Semaphore = .init(3);  // 3 permits

// Acquire
try sem.wait();
defer sem.post();
```

## Migration from 0.15.x

### Removed Types
- `GenericReader`, `GenericWriter` → Use `std.Io.Reader`, `std.Io.Writer`
- `AnyReader`, `AnyWriter` → Use interface types
- `FixedBufferStream` → Use `std.Io.Reader.fixed()`, `std.Io.Writer.fixed()`
- `BufferedWriter`, `BufferedReader` → Buffer is now in the interface

### File I/O Migration
```zig
// OLD (0.15.x)
var buf: [4096]u8 = undefined;
var writer = file.writer(&buf);
const w = &writer.interface;
try w.print("Hello\n", .{});
try w.flush();

// NEW (0.16.0) - Use std.Io interface
var threaded: std.Io.Threaded = .init(gpa);
defer threaded.deinit();
const io = threaded.io();
// File operations now integrated with I/O interface
```

### Allocator Changes
```zig
// OLD (0.15.x)
const thread_safe = std.heap.ThreadSafe.allocator();

// NEW (0.16.0)
var arena = std.heap.ArenaAllocator.init(std.heap.page_allocator);
defer arena.deinit();
const allocator = arena.allocator();  // Now thread-safe by default
```

### Thread Pool Replacement
```zig
// OLD (0.15.x)
var pool: std.Thread.Pool = undefined;
try pool.init(.{ .allocator = gpa, .n_jobs = 4 });

// NEW (0.16.0)
var threaded: std.Io.Threaded = .init(gpa);
defer threaded.deinit();
const io = threaded.io();
var task = try io.concurrent(doWork, .{io});
```
