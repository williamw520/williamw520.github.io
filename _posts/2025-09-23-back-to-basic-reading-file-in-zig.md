Back to Basics: File Reading with the New IO in Zig
===========================================
***Embracing the New Input/Output API in Zig 0.15.1***

Zig 0.15.1 introduced a significant overhaul of its standard library's IO capabilities, 
unveiling a powerful new set of `Reader` and `Writer` APIs. This revamp means many existing 
code examples and assumptions about file handling are now outdated, with some familiar 
functions (like `readUntilDelimiterOrEof`) having been removed.

It's time to go back the basics and understand how to 
effectively work with files in this new landscape. This post will walk you through a 
practical example: reading data from a file and splitting it by a delimiter, all while 
exploring the essential new APIs along the way.

## The Example

Let's start by examining the example code. This program demonstrates opening a file, 
reading its contents, and printing segments separated by a user-defined delimiter (defaulting to a newline).  

`read_lines.zig`
```zig
const std = @import("std");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const alloc = gpa.allocator();

    var argv = try std.process.argsWithAllocator(alloc);
    defer argv.deinit();
    _ = argv.next();
    const filename  = if (argv.next()) |a| std.mem.sliceTo(a, 0) else "test.txt";
    const delimiter = if (argv.next()) |a| std.mem.sliceTo(a, 0) else "\n";

    // Open the file and create a buffered File.Reader.
    // Buffer size is kept small to demostrate delimiter spanning buffer boundaries.
    // In production, use a more sensible buffer size (e.g., 4KB or 8KB).
    var file = try std.fs.cwd().openFile(filename, .{ .mode = .read_only });
    defer file.close();
    var read_buf: [2]u8 = undefined;
    var file_reader: std.fs.File.Reader = file.reader(&read_buf);

    // Pointer to the std.Io.Reader interface to use the generic IO functions.
    var reader = &file_reader.interface;

    // An accumulating writer to store data read from the file.
    var line = std.Io.Writer.Allocating.init(alloc);
    defer line.deinit();

    // Main loop to read data segment by segment.
    while (true) {
        _ = reader.streamDelimiter(&line.writer, delimiter[0]) catch |err| {
            if (err == error.EndOfStream) break else return err;
        };
        _ = reader.toss(1);             // skip the delimiter byte.
        std.debug.print("{s}\n", .{ line.written() });
        line.clearRetainingCapacity();  // reset the accumulating buffer.
    }

    // Handle any remaining data after the last delimiter.
    if (line.written().len > 0) {
        std.debug.print("{s}\n", .{ line.written() });
    }
}
```

To run this example, save it as `read_lines.zig` and create a `test.txt` file (or specify another filename) with some lines of text:
```text
This is a test.
Testing 1 2 3.

After a blank line.
More test
```

Then execute from the terminal:

```bash
zig run read_lines.zig -- test.txt
```

This will print each line of `test.txt`.

### Obtaining the Reader Interface

The new IO API often requires an internal buffer for efficient buffered operations. 
This is evident when creating the `File.Reader` object:

```zig
var read_buf: [1024]u8 = undefined;
var file_reader: std.fs.File.Reader = file.reader(&read_buf);
```

Most high-level IO operations are performed through standard interfaces. 
`std.fs.File.Reader` is an implementation of the `std.Io.Reader` interface. 
It's common practice to obtain a pointer to this interface:
```zig
var reader = &file_reader.interface;
```
**Caution:**
It's crucial to obtain a *pointer* to the interface (`&file_reader.interface`) 
rather than copying the interface object. 
```zig
var reader = file_reader.interface; // DON'T DO THIS
```
The pointer references the inner `std.Io.Reader` struct 
*within* the `std.fs.File.Reader` implementation. Methods of the interface often need to access 
the "outer" parent implementation struct for context and specific details 
(e.g., [`std.fs.File.Reader.stream()`](https://github.com/ziglang/zig/blob/0.15.1/lib/std/fs/File.zig#L1314)).
Copying the interface creates an isolated copy on the stack, losing this vital link to its parent 
implementation, leading to runtime errors.

### Accumulating Read Data

The program uses `std.Io.Writer.Allocating` to dynamically accumulate data as it's read. 
This is useful when processing data of unknown or varying sizes, 
preventing buffer overflows and simplifying memory management.

```zig
var line = std.Io.Writer.Allocating.init(alloc);
defer line.deinit();
```

The core reading operation happens with `reader.streamDelimiter()`:

```zig
reader.streamDelimiter(&line.writer, delimiter[0]) catch |err| {
    if (err == error.EndOfStream) break else return err;
};
```

This function reads data from the `reader` and writes it into `line.writer` until it 
encounters the `delimiter` byte. Note that `streamDelimiter` *does not* include 
the delimiter in the written output. The delimiter byte remains in the input stream. 
To advance past it, call:
```zig
reader.toss(1);
```

The accumulated data can be retrieved as a slice using `line.written()`:
```zig
std.debug.print("{s}\n", .{ line.written() });
```

**Caution:** `line.written()` returns a slice to the `Allocating` writer's internal buffer. 
This buffer can be reallocated and moved in memory if it needs to expand. 
Therefore, the returned slice becomes *invalid* after any subsequent write operation 
that might trigger a reallocation.

### Handling the Last Part

It's common for files not to end with a delimiter. The `while` loop relying on `streamDelimiter` 
encountering a delimiter or `EndOfStream` will naturally exit. However, any data read *after* 
the last delimiter and *before* the actual end of the file will still be present in the `line` buffer.

```zig
    if (line.written().len > 0) {
        std.debug.print("{s}\n", .{ line.written() });
    }
```

This check ensures that any remaining data in the buffer after the loop finishes is also processed and printed.

To test the last part, use a custom delimiter by specifying a character as a delimiter 
on the command-line. To break data using a comma:

```bash
zig run read_lines.zig -- test2.txt ,
```

Let `test2.txt` contain comma-separated values:
```
abc,def,xyz,123,456
```

The program will output each field:
```
abc
def
xyz
123
456
```

This demonstrates the flexibility of `streamDelimiter()` for parsing various delimited data formats. 
After `streamDelimiter()` encounters the last delimiter, it continues reading 
until `EndOfStream` is hit. The final `if (line.written().len > 0)` check ensures the
trailing segment is handled.

