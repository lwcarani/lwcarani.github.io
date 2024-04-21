---
title: Writing a Lisp Interpreter in Python
author: luke
date: 2024-01-04 12:00:00 +0500
categories: [Software Engineering, Coding Projects]
tags: [coding projects, interpreters, python]
---

In this blog post, I'm going to cover another [John Crickett](https://github.com/JohnCrickett) [coding challenge](https://codingchallenges.substack.com/p/coding-challenge-30-lisp-interpreter) that I just completed - building a Lisp interpreter in Python. Lisp is the [second oldest](https://en.wikipedia.org/wiki/Lisp_(programming_language)) high-level language implemented (in 1959, after FORTRAN in 1957), probably most well-known for its fully parenthesized prefix notation (that definitely takes some getting used to). Scott Wlaschin calls Lisp a "galaxy brain" language for introducing the symbolic programming paradigm (if you haven't watched his talk linked [here](https://www.youtube.com/watch?v=0fpDlAEQio4) I highly recommend it). This coding challenge has definitely been the most challenging to date - I had to spend a few hours just familiarizing myself with Lisp syntax so that I could understand it enough to cobble together an implementation with basic features. 

For some interesting historical context on Lisp, and some thoughts on why, perhaps, Lisp is not more popular today, you can read ["Lisp: Good News, Bad News, How to Win Big"](https://dreamsongs.com/WIB.html) by Richard P. Gabriel.

## About

`py-lisp-interpreter` is a basic Lisp interpreter, written in Python. Running `pylisp` from the command line will allow the user to enter the Lisp REPL environment, or execute a .txt file from a provided file path. 

> Throughout this post, I will discuss how I implemented the lexing, parsing, and evaluation phases of my Lisp interpreter. For a helpful overview of the phases of an interpreter (and compiler), please see my previous post [here](https://lwcarani.github.io/posts/compilers-and-interpreters/)
{: .prompt-info }

## Type Definitions

In the header of our .py file containing our interpreter, we'll be explicit about our Lisp types:

```python
Symbol = str            # Implement a Lisp Symbol as a Python str
Number = (int, float)   # Implement a Lisp Number as a Python int or float
Atom = (Symbol, Number) # Implement a Lisp Atom as a Symbol or Number
List = list             # Implement a Lisp List as a Python list
Exp = (Atom, List)      # Implement a Lisp expression as an Atom or List
```

(Thanks to [Peter Norvig](https://norvig.com/lispy.html) for helping me conceptualize how to relate Lisp and Python types).

## Lexer

Typicall, depending on the syntax of a programming language, implementing a lexer can be quite complicated. In this case, however, thanks to Lisp's syntax, and the beauty of Python, all we need to tokenize the input for our simple Lisp interpreter is `str.split()`:

```python 
def tokenize(input: str) -> List[str]:
    """
    Split input string into a list of tokens. Note that we pad
    parentheses with white space before splitting to separate
    parentheses from Atoms
    --> we want '(+ 1 2)' to tokenize to ['(', '+', '1', '2', ')']
    not ['(+', '1', '2)']
    """

    tokenized_input: List[str] = input.replace("(", " ( ").replace(")", " ) ").split()
    return tokenized_input
```

## Parser

Once we've tokenized the input source program, we can generate an abstract syntax tree (AST). We'll use Python lists to express the AST, and lists of lists to denote nested expressions:

```python
def generate_ast(tokens: List) -> List:
    """
    Generate abstract syntax tree from input tokens.

    Example:
    tokenized_input = ['(', 'defun', 'doublen', '(', 'n', ')', '(', '*', 'n', '2', ')', ')']
    -->
    ast = ['defun', 'doublen', ['n'], ['*', 'n', 2]]
    """
    t = tokens.pop(0)

    # start a new sublist everytime we encounter an open parens
    if t == "(":
        ast = []
        # append tokens to sublist until we encounter a close parens
        while tokens[0] != ")":
            ast.append(generate_ast(tokens))
        tokens.pop(0)  # pop off ')'
        return ast
    elif t == ")":
        raise SyntaxError("Mismatched parens.")
    else:
        return atomize(t)

def atomize(token: str) -> Atom:
    """
    Atomize input tokens. Every token is either an int, float, or Symbol.

    Note that
        Symbol := str
        Number := (int, float)
        Atom   := (Symbol, Number)
    """
    try:
        return int(token)
    except ValueError:
        try:
            return float(token)
        except ValueError:
            return Symbol(token)
```

## Evaluation

Once we have the AST, we are ready to evaluate the program! To do this, we need to be able to look up symbols in a mapping from variable name to value. We will call this our `SymbolTable`. We also need to make sure we support locally declared variables. To do this, we will allow for nested mappings, and when we need to look up a variable name in the `SymbolTable`, we can just look up the variable in the innermost mapping, then work outwards until we reach the global scope. 

To implement `SymbolTable`, we will inherit from the Python dict class, and define our own `find` method to check the appropriate scope for where variables are defined. 

```python
import operator as op
import math

class SymbolTable(dict):
"""
A mapping of {variable name: value}
"""
    def __init__(self, params=[], args=[], outer_scope=None):
        self.update(zip(params, args))
        self.outer_scope = outer_scope

    def find(self, var):
        if var in self:
            return self[var]
        elif self.outer_scope is not None:
            return self.outer_scope.find(var)
        else:
            raise NameError(f"NameError: name '{var}' is not defined")


global_symbol_table = SymbolTable()
global_symbol_table.update(
    {
        "+": op.add,
        "-": op.sub,
        "*": op.mul,
        "/": op.truediv,
        "<=": op.le,
        "<": op.lt,
        ">": op.gt,
        ">=": op.ge,
        "!=": op.ne,
        "=": op.eq,
    }
)
# add standard math library operators to symbol_table
global_symbol_table.update(math.__dict__)
```

Now that we have our `SymbolTable` for keeping track of variables in different scopes, we are ready to evaluate:

```python
def eval(x: Exp, st=global_symbol_table):
    """Evaluate the abstract syntax tree"""
    if isinstance(x, Number):
        return x
    elif isinstance(x, Symbol):
        # start with the innermost scope of the symbol table
        # to find symbol definition, searching outer scope until
        # symbol definition is found
        return st.find(x)
    elif x[0] == "if":
        condition, statement, alternative = x[1:4]
        expression = (
            statement
            if eval(condition, SymbolTable(st.keys(), st.values(), st))
            else alternative
        )
        return eval(expression, SymbolTable(st.keys(), st.values(), st))
    elif x[0] == "defun":
        # `func_name`: str
        # `params`: List[str]
        # `func_body`: List
        # Example:
        #   "(defun doublen (n) (* 2 n))" -->
        #   `func_name`: "doublen"
        #   `params`: ["n"]
        #   `func_body`: ["*", 2, "n"]
        func_name, params, func_body = x[1:4]
        st[func_name] = (params, func_body)
        return f"Defined function: {func_name.upper()}"
    elif x[0] == "format":
        if isinstance(x[-1], list):
            fill_val = eval(x[-1], SymbolTable(st.keys(), st.values(), st))
            res = " ".join(str(i) for i in x[2:-1])
        else:
            fill_val = ""
            res = " ".join(str(i) for i in x[2:])
        if "~D~%" in res:
            res = res.replace('"', "").replace("~D~%", str(fill_val))
        else:
            res = res.replace('"', "").replace("~%", str(fill_val))
        return res
    else:
        func_name = x[0]
        func = eval(x[0], SymbolTable(st.keys(), st.values(), st))
        args = [eval(arg, SymbolTable(st.keys(), st.values(), st)) for arg in x[1:]]

        # if `func` is a tuple, it is a user defined function, so update local scoping
        # of symbol table with user-provided parameters
        if isinstance(func, tuple):
            params, func_body = func
            if len(args) != len(params):
                raise ValueError(
                    f'Function "{func_name}" expects {len(params)} arguments, but {len(args)} were provided.'
                )
            st.update(zip(params, args))
            return eval(func_body, SymbolTable(st.keys(), st.values(), st))
        elif isinstance(func, (int, float, str)):
            return func
        else:
            return func(*args)
```

## Instructions
For Windows, create a folder named `Aliases` in your C drive: `C:/Aliases`. Add this folder to PATH. Next, create a batch file that will execute when you call the specified alias. For example, on my machine, I have a batch file named `pylisp.bat` located at `C:/Aliases`, that contains the following script:

```bat
@echo off
echo.
call C:\...\py-lisp-interpreter\pylisp_venv\Scripts\activate.bat
python C:\...\py-lisp-interpreter\main.py %*
```

So now, when I type `pylisp` in the command prompt, this batch file will execute, which in turn, launches the appropriate Python virtual environment, then runs the `py-lisp-interpreter` Python script. 

## Examples

Running `pylisp` from the command line launches a cli option to enter the REPL environment or execute a local file:

```cmd
C:\> pylisp

[?] Would you like to open the REPL environment, or execute a file?: file
 > file
   REPL

Enter the location of the file: test_script.txt
Defined function: HELLO
Defined function: MEANING_OF_LIFE
Defined function: MEANING_OF_LIFE_ANSWER
Defined function: DOUBLEN
Defined function: FIB
Defined function: FACT
Hello Coding Challenge World
42
The meaning of life is 42
The double of 5 is 10
The double of 21 is 42
The double of 107 is 214
Factorial of 5 is 120
Factorial of 6 is 720
Factorial of 7 is 5040
Factorial of 10 is 3628800
Factorial of 12 is 479001600
The 7th number of the Fibonacci sequence is 13
```

After running a local file, the user is then prompted to enter another file to execute, or they can exit the program:

```cmd
[?] Would you like to execute another file?: No
   Yes
 > No
```

If the user chooses to enter the REPL environment, they can then execute simple Lisp expressions:

```cmd
C:\> pylisp

[?] Would you like to open the REPL environment, or execute a file?: REPL
   file
 > REPL

pylisp> (+ 21 21)
42
pylisp> (pow 2 3)
8.0
pylisp> (sin (/ pi 2) )
1.0
pylisp> (sin (/ pi 4) )
0.7071067811865476
pylisp> (defun fact (n) (if (<= n 1) 1 (* n (fact (- n 1)))))
Defined function: FACT
pylisp> (fact 5)
120
pylisp> (fact 10)
3628800
pylisp> (defun add (a b) (+ a b))
Defined function: ADD
pylisp> (add 4 5)
9
pylisp> (add (add 21 21) 42 )
84
pylisp> (add (add 21 21) (fact 5) )
162
```

## Libraries
Just like the `py-wc` and `py-head` projects, everything I needed for `py-lisp-interpreter` was included in Python's Standard Library. For a little more cli functionality, I installed `inquirer`.

## Final Thoughts

This was the most difficult coding challenge for me thus far. I learned a lot about the Lisp programming language, and really came to appreciate it's syntax. Implementing the `eval` function to evaluate the AST proved to be more involved and nuanced than I first thought, and improved my ability to "think recursively."

Finally, if you check my repo for this project, you'll see that I implemented two helper functions that both check if the Lisp program is valid before executing. Specifically, they check if the program has matching parentheses (if parens are not balanced, it is not a valid program). 

For me, using a stack to add and pop parens as we scan the source code was the intuitive way to check balanced parens. After implementing that approach, I enjoyed thinking of a way to solve the same problem in a "functional" paradigm. The below function uses transform (map) reduce to do so. Specifically, `(` get transformed to `1`, `)` get transformed to `-1`, and every other character gets mapped to `0`. Now we can just apply a simple reduction with the `+` operator, and if the result is equal to `0`, we know we had balanced parens in the input source code.

```python
def are_parens_matched_map_reduce(s: str) -> bool:
    """
    A more functional approach to check that all parens are matching.
    Uses built-in Python `map` and `reduce` functions.

    Raises:
        SyntaxError: If `s` is empty or if `s` does
        not have matching opened and closed parentheses
    """
    t: List = tokenize(s)
    if len(t) == 0:
        raise SyntaxError(f"Input string cannot be empty.")
    # make sure that user input starts and end with open/close parens
    elif t[0] != "(" or t[-1] != ")":
        raise SyntaxError(
            f'Input string "{s}" must start and end with open/closed parens.'
        )

    ### transform-reduce
    # transform (map) open parens to 1, close parens to -1,
    # and all other chars to 0, then sum the resulting iterator
    # if all parens are matched, res will be 0, otherwise, throw
    # a SyntaxError, because there are mismatched parens
    d = {"(": 1, ")": -1}
    res = reduce(lambda a, b: a + b, map(lambda x: d.get(x, 0), t))
    if res != 0:
        raise SyntaxError(f'Input string "{s}" contains mismatched parens.')
    else:
        return True
```

## Acknowledgements
Thanks to [John Crickett](https://github.com/JohnCrickett) for the idea from his site, [Coding Challenges](https://codingchallenges.substack.com/p/coding-challenge-30-lisp-interpreter)!

Thanks to [Peter Norvig](https://norvig.com/lispy.html) for helping me when I got stuck, especially in setting up my symbol table for appropriately tracking scope. 

If you happen to peruse my code and notice any bugs or opportunities for optimizations, please let me know!
