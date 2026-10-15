# Nova — Language and Implementation Reference

Audience: engineers who need to understand every rule of the language and
every part of the interpreter well enough to extend or debug it.

Companion documents:
- `NOVA_SPEC.md` — the frozen v0.1 language specification
- `REFERENCE.md` — one-page cheat sheet for end users
- `ROADMAP.md` — features deliberately excluded from v0.1

---

# Part 1 — Language Rules

Every rule below is enforced by the interpreter. Where a rule is enforced
differs — the "Enforced at" line tells you which layer is responsible.
Layers are: **Lexer**, **Parser**, **Runtime**.

## 1.1 Source files

- Extension: `.nova`
- Encoding: UTF-8. A leading UTF-8 BOM is stripped silently.
- Line endings: `\n`, `\r\n`, and `\r` are all normalised to `\n` by the lexer.
- Statements are separated by newlines. There are no semicolons.
- Indentation is cosmetic. It never changes meaning.

Enforced at: **Lexer** (BOM, line endings) and **Parser** (statement termination).

## 1.2 Comments

A `#` begins a comment that runs to the end of the line. Comments are
stripped by the lexer and never reach the parser.

```nova
# whole-line comment
set x to 5   # inline comment
```

Enforced at: **Lexer**.

## 1.3 Types

Exactly four value types exist. There are no user-defined types in v0.1.

| Type      | Literals                    | Notes                                       |
|-----------|-----------------------------|---------------------------------------------|
| `number`  | `42`, `3.14`, `-7`, `0.5`   | IEEE 754 double. No separate integer type.  |
| `text`    | `"hello"`, `"line\nbreak"`  | Escapes: `\n` `\t` `\"` `\\`                |
| `truth`   | `true`, `false`             |                                             |
| `nothing` | `nothing`                   | Absence of a value.                         |

Enforced at: **Lexer** (literal recognition), **Runtime** (type checks).

## 1.4 Variables

```nova
set age to 25
set name to "Amina"
set age to age + 1
```

Rules:

- `set` both creates and updates. There is no `let` / `const` distinction.
- The name must be an identifier: letters, digits, underscores; must start
  with a letter or underscore.
- **Scope is function-level, not block-level.** A variable set inside an
  `if`, `while`, or `repeat` body is visible after the `end`.
- Reading a variable that was never set is a runtime error, not a silent
  `nothing`.

Enforced at: **Parser** (`set` expects `IDENTIFIER` then `TO` then
expression), **Runtime** (undefined variable check).

```nova
if true then
    set x to 10
end
show x        # prints 10 — the variable leaked out of the block
```

## 1.5 Operators

### Arithmetic

| Op    | Meaning                  | Operand types | Result  |
|-------|--------------------------|---------------|---------|
| `+`   | addition                 | number, number| number  |
| `+`   | concatenation            | text, text    | text    |
| `-`   | subtraction              | number, number| number  |
| `*`   | multiplication           | number, number| number  |
| `/`   | division                 | number, number| number  |
| `rem` | remainder (modulus)      | number, number| number  |

`+` is overloaded. It only works on two numbers OR two pieces of text.
Mixing them is a runtime error:

```nova
show 5 + "5"    # Runtime Error: cannot add number and text — use text(x)
show "5" + 5    # Runtime Error: cannot add text and number — use text(x)
show "5" + text(5)   # prints 55
```

Division by zero and `rem` by zero are runtime errors.

### Comparison

All comparison operators yield `truth`.

| Op    | Meaning                    |
|-------|----------------------------|
| `==`  | equal                      |
| `!=`  | not equal                  |
| `<`   | less than                  |
| `>`   | greater than               |
| `<=`  | less than or equal         |
| `>=`  | greater than or equal      |

Rules:

- Numbers compare numerically. Text compares lexicographically.
- Comparing values of **different types** with `==` or `!=` returns `false`
  (or `true` for `!=`). It is never an error.
- Comparing different types with `<`, `>`, `<=`, `>=` **is** a runtime error.

```nova
show 5 == "5"    # false — different types, always false
show 5 != "5"    # true
show 5 < "5"     # Runtime Error: cannot compare number and text
```

