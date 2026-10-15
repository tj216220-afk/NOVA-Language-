# Nova Reference

A one-page cheat sheet. For the complete specification see `NOVA_SPEC.md`.

## Comments

    # everything after a hash is ignored

## Variables

    set age to 25
    set name to "Amina"
    set age to age + 1

## Types

| Type      | Examples                     |
|-----------|------------------------------|
| number    | `42`, `3.14`, `-7`           |
| text      | `"hello"`, `"line\nbreak"`   |
| truth     | `true`, `false`              |
| nothing   | `nothing`                    |

## Output and input

    show "Hello"
    set name to ask("Your name? ")

## Operators

Arithmetic:

    +   -   *   /   rem

Comparison:

    ==   !=   <   >   <=   >=

Logic:

    and   or   not

Grouping:

    (2 + 3) * 4

## Conditionals

    if age >= 18 then
        show "Adult"
    otherwise
        show "Minor"
    end

The `otherwise` block is optional.

## Loops

Counting loop:

    repeat 5 times
        show "Hello"
    end

While loop:

    set i to 0
    while i < 10
        show i
        set i to i + 1
    end

Loop control:

    stop    # exit the loop
    skip    # skip to the next iteration

## Functions

    define greet with name
        return "Hello, " + name
    end

    show greet("Nova")

A function without `return` gives back `nothing`.

## Built-in functions

| Function        | Description                                          |
|-----------------|------------------------------------------------------|
| `length(t)`     | Number of characters in a piece of text              |
| `text(x)`       | Convert a number or truth to text                    |
| `number(t)`     | Convert text to a number                             |
| `round(n)`      | Round a number to the nearest whole number           |
| `random(a, b)`  | Random whole number between `a` and `b`, inclusive   |
| `ask(prompt)`   | Print prompt and read one line of input              |

## Type rules

- `number + number` → number
- `text + text` → text
- `text + number` → error. Use `text(x)` to convert.
- Comparing different types with `==` gives `false`.
- A condition must be `true` or `false`. `if 1 then` is an error.

## Errors

Syntax errors are caught before the program runs:

    Syntax Error on line 4, column 1: expected "then" but found the end of the line

Runtime errors include a call trace:

    Runtime Error on line 2: cannot divide by zero

      at divide (line 6)
      at main

## Not in v0.1

Lists, dictionaries, classes, modules, `try`/`catch`, string
interpolation, closures, ternary operators — all reserved for future
versions.