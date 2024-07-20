---
title: "Simple stdin/stdout text processing with Zig"
date: "2024-07-20T18:03:10+02:00"
tags:
- zig
- repl
---
In my Zig learning journey, I've managed to not use allocators for anything yet.

I've watched [this great video of Dude The Builder explaining the basics of
allocators in Zig](https://www.youtube.com/watch?v=YTcRv_BNPzA). Btw, I am
finding these videos really great, indispensable to understand how to use the
stuff from the standard library.

Anyway, allocators are all good, but for some reason whenever I want to write a
program I find myself figuring out ways to do it with the stack.

I think the main reason is that I find it hard to understand the standard
library documentation: there aren't many code examples for the standard library
in the website -- maybe I should look into the code itself, to see how they're
used? :thinking:

Anyhow, for today I wanted to write a simple program that read line commands
from the stdin and parsed them, a bit like [python's cmd
module](https://docs.python.org/3/library/cmd.html) -- the idea is I can build
something later that would execute the commands, in a repl-style.

I managed to do it avoiding allocators completely, putting everything on the
stack.

Main points I take away from today's exercise:

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

Now, I'll figure out a version with data in the heap for some next time. ;-)