### Logic

| Op    | Meaning             |
|-------|---------------------|
| `and` | logical AND         |
| `or`  | logical OR          |
| `not` | logical negation    |

Both `and` and `or` short-circuit. Both operands must be `truth`.

```nova
show true and false    # false
show true or false     # true
show not true          # false

# Short-circuit demonstration
define side_effect
    show "evaluated"
    return true
end

show false and side_effect()   # prints nothing besides false — the call is skipped
show true or side_effect()     # prints nothing besides true — the call is skipped
```

### Grouping

`(` and `)` group expressions and override precedence.

```nova
show (2 + 3) * 4     # 20
show 2 + 3 * 4       # 14
```

### Precedence

Loosest to tightest:

1. `or`
2. `and`
3. `not` (unary)
4. `==` `!=` `<` `>` `<=` `>=`
5. `+` `-`
6. `*` `/` `rem`
7. unary `-`
8. function call

Within the same level, evaluation is left to right.

Enforced at: **Parser** (precedence, associativity), **Runtime** (type
compatibility).

## 1.6 Type rules

These are the strict rules that make Nova different from PHP.

- A condition (`if`, `while`) must be `truth`. `if 1 then` is a runtime error.
- Arithmetic requires numbers. No coercion of text to number.
- `+` on text requires both sides to be text. No implicit conversion.
- `not`, `and`, `or` require `truth` operands.
- Function call arguments are **not** checked against any declared type —
  there are no declared types in v0.1. Each builtin enforces its own rules.

Enforced at: **Runtime**.

## 1.7 Statements

### Output

```nova
show "Hello, world"
show 2 + 3
show age
```

`show` takes exactly one expression. Its result is converted to text and
written to stdout with a trailing newline.

How values are stringified:

| Value        | Output           |
|--------------|------------------|
| `5.0`        | `5`              |
| `3.14`       | `3.14`           |
| `-7.0`       | `-7`             |
| `"hello"`    | `hello`          |
| `true`       | `true`           |
| `false`      | `false`          |
| `nothing`    | `nothing`        |

Whole numbers drop the decimal. `1.5e300` (non-finite) prints in PHP's
native float notation.

Enforced at: **Parser** (one expression then newline), **Runtime** (value
stringification).

### Input

There is no input keyword. Input is a builtin function:

```nova
set name to ask("Your name? ")
show "Hello, " + name
```

`ask` writes the prompt without a newline, reads one line from stdin,
strips the trailing newline, and returns the result as `text`.

Enforced at: **Runtime** (builtin).

### Conditionals

```nova
if age >= 18 then
    show "Adult"
otherwise
    show "Minor"
end
```

Rules:

- Condition must evaluate to `truth`.
- `otherwise` is optional.
- Nested `if` blocks are allowed.
- `then` is required. `if x > 5` followed by a newline is a syntax error.

Enforced at: **Parser** (grammar), **Runtime** (condition type).

### Counting loop

```nova
repeat 5 times
    show "Hello"
end
```

The count must be a `number`. It is truncated to an integer with `(int)`
before looping. `repeat 2.9 times` runs twice.

Enforced at: **Parser** (grammar), **Runtime** (count type).

### While loop

```nova
set i to 0
while i < 10
    show i
    set i to i + 1
end
```

Condition is re-evaluated before each iteration. Must be `truth`.

Enforced at: **Parser** (grammar), **Runtime** (condition type).

### Loop control

```nova
stop    # exit the innermost loop
skip    # jump to the next iteration of the innermost loop
```

Rules:

- Both can appear only inside a `while` or `repeat` body.
- Using them at the top level is a runtime error.
- They affect the **innermost** enclosing loop only.

Enforced at: **Parser** (grammar), **Runtime** (loop context check).

### Functions

```nova
define greet with name
    return "Hello, " + name
end

define roll
    return random(1, 6)
end

show greet("Nova")
show roll()
```

Rules:

