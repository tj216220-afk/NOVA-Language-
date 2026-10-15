# Nova Language Specification — v0.1

Status: Frozen for the v0.1 demo
Interpreter: Written in PHP 8.2+
File extension: .nova

---

## 1. Design Goals

- Reads like plain English instructions.
- One statement per line. No semicolons. No curly braces.
- Blocks open with a keyword and close with `end`.
- Errors are written for humans and always include the line number.
- Strict runtime types — no silent PHP-style coercion.
- v0.1 is deliberately small: only the basics needed to write real beginner programs.

---

## 2. Source Files

- Extension: .nova
- Encoding: UTF-8
- Statements are separated by newlines. Blank lines are ignored.
- Indentation is cosmetic only. It never changes meaning.

---

## 3. Comments

    # This is a comment. It runs to the end of the line.
    set x to 5   # inline comments work too

---

## 4. Values and Types

| Type      | Literals                     | Notes                                   |
|-----------|------------------------------|-----------------------------------------|
| number    | 42, 3.14, -7, 0.5            | Double precision. No separate integers. |
| text      | "Hello", "line\nbreak"       | Escapes: \n \t \" \\                    |
| truth     | true, false                  |                                         |
| nothing   | nothing                      | Absence of a value.                     |

There are no lists, dictionaries, classes, or other data structures in v0.1.

---

## 5. Variables

    set age to 25
    set name to "Amina"
    set age to age + 1        # reassignment uses the same keyword

- `set` creates or updates. There is no separate `let` / `const`.
- Scope is function-level, not block-level. A variable set inside an `if`
  or loop body is still visible after the `end`.
- Reading a variable that was never set is a runtime error, not a silent `nothing`.

---

## 6. Operators

Arithmetic

    +      addition (also joins two pieces of text)
    -      subtraction
    *      multiplication
    /      division
    rem    remainder (modulus)

Comparison (all yield truth)

    ==     equal to
    !=     not equal to
    >      greater than
    <      less than
    >=     greater than or equal to
    <=     less than or equal to

Logic

    and     or     not

`and` and `or` short-circuit.

Grouping

    ( and )

Precedence, loosest to tightest

    or
    and
    not
    ==  !=  >  <  >=  <=
    +  -
    *  /  rem
    unary minus
    function call

---

## 7. Type Rules

- number + number → number
- text + text → text
- text + number → runtime error:
  "Cannot add text and number on line 4. Use text(x) to convert."
- Comparing values of different types (5 == "5") → false, never an error.
- Conditions must be truth. `if 1 then` is a runtime error in v0.1.
  This is deliberate: beginners must be explicit.

---

## 8. Statements

Output

    show "Hello, world"
    show 2 + 3

Input

    set name to ask("What is your name? ")
    show "Hello, " + name

`ask` is a built-in function, not a keyword.

Conditionals

    if age >= 18 then
        show "You may enter."
    otherwise
        show "Sorry, too young."
    end

`otherwise` is optional.

Counting loop

    repeat 5 times
        show "Hello"
    end

While loop

    set count to 0
    while count < 10
        show count
        set count to count + 1
    end

Loop control

    stop     # exit the nearest loop
    skip     # jump to the next iteration

Functions

    define greet with name
        return "Hello, " + name
    end

    define roll_dice
        return random(1, 6)
    end

    show greet("Nova")
    show roll_dice()

- `define NAME with a, b, c` — parameters are comma-separated.
- `define NAME` — no parameters.
- A function with no `return` returns nothing.
- Functions may be recursive.
- Functions are called with NAME(arg1, arg2).

---

## 9. Built-in Functions (v0.1)

| Function        | Description                                                     |
|-----------------|-----------------------------------------------------------------|
| length(x)       | Number of characters in a piece of text.                        |
| text(x)         | Converts a number or truth to text.                             |
| number(x)       | Converts text to a number; runtime error if it cannot.          |
| round(x)        | Rounds a number to the nearest whole number.                    |
| random(a, b)    | Random whole number from a to b, inclusive.                     |
| ask(prompt)     | Prints the prompt and returns the user's typed input as text.   |

---

## 10. Errors

Syntax errors are found before the program runs:

    Syntax Error on line 4: expected "then" but found "end"
        if x > 5
        end

Runtime errors happen while running and include a trace:

    Runtime Error on line 7: cannot divide by zero

      at divide (line 3)
      at main (line 7)

Every error message must:

1. Say Syntax or Runtime
2. Give the line number
3. Explain the problem in plain English
4. Where possible, say how to fix it

---

## 11. Complete Example Program

    # guessing game
    set secret to random(1, 20)
    set guess to 0
    set tries to 0

    while guess != secret
        set guess to number(ask("Guess a number from 1 to 20: "))
        set tries to tries + 1

        if guess < secret then
            show "Too low!"
        otherwise
            if guess > secret then
                show "Too high!"
            end
        end
    end

    show "Correct! You took " + text(tries) + " tries."

---

## 12. Explicitly OUT of Scope for v0.1

Put these in ROADMAP.md and do NOT build them:

- Lists, dictionaries, or any data structure
- Classes / objects
- Modules and imports
- try / catch
- String interpolation
- Default parameters, variadic functions
- Closures / anonymous functions
- Ternary operator
- for i from 1 to 10 numeric ranges
- repeat for each (needs a collection; revisit in v0.2)