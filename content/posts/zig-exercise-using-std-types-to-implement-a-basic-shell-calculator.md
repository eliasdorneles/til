---
title: "Zig exercise: using std types to implement a basic shell calculator"
date: "2024-07-21T22:26:50+02:00"
tags:
- zig
- standard library
- data structures
- calculator
---

Today I decided to get some practice using Zig's data structures from the
standard library, building an old friend: a `bc` style calculator, implementing
the algorithms to evaluate [reverse Polish
notation](https://en.wikipedia.org/wiki/Reverse_Polish_notation) expressions
and converting from infix to that notation.

These are classic algorithms that use stacks, and it is an exercise I did for
the first time when learning programming back when I was in college. I think
I've implemented some version of it in every major programming language I've
learned over the years, so I feel it's like a little rite of passage for me.

After getting familiar with the stdin/stdout and memory management in Zig as I
talked in the previous post, I suppose it was about time I did my favorite data
structures exercise in it now!

### Calculator using reverse Polish notation

The first version of the calculator, using reverse Polish notation, was fairly
easy to write. The algorithm is quite intuitive and coding it was quite
straightforward.

Here it is:

```zig
const std = @import("std");
const mem = std.mem;
const log = std.log;

const Calculator = struct {
    allocator: std.mem.Allocator,
    context: std.StringHashMap(f128),

    pub fn init(allocator: std.mem.Allocator) Calculator {
        const context = std.StringHashMap(f128).init(allocator);
        return Calculator{ .allocator = allocator, .context = context };
    }

    pub fn deinit(self: *Calculator) void {
        self.context.deinit();
    }

    pub fn evalPostfix(self: *Calculator, expr: []const u8) !f128 {
        var stack = std.ArrayList(f128).init(self.allocator);
        defer stack.deinit();

        var it = mem.tokenizeScalar(u8, expr, ' ');
        while (it.next()) |token| {
            if (mem.eql(u8, token, "")) continue;

            if (token.len == 1 and mem.indexOfAny(u8, token, "+-*/") == 0) {
                if (stack.items.len < 2) {
                    return error.InvalidInput;
                }
                const b = stack.pop();
                const a = stack.pop();
                _ = switch (token.ptr[0]) {
                    '+' => try stack.append(a + b),
                    '-' => try stack.append(a - b),
                    '*' => try stack.append(a * b),
                    '/' => try stack.append(a / b),
                    else => return error.NotImplemented,
                };
            } else {
                const value = try std.fmt.parseFloat(f128, token);
                try stack.append(value);
            }
        }
        if (stack.items.len != 1) {
            return error.InvalidInput;
        }
        return stack.pop();
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

    var calc = Calculator.init(allocator);
    defer calc.deinit();

    while (prompt("calc % ", &line)) |expr| {
        defer line.clearRetainingCapacity();

        const result = calc.evalPostfix(expr) catch |err| {
            try stdout.print("Error: {}\n", .{err});
            continue;
        };
        try stdout.print("{d}\n", .{result});
    } else |err| {
        try switch (err) {
            error.EndOfStream => stdout.print("Bye\n", .{}),
            else => stdout.print("Error: {}\n", .{err}),
        };
    }
}
```

And here is an example session of using it:

```
$ zig run calc_postfix.zig 
calc % 123.45 10 -
113.45
calc % 12 3 * 4 /
9
calc % 12
12
calc % /
Error: error.InvalidInput
calc % 
```

One of the interesting things I learned about `ArrayList` is that the `.pop()`
function doesn't return an error if the list is empty: it panics -- namely,
your program just crashes.


### Calculator using the more familiar infix expressions

For this one, I found out that I had forgotten the algorithm to convert from
infix to RPN and had to look it up. I ended up writing a Python version to
reassure myself I had got it right.

As I can code a lot faster in Python, writing a quick solution in Python
remains an useful technique for me: I prefer that than writing pseudo-algorithm
in the paper, because I can always test my Python code in the shell and
reassure myself that it works as I expect.

Here is the Python code I came up with:

```python
def convert_infix_to_posfix(expr):
    is_operator = lambda x: x in '+-*/'

    precedence = {'+': 0, '-': 0, '*': 1, '/': 1}

    stack = []
    output = []
    for tok in expr:
        if is_operator(tok):
            while stack and stack[-1] in precedence and precedence[stack[-1]] >= precedence[tok]:
                output.append(stack.pop())
            stack.append(tok)
        elif tok == '(':
            stack.append(tok)
        elif tok == ')':
            while stack and stack[-1] != '(':
                output.append(stack.pop())
            if not stack:
                raise ValueError('Unbalanced parentheses')
            stack.pop() # Discard the '('
        else:
            output.append(tok)

    while stack:
        it = stack.pop()
        if it == '(':
            raise ValueError('Unbalanced parentheses')
        output.append(it)
    return output


print(convert_infix_to_posfix('3 - 4 + 2'.split()))
print(convert_infix_to_posfix('3 + 4 * 2 / ( 1 - 5 )'.split()))
```

Armed with that, I went on to implement it in Zig.

My first implementation tokenized the input using `std.mem.tokenizeScalar(u8,
expr, ' ')`, so it couldn't take expressions like `1+2`, you had to separate
everything with spaces.

That is obviously awkward, so to fix that I went on to write my own Tokenizer
that wouldn't require spaces between every symbol. I think my code is a bit
ugly but it does the job nicely.

Without further ado, here is the code:


```zig
const std = @import("std");
const mem = std.mem;
const log = std.log;

fn isOperator(c: u8) bool {
    return c == '+' or c == '-' or c == '*' or c == '/';
}

fn precedence(c: u8) u8 {
    return switch (c) {
        '+' => 1,
        '-' => 1,
        '*' => 2,
        '/' => 2,
        else => 0,
    };
}

const Tokenizer = struct {
    buffer: []const u8,
    index: usize,

    pub fn init(buffer: []const u8) Tokenizer {
        return Tokenizer{ .buffer = buffer, .index = 0 };
    }

    fn skipSpaces(self: *Tokenizer) void {
        while (self.index < self.buffer.len and self.buffer[self.index] == ' ') {
            self.index += 1;
        }
    }

    pub fn next(self: *Tokenizer) ?[]const u8 {
        if (self.index >= self.buffer.len) {
            return null;
        }

        self.skipSpaces();

        const startTok = self.index;

        while (self.index < self.buffer.len) : (self.index += 1) {
            if (self.buffer[self.index] == '(' or self.buffer[self.index] == ')' or
                isOperator(self.buffer[self.index]))
            {
                if (startTok == self.index) {
                    self.index += 1;
                }
                return self.buffer[startTok..self.index];
            }
            if (self.buffer[self.index] == ' ') {
                const endTok = self.index;
                self.skipSpaces();
                return self.buffer[startTok..endTok];
            }
        }

        if (startTok == self.index) return null;

        return self.buffer[startTok..self.index];
    }
};

const Calculator = struct {
    allocator: std.mem.Allocator,
    context: std.StringHashMap(f128),

    pub fn init(allocator: std.mem.Allocator) Calculator {
        const context = std.StringHashMap(f128).init(allocator);
        return Calculator{ .allocator = allocator, .context = context };
    }

    pub fn deinit(self: *Calculator) void {
        self.context.deinit();
    }

    fn evalPostfix(self: *Calculator, tokens: [][]const u8) !f128 {
        var stack = std.ArrayList(f128).init(self.allocator);
        defer stack.deinit();

        for (tokens) |token| {
            if (mem.eql(u8, token, "")) continue;

            if (token.len == 1 and isOperator(token[0])) {
                if (stack.items.len < 2) {
                    return error.InvalidInput;
                }
                const b = stack.pop();
                const a = stack.pop();
                _ = switch (token.ptr[0]) {
                    '+' => try stack.append(a + b),
                    '-' => try stack.append(a - b),
                    '*' => try stack.append(a * b),
                    '/' => try stack.append(a / b),
                    else => return error.NotImplemented,
                };
            } else {
                const value = try std.fmt.parseFloat(f128, token);
                try stack.append(value);
            }
        }
        if (stack.items.len != 1) {
            return error.InvalidInput;
        }
        return stack.pop();
    }

    fn peek(stack: std.ArrayList([]const u8)) []const u8 {
        return stack.items[stack.items.len - 1];
    }

    pub fn eval(self: *Calculator, expr: []const u8) !f128 {
        var stack = std.ArrayList([]const u8).init(self.allocator);
        defer stack.deinit();

        var postfix = std.ArrayList([]const u8).init(self.allocator);
        defer postfix.deinit();

        var it = Tokenizer.init(expr);
        while (it.next()) |token| {
            if (token.len == 1 and isOperator(token[0])) {
                // if it's an operator, we pop any operators from the stack
                // with higher precedence and append them to the postfix result
                while (stack.items.len > 0) {
                    const top = peek(stack);
                    if (isOperator(top[0]) and precedence(token[0]) <= precedence(top[0])) {
                        try postfix.append(stack.pop());
                    } else {
                        break;
                    }
                }
                try stack.append(token);
            } else if (token[0] == '(') {
                try stack.append(token);
            } else if (token[0] == ')') {
                while (stack.items.len > 0) {
                    const top = peek(stack);
                    if (top[0] == '(') break;
                    try postfix.append(stack.pop());
                }
                if (stack.items.len == 0) {
                    return error.UnbalancedParentheses;
                }
                _ = stack.pop(); // pop '('
            } else {
                try postfix.append(token);
            }
        }

        while (stack.items.len > 0) {
            const token = stack.pop();
            if (token[0] == '(') return error.UnbalancedParentheses;
            try postfix.append(token);
        }

        // log.debug("postfix: {s}", .{postfix.items});

        return self.evalPostfix(postfix.items);
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

    var calc = Calculator.init(allocator);
    defer calc.deinit();

    while (prompt("calc % ", &line)) |expr| {
        defer line.clearRetainingCapacity();

        const result = calc.eval(expr) catch |err| {
            try stdout.print("Error: {}\n", .{err});
            continue;
        };
        try stdout.print("{d}\n", .{result});
    } else |err| {
        try switch (err) {
            error.EndOfStream => stdout.print("Bye\n", .{}),
            else => stdout.print("Error: {}\n", .{err}),
        };
    }
}
```

The conversion from infix to postfix notation is done in the `Calculator.eval`
function.

At first I had marked the small functions `isOperator` and `precedence` with
`inline`, but after reading the Zig documentation, I learned that it is meant
mostly for the implication for types, it's actually not an optimization hint
and it's best to leave it upt to the compiler to decide when to inline a
function. I guess that is what `-DReleaseFast` is for, so I left it out.

I have also written some tests for the Tokenizer, but I'd love to figure out a
cleaner way of writing the assertions, this is quite verbose:

```zig
test Tokenizer {
    var it = Tokenizer.init("1 + 22.2 * 3 - 4 / 5");

    var tokens = std.ArrayList([]const u8).init(std.testing.allocator);
    defer tokens.deinit();

    while (it.next()) |token| {
        try tokens.append(token);
    }

    try std.testing.expectEqualStrings(tokens.items[0], "1");
    try std.testing.expectEqualStrings(tokens.items[1], "+");
    try std.testing.expectEqualStrings(tokens.items[2], "22.2");
    try std.testing.expectEqualStrings(tokens.items[3], "*");
    try std.testing.expectEqualStrings(tokens.items[4], "3");
    try std.testing.expectEqualStrings(tokens.items[5], "-");
    try std.testing.expectEqualStrings(tokens.items[6], "4");
    try std.testing.expectEqualStrings(tokens.items[7], "/");
    try std.testing.expectEqualStrings(tokens.items[8], "5");

    tokens.clearRetainingCapacity();

    it = Tokenizer.init("((1+2))");

    while (it.next()) |token| {
        log.debug("token: {s}", .{token});
        try tokens.append(token);
    }

    try std.testing.expectEqualStrings(tokens.items[0], "(");
    try std.testing.expectEqualStrings(tokens.items[1], "(");
    try std.testing.expectEqualStrings(tokens.items[2], "1");
    try std.testing.expectEqualStrings(tokens.items[3], "+");
    try std.testing.expectEqualStrings(tokens.items[4], "2");
    try std.testing.expectEqualStrings(tokens.items[5], ")");
    try std.testing.expectEqualStrings(tokens.items[6], ")");
}
```