- `define NAME` — no parameters.
- `define NAME with a, b, c` — one or more parameters, comma-separated.
- Function bodies may contain any statement, including other `define` blocks.
- A function without an explicit `return` returns `nothing`.
- `return` with no value returns `nothing`.
- Arguments are evaluated left to right **before** the call.
- Arity is checked at call time. Mismatch is a runtime error.
- Nova functions are not first-class. You cannot assign them to variables.
- Recursion is supported.
- Functions see the global scope for reads but writes inside the function
  create locals. There is no closure capture.

Enforced at: **Parser** (grammar), **Runtime** (arity, scope, return).

### Expression statements

A bare expression on its own line is allowed. Its value is discarded.

```nova
greet("Nova")     # calls greet but does nothing with the result
```

This exists mainly to make function calls usable as statements.

Enforced at: **Parser** (default statement fallback).

## 1.8 Built-in functions

Exactly six builtins exist in v0.1. They cannot be redefined — user
functions take precedence, but defining a function named `length` will
shadow the builtin. Avoid it.

| Function        | Args | Returns  | Rules                                           |
|-----------------|------|----------|-------------------------------------------------|
| `length(t)`     | 1    | number   | Argument must be text. Returns character count. |
| `text(x)`       | 1    | text     | Accepts any value. Uses `show`'s stringifier.   |
| `number(t)`     | 1    | number   | Argument must be text parseable as a number.    |
| `round(n)`      | 1    | number   | Argument must be a number. Nearest integer.     |
| `random(a, b)`  | 2    | number   | Both numbers. Returns integer in `[a, b]`.      |
| `ask(prompt)`   | 1    | text     | Argument must be text. Reads one line.          |

Errors:

```nova
length(5)          # Runtime Error: length() expects text, got number
number("abc")      # Runtime Error: cannot convert "abc" to a number
random(10, 5)      # Runtime Error: random() range is empty (10 to 5)
```

Enforced at: **Runtime**.

## 1.9 Errors

### Syntax errors

Detected before execution. Formatted as:

```
Syntax Error on line L, column C: <plain English message>
```

Examples:

```
Syntax Error on line 1, column 9: expected "then" but found the end of the line
Syntax Error on line 2, column 1: the "if" starting here was never closed — missing "end"
Syntax Error on line 1, column 7: unexpected '=' — did you mean '==' (equality) or 'to' (assignment)?
Syntax Error on line 4, column 12: unterminated text — this string was never closed with a "
```

Rules the parser enforces on messages:

1. Always include line and column.
2. Always name the expected token if one is missing.
3. For an unclosed block, blame the **opener**, not the EOF.
4. For an unknown escape or stray character, suggest the likely fix.

Enforced at: **Lexer** and **Parser**.

### Runtime errors

Detected while running. Formatted as:

```
Runtime Error on line L: <message>

  at <innermost frame> (line N)
  at <outer frame>
  at main
```

The trace lists frames **innermost first**, matching Python and Ruby.

Examples:

```
Runtime Error on line 2: variable 'y' is not set

  at main
```

```
Runtime Error on line 2: cannot divide by zero

  at divide (line 6)
  at main
```

```
Runtime Error on line 5: "stop" used outside a loop

  at main
```

Exit code is `1` on any error, `0` on success.

Enforced at: **Runtime**.

---

# Part 2 — Implementation Architecture

## 2.1 Pipeline

```
source.nova
    │
    ▼
┌─────────┐   Token[]   ┌────────┐   Statement[]   ┌─────────────┐
│  Lexer  │ ──────────▶ │ Parser │ ──────────────▶ │ Interpreter │
└─────────┘             └────────┘                 └─────────────┘
     │                       │                            │
     │ SyntaxError           │ SyntaxError                │ RuntimeError
     ▼                       ▼                            ▼
   stderr                  stderr                       stderr
```

The three stages are strictly separated:

- **Lexer** knows nothing about grammar. `show show show` is a perfectly
  valid token stream.
- **Parser** knows nothing about values. It builds the tree without ever
  evaluating anything.
- **Interpreter** knows nothing about source text. It only sees AST nodes
  that carry line and column for error reporting.

This separation is the reason a new feature touches at most three files:
one new `TokenType` case if it needs a keyword, one new AST node, one
parser method, one interpreter method.

## 2.2 Folder map

