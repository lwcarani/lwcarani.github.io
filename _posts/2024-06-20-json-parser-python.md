---
title: Implementing a JSON Parser - Python
author: luke
date: 2024-06-20 12:00:00 +0500
categories: [Software Engineering, Parsers, Compilers, Coding Projects]
tags: [programming, python, json, coding projects]
---

## Overview

In this blog post, I'm going to cover another [John Crickett](https://github.com/JohnCrickett) [coding challenge](https://codingchallenges.fyi/challenges/challenge-json-parser/) that I completed recently - building a JSON parser. In this post, I'll discuss my implementation in Python. In an upcoming post, I'll discuss my implementation of a JSON parser in OCaml using `ocamllex` and `Menhir`. 

I've found that JSON parsers are a great way to learn about parsing because they're nuanced enough to be interesting, but not so complicated that you need a substantial background in lexing and parsing. 

## Background

JSONs are objects with opening and closing curly brackets `{}`, and key value pairs, where keys are separated from values by colons, and key/value pairs are separated from other key/value pairs by commas. Each key is a double quoted string (single quotes are not allowed), and each value can be a string, number, boolean, `null`, list, or another nested object. These are all valid JSON objects:

Example 1:
```json
{}
```

Example 2:
```json
{
    "key": "valuedfk/$#$#434546[]{}{f[d]fdaj,dfjwkejr"
}
```

Example 3:
```json
{
  "key": "value",
  "key2": "value"
}
```

Example 4:
```json
{
  "key": "value",
  "key-n": 101,
  "key-o": {
    "inner key": "inner value",
    "inner key 2": 42,
    "inner key 3": {
      "inner inner key": 17
    }
  },
  "key-l": [
    "list value",
    1,
    2.43e4,
    3.232,
    [
      "a",
      "b",
      "c"
    ],
    42,
    14,
    false,
    true,
    null
  ]
}
```

We can observe a few patterns in JSON file structure:
- objects and lists have values separated by commas
- key/value pairs are separated by colons
- whitespace can be ignored
- keys must be strings, values can be boolean, null, numbers, lists, strings, or objects

## Interface

We need an interface to use as an entry point to parse JSONs. For this, we'll set up a Python `class` with a function `parse_json` that we can use that will handle input files or strings. As we parse the input stream one character at a time, we'll also define a function to skip over whitespace, and another function to reset our pointer as we consume input characters:

```python
class JsonParser(object):
    def __init__(self, s: str = "") -> None:
        self.ptr = 0
        self.s = s

    # ... more functions here...

    def reset_ptr(self) -> None:
        self.ptr = 0

    def skip_whitespace(self) -> None:
        for char in self.s:
            if char in WHITESPACE:
                self.ptr += 1
            else:
                break

        self.s = self.s[self.ptr :]

        self.reset_ptr()

    def parse_json(self, s: str) -> dict:
        if os.path.exists(s):
            f = open(s, "r")
            self.s = f.read()
            f.close()
        else:
            self.s = s

        self.skip_whitespace()

        # do basic checks before parsing
        if len(self.s) == 0:
            raise JsonParseError("Empty JSON file detected: invalid entry.")
        if self.s[0] != "{":
            raise JsonParseError(
                'JSON file is missing starting "{": invalid entry.'
            )

        res: dict = self.parse_object()

        return res
```

Note that we include a couple simple checks immediately: ensuring that the input string or file actually contains characters, and ensuring that the input string or file with a JSON object actually begins with an open curly bracket (`{`).

We'll also define two helper functions within this class for parsing commas and colons in JSON objects to assist as we process input streams. These allow us to advance our pointer and parse out the key/value pairs, while also ensuring the JSON has a valid structure. Note that trailing commas are not allowed:

```python
def parse_comma(self) -> None:
    if self.s[0] != ",":
        return

    self.s = self.s[1:]
    self.skip_whitespace()

    # if we encounter a closing bracket immediately
    # following a comma, the JSON is invalid
    if self.s[0] == "}":
        raise JsonParseError("Trailing commas are not allowed.")

def parse_colon(self) -> None:
    if self.s[0] != ":":
        return

    self.s = self.s[1:]
    self.skip_whitespace()
```

## Parsing

### Objects

Parsing a JSON file always begins with parsing an object (i.e., curly brackets), so we'll define that first:

