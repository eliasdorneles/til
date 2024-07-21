---
title: "Simple stdin/stdout text processing with Zig"
date: "2024-07-20T18:03:10+02:00"
tags:
- zig
- repl
---
In my Zig learning journey, I've managed to not use allocators for anything yet.

I've watched [this great video of Dude The Builder explaining the basics of
allocators in Zig](https://www.youtube.com/watch?v=YTcRv_BNPzA). Side note: I
am finding the videos from this Youtube channel really great, indispensable to
understand how to use the stuff from the standard library.

Anyway, allocators are all good, but for some reason whenever I want to write a
program I end up finding easier to just hardcode everything and do it with the
stack.

I think the main reason is that I find it hard to understand the standard
library documentation: there aren't many code examples for the standard library
in the website -- maybe I should look into the code itself, to see how they're
used? :thinking:

Anyhow, for today I wanted to write a simple program that read line commands
from the stdin and parsed them, a bit like [python's cmd
module](https://docs.python.org/3/library/cmd.html) -- the idea is I can build
something later that would execute the commands, in a repl-style.


### First version: hardcoded max sizes

I managed to do one first avoiding allocators completely, putting everything on
the stack.

This is both convenient and inconvenient:

* convenient to implement because you don't need to worry about (re)allocating
  and freeing memory manually, using the heap allocators
* inconvenient because it means all your inputs are limited to hardcoded max
  sizes defined at compile time

Main points I learned from that exercise:

* bunch of useful functions to split byte sequences in the standard libary, they all start with: `std.mem.split`
* in my `Command.init` class, I had forgotten to put `self: *Command` instead
  of `self: Command`, so the compiler was rightfully telling that `self.name`
  was const, and it took me a while to figure out that that was the problem. At
  first I was messing around with my type declaration `name: []const u8`, which
  was actually fine.

Here's the full code:


```zig
const std = @import("std");

const MAX_ARGS = 20;

const Command = struct {
    name: []const u8 = "",
    args: [MAX_ARGS][]const u8 = .{""} ** MAX_ARGS,

    pub fn init(self: *Command, cmd_line: []u8) void {
        var it = std.mem.splitScalar(u8, cmd_line, ' ');
        self.name = it.next() orelse "";

        var argIndex: u8 = 0;
        while (it.next()) |word| {
            self.args[argIndex] = word;
            argIndex += 1;
            if (argIndex == self.args.len) {
                break;
            }
        }
    }

    pub fn print(self: Command) !void {
        const stdout_file = std.io.getStdOut().writer();
        var bw = std.io.bufferedWriter(stdout_file);
        const stdout = bw.writer();
        try stdout.print("Cmd: {s}\n", .{self.name});
        try stdout.print("Args: ", .{});
        for (self.args) |arg| {
            if (std.mem.eql(u8, arg, "")) {
                break;
            }
            try stdout.print("{s} ", .{arg});
        }
        try stdout.print("\n\n", .{});
        try bw.flush();
    }
};

fn prompt(text: []const u8, buf: []u8) ![]u8 {
    const stdout = std.io.getStdOut().writer();
    const stdin = std.io.getStdIn().reader();

    try stdout.print("{s}", .{text});

    return stdin.readUntilDelimiter(buf, '\n');
}

pub fn main() !void {
    const stdout = std.io.getStdOut().writer();

    var buf: [1000]u8 = undefined;
    while (prompt("repl % ", &buf)) |cmd_line| {
        var cmd = Command{};
        cmd.init(cmd_line);
        try cmd.print();
    } else |err| {
        try switch (err) {
            error.EndOfStream => stdout.print("Bye\n", .{}),
            else => stdout.print("Error: {}\n", .{err}),
        };
    }
}
```

So this works fine, but as you can see, it limits input lines to 1000
characters and the command arguments to 20 max.

Now, let's see what it looks like a version with dynamically allocated memory.


### Second version: using the allocators

Before implementing this, I watched [another video from Dude the Builder where
he shows how to implement a
Stack](https://www.youtube.com/watch?v=RCiGRNlaoJY), this helped to understand
better how to use the allocator and how to manipulate slices.

I also applied the idea of adding a `.format(...)` method to my `struct` type
that can be used in the `print()` functions, instead of the ad-hoc `print()`
method I had done previously.

Here is what the code looks like:


```zig
const std = @import("std");
const mem = std.mem;

const Command = struct {
    allocator: mem.Allocator,
    name: []const u8 = "",
    args: [][]const u8 = &.{},
    _args_cap: usize = 0,

    const args_grow_factor: u8 = 2;

    pub fn init(self: *Command, cmd_line: []u8) !void {
        var it = mem.splitScalar(u8, cmd_line, ' ');
        self.name = try self.allocator.dupe(u8, it.next() orelse "");

        var i: u16 = 0;
        while (it.next()) |word| {
            if (mem.eql(u8, word, "")) continue;

            if (i >= self._args_cap) try self.growArgs();
            if (i >= self.args.len) self.args.len += 1;

            self.args[i] = try self.allocator.dupe(u8, word);
            i += 1;
        }
    }

    pub fn deinit(self: *Command) void {
        self.allocator.free(self.name);
        for (self.args) |arg| {
            self.allocator.free(arg);
        }
        if (self._args_cap > 0) self.allocator.free(self.args.ptr[0..self._args_cap]);
    }

    fn growArgs(self: *Command) !void {
        const old_len = self.args.len;
        const new_cap = if (self._args_cap == 0) 8 else self._args_cap * args_grow_factor;
        self.args = try self.allocator.realloc(self.args.ptr[0..self._args_cap], new_cap);
        self.args.len = old_len;
        self._args_cap = new_cap;
    }

    pub fn format(
        self: Command,
        comptime _: []const u8,
        _: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        try writer.writeAll("Command {");
        try writer.print(" name={s}, args=.{{", .{self.name});
        for (self.args, 0..) |arg, i| {
            if (mem.eql(u8, arg, "")) {
                break;
            } else {
                if (i != 0) try writer.writeAll(", ");
            }
            try writer.print("\"{s}\"", .{arg});
        }
        try writer.writeAll("} }");
    }
};

fn prompt(text: []const u8, line: *std.ArrayList(u8)) ![]u8 {
    const stdout = std.io.getStdOut().writer();
    try stdout.print("{s}", .{text});

    const stdin = std.io.getStdIn().reader();
    const line_writer = line.writer();
    try stdin.streamUntilDelimiter(line_writer, '\n', null);
    return line.items;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const stdout = std.io.getStdOut().writer();

    var line = std.ArrayList(u8).init(allocator);
    defer line.deinit();

    while (prompt("repl % ", &line)) |cmd_line| {
        defer line.clearRetainingCapacity();

        var cmd = Command{ .allocator = allocator };
        defer cmd.deinit();

        try cmd.init(cmd_line);
        try stdout.print("{s}\n", .{cmd});
    } else |err| {
        try switch (err) {
            error.EndOfStream => stdout.print("Bye\n", .{}),
            else => stdout.print("Error: {}\n", .{err}),
        };
    }
}
```

What were the main things that changed?

1. the `prompt()` function now takes `ArrayList(u8)` instead of a `[]u8` buffer
2. the `Command` struct now takes a `mem.allocator`, its field `args` no longer
   has a fixed size, plus it has now an `_args_capacity` field
3. the `Command.init` method now duplicates memory from the input line, and
   also grows the `args` slice dynamically.

The code has increased about 50% of the size, but now it works for any input
size, limited only by the available memory.

I've just now realized that I could have used `ArrayList` for building `args`
instead of allocating memory manually. I don't regret it as this was well worth
the exercise, but now I'm curious how much simpler it will get after I
refactor!

For some next time! ;-)

**Update:** here is the refactored version with ArrayList, shorter and simpler:

```zig
const std = @import("std");
const mem = std.mem;

const Command = struct {
    allocator: mem.Allocator,
    name: []const u8 = "",
    args: [][]const u8 = &.{},

    pub fn init(self: *Command, cmd_line: []u8) !void {
        var it = mem.splitScalar(u8, cmd_line, ' ');
        self.name = try self.allocator.dupe(u8, it.next() orelse "");

        var temp = std.ArrayList([]const u8).init(self.allocator);
        while (it.next()) |word| {
            if (mem.eql(u8, word, "")) continue;

            try temp.append(try self.allocator.dupe(u8, word));
        }
        self.args = try temp.toOwnedSlice();
    }

    pub fn deinit(self: *Command) void {
        self.allocator.free(self.name);
        for (self.args) |arg| {
            self.allocator.free(arg);
        }
        self.allocator.free(self.args);
    }

    pub fn format(
        self: Command,
        comptime _: []const u8,
        _: std.fmt.FormatOptions,
        writer: anytype,
    ) !void {
        try writer.writeAll("Command {");
        try writer.print(" name={s}, args=.{{", .{self.name});
        for (self.args, 0..) |arg, i| {
            if (mem.eql(u8, arg, "")) {
                break;
            } else {
                if (i != 0) try writer.writeAll(", ");
            }
            try writer.print("\"{s}\"", .{arg});
        }
        try writer.writeAll("} }");
    }
};

fn prompt(text: []const u8, line: *std.ArrayList(u8)) ![]u8 {
    const stdout = std.io.getStdOut().writer();
    try stdout.print("{s}", .{text});

    const stdin = std.io.getStdIn().reader();
    const line_writer = line.writer();
    try stdin.streamUntilDelimiter(line_writer, '\n', null);
    return line.items;
}

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    defer _ = gpa.deinit();
    const allocator = gpa.allocator();

    const stdout = std.io.getStdOut().writer();

    var line = std.ArrayList(u8).init(allocator);
    defer line.deinit();

    while (prompt("repl % ", &line)) |cmd_line| {
        defer line.clearRetainingCapacity();

        var cmd = Command{ .allocator = allocator };
        defer cmd.deinit();

        try cmd.init(cmd_line);
        try stdout.print("{s}\n", .{cmd});
    } else |err| {
        try switch (err) {
            error.EndOfStream => stdout.print("Bye\n", .{}),
            else => stdout.print("Error: {}\n", .{err}),
        };
    }
}
```


### Little gotcha: globals inside structs are globally shared

Another thing I've learned today that was a bit unexpected to me was that
private global variables defined inside structs are shared globally across all
"instances" of that struct.

It makes sense, once you think that modules are structs as well, but it was a
bit unexpected to me.

Take this code, for instance, what do you expect its output to be:

```zig
const std = @import("std");
const log = std.log;

const SomeStruct = struct {
    name: []const u8 = "",

    var counter: u8 = 0;  // this variable is actually globally shared

    pub fn rename(self: *@This(), newName: []const u8) void {
        self.name = newName;
        counter += 1;
        log.debug("Renamed {} times", .{counter});
    }
};

pub fn main() !void {
    var s = SomeStruct{ .name = "hello" };
    s.rename("world");
    s.rename("world2");

    var s2 = SomeStruct{ .name = "hello2" };
    s2.rename("hola");
    s2.rename("buenas");
}
```

I was initially thinking each struct would have its own `counter` variable, and
that the output would be:

```
debug: Renamed 1 times
debug: Renamed 2 times
debug: Renamed 1 times
debug: Renamed 2 times
```

But, the actual output is:

```
debug: Renamed 1 times
debug: Renamed 2 times
debug: Renamed 3 times
debug: Renamed 4 times
```

as `counter` variable is indeed a global.
