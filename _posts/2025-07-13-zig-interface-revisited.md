Zig Interface Revisited
=======================
***Achieving polymorphism via dynamic dispatch in Zig***

Unlike many languages that offer `interface` or `virtual` constructs, 
**Zig has no built-in notion of interfaces**. This reflects Zig’s commitment 
to simplicity and performance. But that doesn't mean polymorphism is off the table.
Zig has the tools to build interface-like behavior, making dynamic dispatch possible.

## Polymorphism in Zig: The Options

Let's backtrack a bit. There are several ways to achieve polymorphism in Zig, depending on the use case:

* **Generics and `comptime` dispatch** - for static polymorphism based on types and functions.
* **Tagged unions** - for closed sets of known types, enabling sum-type polymorphism.
* **VTable interfaces** - for dynamic dispatch across heterogeneous implementations.

A common motivation for interfaces is to allow **uniform typing**, e.g. storing multiple 
implementations in an array or map. Both **tagged unions** and **vtable-based interfaces** support this.

In this post we'll focus on **vtable interfaces**. 
There're a number of approaches to make vtable interface in Zig. As the language evolves, 
new possibilities open up. After a deep dive into the language, I have settled on one pattern. 
With some finetuning, this pattern can provide a clean, flexible, and reusable approach,
with little to no impact on implementation types.

## Goals of This Interface Pattern

This approach achieves:

* Clear separation between interface and implementation.
* No changes required in implementation types.
* Full dynamic dispatch via function pointers.
* Uniform type for all interface instances (enabling storage in arrays, maps, etc.).

Let’s explore how it works step-by-step.

---

## Example Use Case: Loggers

Let’s say we’re building a logging system that supports multiple backends:

* A debug logger that prints to the console.
* A file logger that writes to disk.
* And others.

Each logger supports a common interface: `log()` and `setLevel()`.


### Debug Logger

Below is the implementation of a logger logging to the `std.debug` output. 
It has its own idiosyncrasy like keeping track of the message count.

```zig
pub const DbgLogger = struct {
    level:  usize = 0,
    count:  usize = 0;

    pub fn log(self: *@This(), msg: []const u8) void {
        self.count += 1;
        std.debug.print("{}: [level {}] {s}\n", .{self.count, self.level, msg});
    }

    pub fn setLevel(self: *@This(), level: usize) void {
        self.level = level;
    }
};
```

### File Logger

Here is another logging implementation that logs to a file. It has its own `init()` and `deinit()` methods
dealing with the log file.

```zig
pub const FileLogger = struct {
    file:   std.fs.File,

    pub fn init(path: []const u8) !FileLogger {
        return .{ 
            .file = try std.fs.cwd().createFile(path, .{ .read = false }) 
        };
    }

    pub fn deinit(self: *@This()) void {
        self.file.close();
    }

    pub fn log(self: *@This(), msg: []const u8) void {
        self.file.writer().print("{s}\n", .{msg})
            catch |err| std.debug.print("Err: {any}\n", .{err});
    }

    pub fn setLevel(self: *@This(), level: usize) void {
        self.file.writer().print("== New Level {} =={s}\n", .{level})
            catch |err| std.debug.print("Err: {any}\n", .{err});
    }
};
```

Note: The two implementations are totally independent. They don’t know about any "interface" 
but they share the same "interface" method signatures.

## Building the VTable Interface

Below is the core `Logger` interface type, with labeled sections for explanation.

```zig
/// Logger interface
pub const Logger = struct {
    impl:       *anyopaque,     // (1) pointer to the implementation
    v_log:      *const fn(*anyopaque, []const u8) void, // (2) vtable
    v_setLevel: *const fn(*anyopaque, usize) void,      // (2) vtable

    // (3) Link the implementation to the interface.
    pub fn implBy(impl_obj: anytype) Logger {
        const delegate = LoggerDelegate(impl_obj);
        return .{
            .impl = impl_obj,
            .v_log = delegate.log,
            .v_setLevel = delegate.setLevel,
        };
    }

    // (4) Public methods of the interface

    pub fn log(self: @This(), msg: []const u8) void {
        self.v_log(self.impl, msg);
    }

    pub fn setLevel(self: @This(), level: usize) void {
        self.v_setLevel(self.impl, level);
    }
};

// (5) Delegate to turn the opaque pointer back to the implementation.
inline fn LoggerDelegate(impl_obj: anytype) type {
    const ImplType = @TypeOf(impl_obj);
    return struct {
        fn log(impl: *anyopaque, msg: []const u8) void {
            TPtr(ImplType, impl).log(msg);
        }

        fn setLevel(impl: *anyopaque, level: usize) void {
            TPtr(ImplType, impl).setLevel(level);
        }
    };
}

fn TPtr(T: type, opaque_ptr: *anyopaque) T {
    return @as(T, @ptrCast(@alignCast(opaque_ptr)));
}
```

### Using the Interface

```zig
    var dbg_logger = DbgLogger {};
    const logger1 = Logger.implBy(&dbg_logger);
    logger1.log("Hello1");
    logger1.log("Hello2");
    logger1.setLevel(2);
    logger1.log("Hello3");

    var file_logger = try FileLogger.init("log.txt");
    defer file_logger.deinit();
    const logger2 = Logger.implBy(&file_logger);
    logger2.log("Hello1");
    logger2.setLevel(3);
    logger2.log("Hello2");
    logger2.log("Hello3");
    
    const loggers = [_] Logger {logger1, logger2};
    for (loggers) |l|
        l.log("Hello to all loggers");
    
```

Both `logger1` and `logger2` are of type `Logger`, so they can be stored in arrays, 
passed to functions, or placed in maps, just like in strict typed languages with first-class interfaces.

## How It Works

Let’s review the key parts of this interface pattern:

| Part                        | Role                                                                             |
|-----------------------------|----------------------------------------------------------------------------------|
| (1) `impl: *anyopaque`      | Stores the implementation as an untyped pointer.                                 |
| (2) function pointers | The "vtable" — pointers to method shims that downcast and call the real methods. |
| (3) `implBy()`              | Connects an implementation to the interface.              |
| (4) Interface methods       | Public API of the interface. Call into the vtable with the opaque pointer. |
| (5) Delegate struct         | Reconstructs the original type and calls its methods.                            |

---

## Advantages

* **Clean separation**: Implementations don’t know about the interface.
* **Reusable**: You can build multiple interfaces using the same pattern.
* **Extensible**: Adding a new implementation requires no changes to the interface.
* **Uniform type**: You can store different implementations together.


##  Trade-offs

* Boilerplate: You must manually define each vtable method and delegate.
* Some indirection: Dynamic dispatch requires extra function pointer calls (minimal, but real).

Still, this is often the best and most flexible pattern for **runtime polymorphism in Zig**.

> The upside is that all boilerplate and complexity is confined to the interface layer; 
your implementations remain simple and pure.


## Closing Thoughts

Though Zig doesn’t have interfaces as a language feature, you can still build your own. 
You get **full control over abstraction**, and zero runtime overhead if you want it.

By manually defining vtables, you can achieve dynamic dispatch, support uniform types, 
and write expressive, decoupled APIs.

With better tooling or codegen in the future, some of the boilerplate may even be eliminated.

Until then, this approach gives you the flexibility of interfaces, in the Zig way.

---