```python
def parse_object(self) -> None | dict:
    res = {}
    self.skip_whitespace()

    if self.s[0] != "{":
        return

    self.s = self.s[1:]
    self.skip_whitespace()

    while self.s[0] != "}":
        key = self.parse_string()
        if key is None:
            raise JsonParseError("Keys must be valid strings.")
        self.skip_whitespace()
        self.parse_colon()
        val = self.parse_value()
        self.skip_whitespace()
        self.parse_comma()
        res[key] = val

    # drop final closing bracket
    self.s = self.s[1:]

    return res
```

We begin by instantiating an empty Python dictionary where we'll store our key/value pairs as we parse. We have some basic checks in this function, like making sure valid JSON objects start with an open curly bracket (`{`), and making sure keys are always strings. This also ensures that keys and values are separated by colons, and key/value pairs are separated by commas. 

### Values

Once we've successfully parsed a key, we need to parse the corresponding value. We first attempt to parse the value as a string. If it fails to parse, the `parse_string` function will return `None`, and we will then attempt to parse the value as a number with the `parse_number` function. This continues until we either (1) correctly parse the value (as a string, number, reserved word, object, or list), or (2) fail to parse the value, in which case we raise an error, because the value had an invalid type:

```python
def parse_value(self) -> None | int | float | str | bool | list | dict:
    item = self.parse_string()

    if item is None:
        item = self.parse_number()
    if item is None:
        item = self.parse_reserved_word()
    if item is None:
        item = self.parse_object()
    if item is None:
        item = self.parse_list()
    if item is None:
        raise JsonParseError("Value unable to be parsed: invalid entry.")

    return item
```

Below you'll find the remaining helper functions for parsing numbers, reserved words, lists, and strings:

### Lists

```python
def parse_list(self) -> None | list:
    res = []
    self.skip_whitespace()

    if self.s[0] != "[":
        return

    self.s = self.s[1:]
    self.skip_whitespace()

    while self.s[0] != "]":
        elem = self.parse_value()
        self.skip_whitespace()
        self.parse_comma()
        res.append(elem)

    # drop final closing square bracket
    self.s = self.s[1:]

    return res
```

### Strings

```python
def parse_string(self) -> None | str:
    res = ""

    if self.s[0] != '"':
        return

    self.ptr += 1

    while self.s[self.ptr] != '"':
        char = self.s[self.ptr]
        res += char
        self.ptr += 1

        # no closing quotation encountered, invalid JSON format
        if self.ptr >= len(self.s):
            raise JsonParseError("String is missing close quote.")

    # advance ptr on input string json
    self.s = self.s[self.ptr + 1 :]

    self.reset_ptr()

    return res
```

### Reserved Words ("null", "true", "false")

```python
def parse_reserved_word(self) -> None | str | bool:
    if self.s[:4] == ReservedWords.TRUE.value:
        self.s = self.s[len(ReservedWords.TRUE.value) :]
        return True
    elif self.s[:5] == ReservedWords.FALSE.value:
        self.s = self.s[len(ReservedWords.FALSE.value) :]
        return False
    elif self.s[:4] == ReservedWords.NULL.value:
        self.s = self.s[len(ReservedWords.NULL.value) :]
        return ReservedWords.NULL.value
    else:
        return
```

### Numbers

```python
def parse_number(self) -> None | int | float:
    res = ""
    first_char = self.s[0]
    e_or_dot_counter = 0
    neg_counter = 0

    if not first_char.isdigit() and first_char != "-":
        return

    # parse one char at a time until the string 's' is empty,
    # or until we trigger a reason to break or return 'None'
    for char in self.s:
        if char.isdigit():
            res += char
            self.ptr += 1
        elif char in {"e", "."} and e_or_dot_counter == 0:
            res += char
            e_or_dot_counter += 1
            self.ptr += 1
        elif char == "-" and neg_counter == 0:
            res += char
            neg_counter += 1
            self.ptr += 1
        elif char in WHITESPACE + ["}", ",", "]"]:
            # end of number encountered, break and return the number
            break
        else:
            self.reset_ptr()
            return

    # advance ptr on input string
    self.s = self.s[self.ptr :]
    self.reset_ptr()
    return int(float(res)) if float(res) % 1 == 0 else float(res)
```

## CLI Usage