```
bin/nova                CLI entry point. Requires autoloader, calls Application.
docs/                   Spec, reference, roadmap, this file.
examples/               Sample .nova programs.
src/
  Ast/
    Node.php            Marker interface: line(), column().
    Expression.php      Marker interface extending Node.
    Statement.php       Marker interface extending Node.
    AstPrinter.php      Debug dumper. Not used by the interpreter.
    Expression/         10 expression node classes + 2 enums.
    Statement/          10 statement node classes.
  Cli/
    Application.php     Command dispatcher.
    Repl.php            Interactive loop.
  Error/
    SyntaxError.php     Thrown by Lexer and Parser. Carries line + column.
  Lexer/
    TokenType.php       Enum of every token kind + keyword lookup table.
    Token.php           A single token with lexeme, literal, line, column.
    Lexer.php           Source string → Token[].
  Parser/
    Parser.php          Token[] → Statement[]. Recursive descent.
  Runtime/
    Environment.php     Variable storage. Linked-list of scopes.
    RuntimeError.php    Thrown at runtime. Carries frames.
    StopSignal.php      Exception used to unwind out of loops.
    SkipSignal.php      Exception used to jump to the next iteration.
    ReturnSignal.php    Exception used to return from a function.
    Builtins.php        The six builtin functions.
    Interpreter.php     AST walker. The core of the runtime.
```

## 2.3 The Lexer

**Responsibility:** turn a string into a flat array of `Token`.

**Position tracking.** Two integers track position:

- `$pos` — byte offset into the source.
- `$lineStart` — byte offset of the start of the current line.

Column is derived as `$pos - $lineStart + 1`. Every token stores the line
and column where it *starts*.

**Order of operations in `scanToken`:**

1. Read one character with `advance()`.
2. If whitespace → discard, return.
3. If `\n` → emit `NEWLINE`, increment `$line`, reset `$lineStart`.
4. If `#` → consume to end of line (but not the `\n`).
5. If digit → `number()`.
6. If `"` → `text()`.
7. If letter or `_` → `identifier()` (which may produce a keyword).
8. Otherwise, match single and double-character symbols:
   - `=`: if followed by `=` → `EQUAL`; else a helpful error.
   - `!`: if followed by `=` → `NOT_EQUAL`; else a helpful error.
   - `<`: if followed by `=` → `LESS_EQUAL`; else `LESS`.
   - `>`: if followed by `=` → `GREATER_EQUAL`; else `GREATER`.
   - `(` `)` `,` `+` `-` `*` `/` map directly.

**Numbers.** Scan digits, then optionally `.` followed by at least one
digit. `5.` is rejected at lex time with a specific message. The literal
stored on the token is a `float`.

**Text.** Scan until the closing `"`. Process escapes `\n`, `\t`, `\"`,
`\\`. Any other escape is a `SyntaxError` naming the offending sequence.
A raw newline inside a string is a `SyntaxError`. Reaching EOF before the
closing quote is a `SyntaxError`. The literal stored is the decoded string
(escapes already applied).

**Identifiers and keywords.** Scan `[A-Za-z_][A-Za-z0-9_]*`. Then call
`TokenType::fromKeyword($word)`. If the enum returns a case, use it.
Otherwise emit `IDENTIFIER`. For `true` and `false`, the literal field is
set to the appropriate PHP `bool`. For `nothing`, the literal is `null`.

**Keyword table.** `TokenType::fromKeyword` is the single source of truth.
Adding a keyword requires exactly two changes: a new enum case and a new
line in the `match` inside `fromKeyword`.

**EOF sentinel.** The lexer always appends a final `EOF` token. The parser
never has to bound-check; it can always peek a valid token.

## 2.4 The Parser

**Responsibility:** turn `Token[]` into `Statement[]`.

**Style.** Recursive descent for statements, precedence climbing for
expressions. Every grammar rule maps to one method.

**Statement dispatch.** `statement()` switches on the current token type
and calls the corresponding method. Anything unrecognised falls through to
`expressionStatement()`, which is how a bare function call like
`greet("Nova")` becomes a valid statement.

**Block parsing.** `parseBlock(array $terminators)` is the workhorse.

