---
title: "Pratt parsing, aka top-down precedence parsing, aka precedence climbing"
date: "2024-07-23T22:12:09+02:00"
tags:
- parsing
- tokenize
- Python
- Pratt parsing
- recursive parsing
---
Since last weekend I've been toying around implementing a [calculator in
Zig](https://github.com/eliasdorneles/zig-calc).

One of my favorite algorithms I learned in college from my data structures
classes were the shunting yard algorithm to parse and evaluate expressions,
converting to the reverse Polish notation. Those exercises were clearly an
important part of my becoming a programmer, as I've implemented those in most
major programming languages I've learned over the years.

Anyway, I realized I needed something better for a calculator with variables,
so I did some research on alternatives and that's how I ended up finding about
Top-Down Precedence Parsing algorithm described by [Vaughan
Pratt](https://en.wikipedia.org/wiki/Vaughan_Pratt) in 1973 -- also known as
Pratt parsing.

I learned it mostly through reading [this nice blog post
explanation](https://journal.stuffwithstuff.com/2011/03/19/pratt-parsers-expression-parsing-made-easy/),
reading its [accompanying code](https://github.com/munificent/bantam) and
writing my own implementation in Python. For me, the best way to really learn
something is by _doing_, rather than just reading.

I really like this algorithm, because it is not only quite simple, but it's
pretty much extensible: there is a nice separation of concerns between the core
parsing implementation and the definition of the "parselets".

It's like, it is the sweet spot between recursive descent parsing and say, an
LL parser: in a recursive descent parser, you don't have a "grammar view" of
the language, the grammar is pretty much implicitly hardcoded in the
implementation. On the other hand, an LL parser lets you write a grammar
separately from the parsing code, but is a lot of more complex and even getting
a grammar right isn't very straightforward.

The top-down precedence parsing discovered by Pratt lets you easily extend the
parser by adding new "parselets", which are put in one of two hash tables: one
for the "prefix parselets", and the other for the "infix parselets". If you
push it, you can use it to make a language that lets you change add new syntax
at runtime!

So yeah, yay hash tables, yay Pratt parsing!

I want to incorporate it into my calculator program in Zig later, but hey, one
thing at the time! Porting it to Zig will be a whole new exercise, and Python
is great for quickly whiping out some code to test out the idea.

Reading the [Wikipedia page for Operator-precedence
parser](https://en.wikipedia.org/wiki/Operator-precedence_parser) I realized
that their description of "Precedence climbing" looked A LOT like to what I had
just implemented. It turns out, [they are actually the same
algorithm](https://www.oilshell.org/blog/2016/11/01.html) -- apparently, one of
those cases of parallel discovery.

Anyway, without further ado, here is the code:

```Python
import re
from enum import StrEnum


class TokenType(StrEnum):
    PLUS = "PLUS"
    MINUS = "MINUS"
    MUL = "MUL"
    DIV = "DIV"
    LPAREN = "LPAREN"
    RPAREN = "RPAREN"
    NUMBER = "NUMBER"
    IDENTIFIER = "IDENTIFIER"
    ASSIGNMENT = "ASSIGNMENT"


class Token:
    def __init__(self, value):
        if not value:
            raise ValueError("Token value cannot be empty")
        self.value = value
        self.type = self._get_type()

    def _get_type(self):
        operators = {
            "+": TokenType.PLUS,
            "-": TokenType.MINUS,
            "*": TokenType.MUL,
            "/": TokenType.DIV,
            "(": TokenType.LPAREN,
            ")": TokenType.RPAREN,
            "=": TokenType.ASSIGNMENT,
        }
        if self.value in operators:
            return operators[self.value]

        if self.value.isdigit():
            return TokenType.NUMBER

        if self.value[0].isalpha():
            return TokenType.IDENTIFIER

        raise ValueError(f"Unknown token type for {self.value}")

    def __repr__(self):
        return f"Token(value={self.value}, type={self.type})"


def tokenize(text):
    return [Token(x) for x in re.split(r"(\s+|[-+*/=()])", text) if x.strip()]



class PrattParser:
    """
    Parser implementing top-down operator precedence parsing, also known as
    Pratt parsing.
    """
    tokens: list

    def __init__(self, input_text):
        self.tokens = tokenize(input_text)
        self.prefix_parselets = {}
        self.infix_parselets = {}

    def parse(self, precedence=0):
        cur_token = self.tokens.pop(0)

        prefix_parser = self.prefix_parselets.get(cur_token.type)

        if not prefix_parser:
            raise ValueError(f"Could not parse {cur_token!r} (prefix)")

        # prefix parselets signature: (parser, token):
        left = prefix_parser(self, cur_token)

        while precedence < self.get_precedence():
            cur_token = self.tokens.pop(0)
            infix_parser = self.infix_parselets.get(cur_token.type)

            if not infix_parser:
                raise ValueError(f"Could not parse {cur_token!r} (infix)")

            # infix parselets signature: (parser, left_expr, token):
            left = infix_parser(self, left, cur_token)

        return left

    def consume(self, token_type):
        if not self.tokens:
            raise ValueError(f"Expected {token_type!r} but got nothing")
        token = self.tokens.pop(0)
        if token.type != token_type:
            raise ValueError(f"Expected {token_type!r} but got {token!r}")
        return token

    def get_precedence(self):
        if not self.tokens:
            return 0
        infix_parser = self.infix_parselets.get(self.tokens[0].type)
        return infix_parser.precedence if infix_parser else 0


# Here we define the parselets that we're going to use
# in our parser:

# Prefix Parselets:
def parse_scalar(_parser, token):
    return (token.type.value, token.value)


def build_prefix_op_parselet(precedence):
    def parse_operator(parser, token):
        right = parser.parse(precedence)
        return ('UNOP_' + token.type.value, right)
    return parse_operator


def parse_group(parser, _token):
    expr = parser.parse()
    parser.consume(TokenType.RPAREN)
    return expr


# Infix Parselets:
class BinaryOperatorParselet:
    def __init__(self, precedence):
        self.precedence = precedence

    def __call__(self, parser, left, token):
        right = parser.parse(self.precedence)
        return ('OP_' + token.type.value, left, right)


# Here is where we construct our actual parser.
# It's not a grammar, but it isn't too far from one either,
# it gives a bird's-eye view of the language.
class Parser(PrattParser):
    def __init__(self, raw_tokens):
        super().__init__(raw_tokens)
        self.prefix_parselets[TokenType.IDENTIFIER] = parse_scalar
        self.prefix_parselets[TokenType.NUMBER] = parse_scalar
        self.prefix_parselets[TokenType.MINUS] = build_prefix_op_parselet(10)
        self.prefix_parselets[TokenType.PLUS] = build_prefix_op_parselet(10)
        self.prefix_parselets[TokenType.LPAREN] = parse_group

        self.infix_parselets[TokenType.ASSIGNMENT] = BinaryOperatorParselet(1)

        self.infix_parselets[TokenType.PLUS] = BinaryOperatorParselet(5)
        self.infix_parselets[TokenType.MINUS] = BinaryOperatorParselet(5)

        self.infix_parselets[TokenType.MUL] = BinaryOperatorParselet(7)
        self.infix_parselets[TokenType.DIV] = BinaryOperatorParselet(7)


if __name__ == "__main__":
    # some test inputs
    print(Parser('1 + 2 * 3').parse())
    print(Parser('(1 + 2) * 3').parse())
    print(Parser('+ 1 + 2').parse())
    print(Parser('+ a + 22').parse())
    print(Parser('1 - 2 - 3').parse())

    # for extra interactive testing
    while True:
        try:
            line = input()
            print(Parser(line).parse())
        except (KeyboardInterrupt, EOFError):
            break
```
