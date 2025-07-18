Memory-Efficient Zig Interface
==============================
***Memory-efficient dynamic dispatch via shared static vtables***

This post follows up on [Zig Interface Revisited](/2025/07/13/zig-interface-revisited.html).

The **vtable interface** introduced in the previous post embeds function pointers directly 
within each interface instance, alongside the opaque implementation pointer. For example:

```zig
/// Logger interface
pub const Logger = struct {
    impl:       *anyopaque,     // (1) pointer to the implementation
    v_log:      *const fn(*anyopaque, []const u8) void, // (2) vtable
    v_setLevel: *const fn(*anyopaque, usize) void,      // (2) vtable
    ...
};
```

This layout simplifies usage and provides excellent performance: calling a function pointer 
involves only a single indirection. However, it increases the memory footprint per 
interface instance, as each instance stores all its function pointers.

For interfaces like `Logger`, which may only have a few instances, this overhead is negligible. 
But for interfaces that are instantiated many times, such as a Shape interface for triangles, rectangles, circles, etc.,
potentially with millions of objects, this memory cost can become significant.

## A More Memory Efficient VTable

Adopting a more memory efficient vtable layout requires only minor changes.

The following updated version of `Logger` (adopted from the [previous post](2025/07/13/zig-interface-revisited.html))
uses a compact representation. Each interface instance now stores just two pointers: 
one to the implementation instance and one to a shared vtable.

Here's the important part.

**The vtable itself is a const struct value created at compile time as a comptime value.
Zig places it in the program’s static memory with static lifetime, allowing it to be shared 
across all instances of the same interface type.**

```zig
pub const Logger = struct {
    impl:       *anyopaque,
    vtable:     *const VTable,  // (1) vtable pointer

    // (2) implementation function pointers.
    const VTable = struct {
        v_log:      *const fn(*anyopaque, []const u8) void,
        v_setLevel: *const fn(*anyopaque, usize) void,
    };

    pub fn implBy(impl_obj: anytype) Logger {
        const delegate = LoggerDelegate(impl_obj);
        const vtable = VTable { // (3) const vtable is a comptime value.
            .v_log = delegate.log,
            .v_setLevel = delegate.setLevel,
        };
        return .{
            .impl = impl_obj,
            .vtable = &vtable,  // (4) comptime value is in static memory.
        };
    }

    pub fn log(self: Logger, msg: []const u8) void {
        self.vtable.v_log(self.impl, msg);
    }

    pub fn setLevel(self: Logger, level: usize) void {
        self.vtable.v_setLevel(self.impl, level);
    }
};
```

Let’s review the changes.

| Part                                    | Role                                                                                                                    |
|-----------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| (1) `vtable`                            | A pointer to a shared const vtable stored in static memory.                                                             |
| (2) `VTable`                            | Defines the function pointers for the implementation.                                                                        |
| (3)&nbsp;<code>const&nbsp;vtable</code> | The vtable is created at comptime and stored statically.                                                                |
| (4) `&vtable`                           | The pointer address is shared across all interface instances using the same implementation type, reducing per-instance memory. |

---
Conclusion

Moving the function pointers out of each interface instance and into a shared static vtable
significantly reduce memory overhead without sacrificing performance. 
This approach makes Zig interfaces more scalable in memory-constrained scenarios,
especially when dealing with a large number of interface instances.