- Skip blank lines (consecutive `NEWLINE` tokens).
- Stop at EOF.
- Stop when the current token's type is one of `$terminators`.
- Otherwise parse a statement and loop.

An `if` is parsed by calling `parseBlock([OTHERWISE, END])` for the then
branch, then optionally consuming `OTHERWISE` and calling
`parseBlock([END])`. A `while` uses `parseBlock([END])`. This is why a
block's contents are just a nested array of statements on the node.

**Unclosed blocks.** `expectBlockEnd($construct, $opener)` checks whether
we reached EOF first. If yes, it throws a `SyntaxError` anchored at the
**opener's** line and column, not at EOF. That is why the error message
for a missing `end` points at the `if` line.

**Expression parsing.** Six methods, one per precedence level, loosest
first:

```
expression()  → orExpression()  → andExpression()  → equality()
              → comparison()    → term()          → factor()
              → unary()         → primary()
```

Each level:

1. Parse the next-tighter level for the left operand.
2. Loop: while the current token is an operator at this level, consume it,
   parse the next-tighter level for the right operand, and wrap both in a
   `Binary` node.

This gives left associativity for free. `1 - 2 - 3` becomes
`Binary(Binary(1, -, 2), -, 3)`.

**`primary()`** handles atoms:

- Literals: `NUMBER`, `TEXT`, `TRUE`, `FALSE`, `NOTHING`.
- `IDENTIFIER`: if next token is `(`, parse a call; otherwise an identifier.
- `(`: parse a full expression, require `)`.

Anything else raises a syntax error naming the unexpected token.

**Calls.** `callExpression()` parses a comma-separated argument list
inside parentheses. The function name is a plain string, not an
expression — Nova has no first-class functions.

**Statement termination.** Every statement method ends with
`expectNewline()`. This accepts `NEWLINE` or `EOF`. Anything else is a
syntax error. That is what produces the "expected end of line but found
X" errors when two statements are on one line.

## 2.5 The AST

**Design principles:**

- Every node is `final` and `readonly`.
- Every node implements `Node` and returns `line()` and `column()`.
- Expressions implement `Expression`. Statements implement `Statement`.
- No behaviour on nodes. All execution lives in the interpreter.

**Expression nodes:**

| Node             | Fields                     |
|------------------|----------------------------|
| `NumberLiteral`  | `float $value`             |
| `TextLiteral`    | `string $value`            |
| `BooleanLiteral` | `bool $value`              |
| `NothingLiteral` | —                          |
| `Identifier`     | `string $name`             |
| `Unary`          | `UnaryOp $op`, `Expression $operand` |
| `Binary`         | `Expression $left`, `BinaryOp $op`, `Expression $right` |
| `Call`           | `string $name`, `Expression[] $args` |

**Statement nodes:**

| Node                    | Fields                                              |
|-------------------------|-----------------------------------------------------|
| `ShowStatement`         | `Expression $value`                                 |
| `SetStatement`          | `string $name`, `Expression $value`                 |
| `IfStatement`           | `Expression $condition`, `Statement[] $thenBranch`, `Statement[] $elseBranch` |
| `WhileStatement`        | `Expression $condition`, `Statement[] $body`        |
| `RepeatTimesStatement`  | `Expression $count`, `Statement[] $body`            |
| `StopStatement`         | —                                                   |
| `SkipStatement`         | —                                                   |
| `ReturnStatement`       | `?Expression $value`                                |
| `DefineStatement`       | `string $name`, `string[] $params`, `Statement[] $body` |
| `ExpressionStatement`   | `Expression $expression`                            |

**Why line and column live on every node.** When the interpreter hits a
division by zero, it needs to say "on line N". It only has the AST node in
hand, not the source. Every node carrying its own position means error
reporting is trivial.

**`AstPrinter`.** A debug-only class. Given a `Statement[]`, prints a
human-readable tree. Used by `nova ast`. Not referenced by the runtime.

## 2.6 The Interpreter

**Responsibility:** walk the AST and execute it.

**Environment.** A linked list of scopes.

```php
final class Environment
{
    private array $values = [];
    public function __construct(private readonly ?Environment $parent = null) {}
    public function define(string $name, mixed $value): void
    public function get(string $name): mixed   // returns UNDEFINED sentinel
}
```

