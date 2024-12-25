---
title: "Parsing command-line args in Zig using only stdlib"
date: "2024-12-24T18:54:07+01:00"
tags: []
---
In my Zig learning journey, for some problems I have found the standard library
documentation a bit too curt.
Some solutions to common problems are documented in other non-official sources.

One such source is the [Zig Cookbook](https://cookbook.ziglang.cc/) website,
which is pretty great for that.

Wanna know how to generate random numbers? [There is a short example code for that.](https://cookbook.ziglang.cc/06-01-rand.html)

Wanna know how to make an HTTP request with Zig's stdlib functions? [You got it!](https://cookbook.ziglang.cc/05-01-http-get.html).

So I was a bit surprised that [the example for argument
parsing](https://cookbook.ziglang.cc/13-01-argparse.html) doesn't use the
standard library, instead it lists three different libraries and showcase one
of them.

I suppose if you want to build a big project, with subcommands and whatnot,
using a library is justified, but I don't like adding new dependencies unless I
have a really big reason for it.

So I decided to write some simple code that would cover my needs for simple
scripts.

Without further ado, here is the code:

```zig
const std = @import("std");

var PROGRAM_NAME: []const u8 = undefined;

const USAGE_FMT =
    \\Usage: {s} [-B buf_size] [-o OUT_WAV_FILE] DURATION PORT1 [PORT2]...
    \\
    \\Options:
    \\  -B buf_size    Buffer size in bytes (default: 16384)
    \\  -o OUT_WAV_FILE  Output WAV file path (default: wave_out.wav)
    \\  -v             Verbose output
    \\
    \\Arguments:
    \\  DURATION       Duration in seconds
    \\  port1 [ port2 ... ]  List of ports
    \\
    \\
;

// CLI options
const CliArgs = struct {
    verbose: bool = false,
    buf_size: u32 = 16384,
    output_path: []const u8 = "wave_out.wav",
    duration: u32 = 0,
    ports: [][]u8 = undefined,
};

fn display_usage() void {
    std.debug.print(USAGE_FMT, .{PROGRAM_NAME});
}

const ArgParseError = error{ MissingArgs, InvalidArgs };

fn parseArgs(argv: [][]u8) ArgParseError!CliArgs {
    PROGRAM_NAME = std.fs.path.basename(argv[0]);
    var args = CliArgs{};

    // parse optional arguments i.e. anything that start with a dash '-'
    var optind: usize = 1;
    while (optind < argv.len and argv[optind][0] == '-') {
        if (std.mem.eql(u8, argv[optind], "-v")) {
            args.verbose = true;
        } else if (std.mem.eql(u8, argv[optind], "-B")) {
            if (optind + 1 >= argv.len) {
                display_usage();
                return error.MissingArgs;
            }
            optind += 1;
            args.buf_size = std.fmt.parseInt(u32, argv[optind], 10) catch {
                display_usage();
                std.debug.print("Invalid buffer size: '{s}'\n", .{argv[optind]});
                return error.InvalidArgs;
            };
        } else if (std.mem.eql(u8, argv[optind], "-o")) {
            if (optind + 1 >= argv.len) {
                display_usage();
                return error.MissingArgs;
            }
            optind += 1;
            args.output_path = argv[optind];
        } else {
            display_usage();
            std.debug.print("Unknown option: {s}\n", .{argv[optind]});
            return error.InvalidArgs;
        }
        optind += 1;
    }

    // validate and parse positional arguments
    if (argv.len - optind < 2) {
        display_usage();
        return error.MissingArgs;
    }

    args.duration = std.fmt.parseInt(u32, argv[optind], 10) catch {
        display_usage();
        std.debug.print("Invalid duration: '{s}'\n", .{argv[optind]});
        return error.InvalidArgs;
    };
    optind += 1;

    args.ports = argv[optind..];

    return args;
}

pub fn main() !u8 {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    const allocator = gpa.allocator();
    defer _ = gpa.deinit();

    const argv = try std.process.argsAlloc(allocator);
    defer std.process.argsFree(allocator, argv);

    const args = parseArgs(argv) catch {
        return 1;
    };

    // print parsed arguments
    std.debug.print("Verbose: {}\n", .{args.verbose});
    std.debug.print("Output path: {s}\n", .{args.output_path});
    std.debug.print("Duration: {d}\n", .{args.duration});
    std.debug.print("Buffer size: {}\n", .{args.buf_size});
    for (args.ports, 1..) |port, n_port| {
        std.debug.print("Port {d}: {s}\n", .{ n_port, port });
    }
    return 0;
}
```

This may not be the best-looking code, nor the most ergonomic CLI tool, but
it's simple. No extra dependencies, all fits in one file and it is reasonably
structured. Simplicity, I like that.

In comparison to a library, I suppose the biggest disadvantage is that you have
to keep the hardcoded usage help text and the actual parsing code in sync
manually. However, for small programs, that's not hard to do.

So I am pretty satisfied with this, and I intend to use it as a "CLI template"
for small programs.
