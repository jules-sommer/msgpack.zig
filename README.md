# Zig library for working with msgpack messages

## Installation

1) Add msgpack.zig as a dependency in your `build.zig.zon`:

```bash
zig fetch --save git+https://github.com/lalinsky/msgpack.zig#main
```

2) In your `build.zig`, add the `msgpack` module as a dependency you your program:

```zig
const msgpack = b.dependency("msgpack", .{
    .target = target,
    .optimize = optimize,
});

// the executable from your call to b.addExecutable(...)
exe.root_module.addImport("msgpack", httpz.module("msgpack"));
```

## Usage

Basic encoding and decoding:

```zig
const std = @import("std");
const msgpack = @import("msgpack");

const Message = struct {
    name: []const u8,
    age: u8,
};

var buffer = std.ArrayList(u8).init(allocator);
defer buffer.deinit();

try msgpack.encode(Message{
    .name = "John",
    .age = 20,
}, buffer.writer());

const decoded = try msgpack.decodeFromSlice(Message, allocator, buffer.items);
defer decoded.deinit();

std.debug.assert(std.mem.eql(u8, decoded.name, "John"));
std.debug.assert(decoded.age == 20);
```

The encoded message will use field names as keys to encode the message. In order to generate more compact messages, you can change the format to use field indexes:

```zig
const std = @import("std");
const msgpack = @import("msgpack");

const Message = struct {
    name: []const u8,
    age: u8,

    pub fn msgpackFormat() msgpack.StructFormat {
        return .{ .as_map = .{ .key = .field_index } };
    }
};
```

Or you can also use field name prefixes:

```zig
const std = @import("std");
const msgpack = @import("msgpack");

const Message = struct {
    name: []const u8,
    age: u8,

    pub fn msgpackFormat() msgpack.StructFormat {
        return .{ .as_map = .{ .key = .{ .field_name_prefix = 1 } } };
    }
};
```

Both options have the disadvantage that changing the fields in the struct will have impact on the encoded message, so you need to be careful about backwarads compatibility.
You can also use custom protobuf-like field keys to ensure full compatibility even after changing the struct:

```zig
const std = @import("std");
const msgpack = @import("msgpack");

const Message = struct {
    items: []u32,

    pub fn msgpackFormat() msgpack.StructFormat {
        return .{ .as_map = .{ .key = .custom } };
    }

    pub fn msgpackFieldKey(field: std.meta.FieldEnum(@This())) u8 {
        return switch (field) {
            .items => 1,
        };
    }
};
```

Or you can use a completely custom format:

```zig
const std = @import("std");
const msgpack = @import("msgpack");

const Message = struct {
    items: []u32,

    pub fn msgpackWrite(self: Message, packer: anytype) !void {
        try packer.writeArray(u32, self.items);
    }

    pub fn msgpackRead(unpacker: anytype) !Message {
        const items = try unpacker.readArray(u32);
        return Message{ .items = items };
    }
};
```

