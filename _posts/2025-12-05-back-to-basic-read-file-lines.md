Back to Basics: Reading a File into Lines in Zig
====================================
***Building a Simple API for Reading Lines of a File in Zig***

<small>(See [File Reading with the New IO in Zig](/2025/09/23/back-to-basic-reading-file-in-zig.html) for reading a file line by line.)</small>  
<small>(See [Reading Files Fully in Zig](/2025/12/04/back-to-basic-read-file-fully.html) for reading an entire file.)</small>

A common need in programming is to process the lines of a text file, 
whether you are parsing configuration files, analyzing logs, or importing data sets. 
While reading a file line by line is memory efficient for massive files, 
it can be cumbersome if you need random access to specific lines or 
need to perform multiple passes over the data. In such cases, it would be 
convenient to have a simple API that reads the entire file into memory in one shot 
and makes the lines available afterward as indexable slices.

Let's build this API in Zig.

To implement this robustly, we must manage the lifespan of the data. 
We cannot simply return a list of strings without keeping the underlying file contents alive. 

We want to achieve the following:

- Open and read the file into a buffer.
  Since we might not know the file size beforehand, we'll use a growing buffer to accommodate the file's contents.

- Split the file data into lines.
  Parse the raw byte data to identify newline characters and create views (slices) into that data.

- Store the file data and lines in a struct.
  The struct will act as the "owner" of both the raw file text and the list of line slices, 
  ensuring the text isn't freed while we are still trying to read the lines.

The following `FileLines.read()` takes a `Dir` and a filename, reads its contents, and parses them into lines.
The lines can then be accessed via the `.lines()` method.

```zig
// Read a file into lines.
pub const FileLines = struct {
    alloc:      std.mem.Allocator,
    file_data:  std.Io.Writer.Allocating,       // data of the entire file.
    slices:     std.ArrayList([]const u8),      // slices into the file data.

    pub fn read(alloc: std.mem.Allocator, dir: std.fs.Dir, 
                filename: []const u8) !FileLines {
        var file = try dir.openFile(filename, .{});
        defer file.close();

        const buf = try alloc.alloc(u8, 4096);  // use heap; avoid taking stack space
        defer alloc.free(buf);
        var f_reader = file.reader(buf);
        var file_data = std.Io.Writer.Allocating.init(alloc);
        _ = try f_reader.interface.streamRemaining(&file_data.writer);

        var line_slices: std.ArrayList([]const u8) = .empty;
        var itr = std.mem.splitScalar(u8, file_data.written(), '\n');
        while (itr.next()) |line| {
            try line_slices.append(alloc, std.mem.trim(u8, line, "\r"));
        }

        return .{
            .alloc = alloc,
            .file_data = file_data,
            .slices = line_slices,
        };
    }

    pub fn deinit(self: *FileLines) void {
        self.file_data.deinit();
        self.slices.deinit(self.alloc);
    }

    pub fn lines(self: *const FileLines) [][]const u8 {
        return self.slices.items;
    }
};
```

Here is how to use the API to read and access the lines of a file. 
Note the importance of calling deinit to clean up the allocated memory once you are done processing:

```zig
    var fl = try FileLines.read(alloc, std.fs.cwd(), "test.txt");
    defer fl.deinit();
    for (fl.lines())|line| {
        std.debug.print("line = '{s}'\n", .{line});
    }

```

See [file_lines.zig](https://github.com/williamw520/misc_zig/blob/main/file_lines.zig) for the code.