[For this implementation](https://github.com/lwcarani/json-parser/tree/main/python), I've set it up so that running `jp` from the command line will allow the user to parse a JSON file from a provided file path, or manually type a JSON object in the command line to be parsed. Successful parsing returns `0`, unsuccessful parsing returns `1` along with an error message. 

### Instructions
For Windows, create a folder named `Aliases` in your C drive: `C:/Aliases`. Add this folder to PATH. Next, create a batch file that will execute when you call the specified alias. For example, on my machine, I have a batch file named `jp.bat` located at `C:/Aliases`, that contains the following script:

```bat
@echo off
echo.
python C:\...\json-parser\python\main.py %*
```

So now, when I type `jp` in the command prompt, this batch file will execute, which in turn runs the `json-parser` Python script.

### Examples

Running `jp` from the command line with no input file specified prompts the user to manually input a JSON object to parse:

```cmd
C:\> jp
No input file detected.
Manually type the JSON object you would like to parse:
>
```
So now, the user can type in a JSON object, and the program will return `1` if the JSON object was unable to be parsed (and print an error message), or the program will return `0` if the JSON object was able to be parsed, and print the resulting JSON object:

```cmd
C:\> jp
No input file detected.
Manually type the JSON object you would like to parse:
> { "key1": "value1", "key2": {"inner key": 42} }
{
    "key1": "value1",
    "key2": {
        "inner key": 42,
    },
}
0
```
```cmd
C:\> jp
No input file detected.
Manually type the JSON object you would like to parse:
> { "key1": "value1", "key2": "value2", }
Trailing commas are not allowed.
1
```
```cmd
C:\> jp
No input file detected.
Manually type the JSON object you would like to parse:
> { "key1": "value1", key2: "value2" }
Keys must be valid strings.
1
```
```cmd
C:\> jp
No input file detected.
Manually type the JSON object you would like to parse:
> { "key1": true, "key2": false, "key3": 1e10 }
{
    "key1": True,
    "key2": False,
    "key3": 10000000000,
}
0
```
You can also pass in a `-t` or `--tests` flag to run the test suite:
```cmd
C:\> jp -t
....................................................................................................................................................................................
----------------------------------------------------------------------
Ran 180 tests in 0.118s

OK
No input file detected.
Manually type the JSON object you would like to parse:
> 
```
```cmd
C:\> jp --tests
....................................................................................................................................................................................
----------------------------------------------------------------------
Ran 180 tests in 0.118s

OK
No input file detected.
Manually type the JSON object you would like to parse:
> 
```
You can run tests, and provide a path to the JSON file you want to parse:
```cmd
C:\> jp --tests step3/valid.json
....................................................................................................................................................................................
----------------------------------------------------------------------
Ran 180 tests in 0.119s

OK
{
    "key1": True,
    "key2": False,
    "key3": "null",
    "key4": "value",
    "key5": 101,
}
0
```
Or, you can just parse a single JSON file:
```cmd
C:\> jp step4/valid3.json
{
    "key": "value",
    "key-n": 101,
    "key-o": {
        "inner key": "inner value",
        "inner key 2": 42,
        "inner key 3": {
            "inner inner key": 17,
        },
    },
    "key-l": [
        "list value",
        1,
        2,
        3,
        [
            "a",
            "b",
            "c",
        ],
        42,
        14,
        False,
        True,
        "null",
    ],
}
0
```
And finally, you can pass in multiple JSON files to parse:
```cmd
C:\> jp step3/valid.json step4/valid2.json step4/valid.json
{
    "key1": True,
    "key2": False,
    "key3": "null",
    "key4": "value",
    "key5": 101,
}
0
{
    "key": "value",
    "key-n": 101,
    "key-o": {
        "inner key": "inner value",
    },
    "key-l": [
        "list value",
    ],
}
0
{
    "key": "value",
    "key-n": 101,
    "key-o": {
    },
    "key-l": [
    ],
}
0
```
```cmd
C:\> jp step1/invalid.json step2/invalid2.json step3/invalid.json step4/invalid.json
JSON file is missing starting "{": invalid entry.
1
Keys must be valid strings.
1
Value unable to be parsed: invalid entry.
1
Value unable to be parsed: invalid entry.
1
```

## Acknowledgements
Thanks to [John Crickett](https://github.com/JohnCrickett) for the idea from his site, [Coding Challenges](https://codingchallenges.fyi/challenges/challenge-json-parser)!

Feedback, bug reports, issues, and pull requests welcome!