- `define()` always writes to the **current** scope.
- `get()` checks the current scope, then walks up to the parent.
- An undefined variable returns the sentinel `Environment::UNDEFINED`
  (a unique string) rather than `null`, because `null` is a legal value
  (`nothing`).

**Scope model.** Nova is a mix of Python and JavaScript `var`:

- The global environment is created once, on the first `run()` call.
- Every user function call creates a fresh `Environment` whose parent is
  the **global** environment, not the caller's environment.
- That means functions cannot see their caller's locals — no lexical
  closure, no dynamic scoping. It also means functions can read globals.
- Writes inside a function create locals. To update a global, you'd need a
  `global` keyword, which v0.1 doesn't have.

**Statement dispatch.** `executeStatement()` is a chain of `instanceof`
checks. Order matters only for correctness, not for speed — there are
ten node types. Each branch either produces a side effect (show, set,
define), recurses on a body, or throws a control-flow signal.

**Expression dispatch.** `evaluate()` mirrors the statement dispatcher.
Binary operators route through `evalBinary()`, which:

1. Special-cases `AND` and `OR` for short-circuit.
2. Evaluates both operands otherwise.
3. Dispatches to `add()`, `divide()`, `remainder()`, `equals()`,
   `compare()`, or inline for `-`, `*`.

**Type checks.** Helper methods `requireNumber()` and
`requireBoolean()` throw a `RuntimeError` with a caller-supplied context
string. Context strings are what make error messages readable:
*"subtraction requires a number, got text"* beats *"type error"*.

**Control flow via exceptions.** Nova uses three PHP exceptions as
signals:

- `StopSignal` — thrown by `stop`, caught by the nearest loop.
- `SkipSignal` — thrown by `skip`, caught by the nearest loop.
- `ReturnSignal` — thrown by `return`, caught by `callUserFunction`.

These are not errors. They are control transfer. Using PHP exceptions is
the simplest way to unwind out of nested blocks without threading a state
machine through every `executeStatement` call.

If a signal escapes all the way up to `run()`, it is a real error —
`stop` outside a loop, `skip` outside a loop, or `return` at top level.
`run()` catches them and converts them into `RuntimeError`s.

**Function calls.** `callUserFunction()`:

1. Checks arity against the `DefineStatement`.
2. Evaluates all arguments in the **caller's** environment.
3. Creates a new `Environment` with the global as parent.
4. Binds parameters into the new environment.
5. Swaps `$this->env` to the new environment, saves the old one.
6. Pushes a frame onto `$this->frames`.
7. Executes the body statement by statement.
8. If a `ReturnSignal` is thrown, uses its `$value` as the result.
9. In a `finally` block, restores the previous environment and pops the
   frame — whether we returned normally, via `return`, or via an error.

**Error frames.** `$this->frames` is an array of `{name, line}` entries.
The first entry is always `main`. Every function call pushes one. When a
`RuntimeError` is constructed, it captures the current frames. When it is
formatted, the frames are reversed so the innermost appears first.

**Recursion cost.** Nova's tree-walking interpreter recurses in PHP on
every user function call. `fib(25)` is fine. `fib(40)` will hit PHP's
default stack limits and crash. This is acceptable for v0.1.

## 2.7 The CLI

`Application::run($argv)` dispatches on `$argv[1]`:

| Command   | Behaviour                                                 |
|-----------|-----------------------------------------------------------|
| `run`     | Lex, parse, interpret. Print errors to stderr. Return 0/1.|
| `repl`    | Start the interactive loop.                               |
| `tokens`  | Lex only. Print token dump. Useful for debugging the lexer.|
| `ast`     | Lex and parse. Print the tree. Useful for debugging the parser.|
| `version` | Print the version.                                        |
| `help`    | Print usage.                                              |

Every command that reads a file goes through `readSource()`, which
returns `null` (and prints a friendly error) if the file is missing.

`run` catches `SyntaxError` and `RuntimeError` separately. Syntax errors
are reported before any output; runtime errors after whatever output the
program produced up to that point.

## 2.8 The REPL

`Repl::run()` reads stdin line by line. It maintains two pieces of state:

- `$buffer` — accumulated source text of the current multi-line entry.
- `$depth` — nesting level of `if` / `while` / `repeat` / `define` blocks.

After every line, `depthDelta()` scans the line and returns +1, -1, or 0:

- A line consisting solely of `end` → -1.
- A line starting with `if `, `while `, `repeat `, or `define ` → +1.
- Anything else → 0.

Depth does **not** count nested constructs on one line, because Nova has
one statement per line by design. It also does not understand the block
structure — it only sees keywords. This is fast and wrong for exotic
input, but always correct for well-formed input.

When depth returns to zero, the REPL:

1. Lexes and parses the buffer.
2. If the parse produced exactly one `ExpressionStatement`, wraps it in a
   `ShowStatement`. This is why typing `1 + 2` at the prompt prints `3`
   rather than nothing.
3. Calls `interpreter->run($program)` **with the same interpreter
   instance** used for previous entries. That is how variables and
   functions persist across REPL lines — the interpreter holds the
   environment.

`quit`, `exit`, and `:q` terminate the loop at depth zero.

If the parse fails, the buffer is discarded and the REPL continues. If a
runtime error occurs, it is printed and the REPL continues. Only Ctrl-D
(EOF on stdin) exits cleanly.

## 2.9 Error format contract

Every error the interpreter prints respects this contract:

| Prefix                    | Source     | Exit code |
|---------------------------|------------|-----------|
| `Syntax Error on line`    | Lexer/Parser | 1       |
| `Runtime Error on line`   | Interpreter | 1        |

Every message is plain English, no exception class names leak through.
Every message names a line, and where relevant a column. Every runtime
error with a trace lists frames innermost first, ending with `at main`.

This is not a stylistic preference. It is the product. A language whose
errors are unreadable has failed regardless of how good the grammar is.

## 2.10 Adding a new feature — checklist

To add a new keyword-based statement (say, `assert`):

1. Add `ASSERT = 'ASSERT'` to `TokenType` and a line in `fromKeyword`.
2. Create `src/Ast/Statement/AssertStatement.php` implementing
   `Statement`. Include `line()` and `column()`.
3. Add a method `assertStatement()` to `Parser` and a case in the
   `statement()` match.
4. Add a branch in `Interpreter::executeStatement()`.
5. If the statement affects control flow, add a signal class and a
   matching `catch` in `run()` and in the enclosing constructs.
6. Add an example program under `examples/`.
7. Add a section to `NOVA_SPEC.md`.

That is the complete surface area for adding a statement. Adding an
expression is the same shape: one enum case if it's a new operator, one
node, one parser method, one interpreter method.

## 2.11 What is deliberately missing

These are design choices, not oversights. Each is documented in
`ROADMAP.md`:

- **No lists or dictionaries.** Collections need their own type-checking
  and printing rules. Deferred to v0.2.
- **No closures.** Functions always run in the global scope. Makes scope
  analysis trivial.
- **No first-class functions.** The parser's `Call` node holds a string,
  not an expression. Adding first-class functions means changing that and
  every call site.
- **No `try` / `catch`.** Nova's own error handling is via the host
  runtime, not the language. Deferred to v0.3.
- **No string interpolation.** `"Hello, {name}"` requires lexer state and
  a small expression sub-parser. Deferred.
- **No tail-call optimisation.** Recursion depth is bounded by PHP's
  stack. v1.0 would add bytecode and a trampoline if it mattered.

## 2.12 Performance notes

- Lexing and parsing are O(n) in source size.
- Interpreter is O(n) in AST nodes, plus the cost of running loops.
- Every variable read walks the environment chain. Chains are at most two
  deep in normal code (function scope → global). This is fine.
- Every function call allocates a fresh `Environment` and pushes a frame.
  This is the only nontrivial allocation in the runtime.
- `fib(25)` runs in roughly the same order of magnitude as the equivalent
  Python, which is to say: adequate, not fast.

If you profile and find a hotspot, the first thing to try is caching the
result of `strlen($this->source)` in the lexer instead of recomputing it
on every `isAtEnd()` check. That alone cuts a noticeable percentage off
large files.