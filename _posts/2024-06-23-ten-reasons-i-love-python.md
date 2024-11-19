---
title: 10 Reasons I Love Python
author: luke
date: 2024-06-23 12:00:00 +0500
categories: [Programming]
tags: [programming, python]
image:
  path: /assets/img/wise_snake.png
---

I love Python. And who doesn't? It's pervasive for a reason. Sure, it's a dynamically typed interpreted language that pushes type errors that other languages would catch at compile-time to run-time, but that also enables extremely rapid scripting and iterating when developing in Python. When I was first learning programming, I thought I had to pick - I'm either a statically typed guy or dynamically typed guy. "I like interpreted languages" or "I like compiled languages". I quickly learned that (1) these distinctions are not always as black and white as they seem, and (2) although it is extremely useful to have deep expertise in a language, it's important to recognize that programming languages are just tools, and we as software engineers and programmers need to make sure we're picking the best tool for the job. 

Anyway, back to why I love Python! First and foremost, I am forever grateful to Python for re-kindling my love of coding, and seducing me into the world of programming. When I took my first programming course in undergrad, it was taught in Java. Although I think Java can be a fine first language for some people to learn, for me, it was extremely challenging and dense. I just couldn't get past all of the boilerplate to learn, appreciate, and understand programming and computer science fundamentals like data structures, algorithms, object-oriented programming, etc. The result was that I left undergrad telling myself I just wasn't cut out to be a programmer, coder, software engineer, or anything in that line-of-work. Fortunately, a few years after undergrad, [Jake Taylor](https://github.com/jakee417) encouraged me to learn Python with him, and after muddling through a few tutorials I was hooked! I hadn't realized the "other" programming languages out there could be so different, I sort of just assumed everything was like Java. After a couple years I came back to languages like Java, C++, OCaml, and TypeScript, and because I now had a strong programming foundation, I was able to pick them up much more easily. So yeah, all that to say, I owe a lot to the Python programming language and ecosystem for leading me down the software engineering career path. Here are just a few things I really enjoy about the language:

## 1. Progressive disclosure of complexity

This is a principle I first learned about from a podcast where Chris Lattner was talking about designing the Swift programming language (Lex Fridman Podcast #131 - Chris Lattner: The Future of Computing and Programming Languages, starting at 27:45). To get started, you don't need a `main` function, no imports, no classes, you can simply type `print("Hello World")` and click run, and the program will run:

```python
print("Hello World")
# "Hello World" prints to console
```

## 2. Readability and Clean Syntax

Python's clean and readable syntax makes it easy to write and understand code. Its use of indentation for block delimitation enhances code structure. No curly braces, no semi colons to denote statement termination, etc., just whitespace! 

```python
def greet(name):
    print(f"Hello, {name}!")

greet("Python lover")
# "Hello, Python lover!" prints to console
```

## 3. Versatility

Python is a versatile language used in web development, data science, AI, automation, and more. Famously, [Instagram is full-stack Python](https://instagram-engineering.com/tagged/python), and almost certainly any data scientist job or machine learning engineer role will require you to know Python. 

```python
### Web development with Flask ###
from flask import Flask
app = Flask(__name__)

@app.route('/')
def hello():
    return "Hello, World!"

### Data analysis with pandas ###
import pandas as pd
df = pd.read_csv('data.csv')
print(df.describe())
```

## 4. Large Standard Library

Python comes with a comprehensive standard library, reducing the need for external dependencies.

```python
import os
import datetime
import json
import random
from collections import Counter

# prints the current working directory location
# Example: 'C:\\Users\\lukec\\GitHub'
print(os.getcwd())

today = datetime.date.today()
data = {"lucky_number": random.randint(1, 100)}
json_data = json.dumps(data)

# Example: "Today is 2024-06-23. Your lucky number: {"lucky_number": 23}"
print(f"Today is {today}. Your lucky number: {json_data}")  

Counter("Mississippi")  # Counter({'i': 4, 's': 4, 'p': 2, 'M': 1})
```

## 5. Rich Ecosystem of Third-Party Packages

Python's package manager, pip, gives access to thousands of high-quality, open source libraries. Notably in data science you have packages like `numpy`, `pandas`, and `scikit-learn`, and for web development, you have [`Flask`](https://flask.palletsprojects.com/en/3.0.x/installation/#python-version) and [`Django`](https://www.djangoproject.com/)

```python
# Install a package: pip install numpy
import numpy as np

# make a 5x5 matrix and fill with 1s
np.ones((5,5))
"""
array([[1., 1., 1., 1., 1.],
       [1., 1., 1., 1., 1.],
       [1., 1., 1., 1., 1.],
       [1., 1., 1., 1., 1.],
       [1., 1., 1., 1., 1.]])
"""
```

This can be a bit overwhelming since anyone can upload and host a package on [PyPI](https://pypi.org/) - you just have to do your due diligence to figure out which packages are useful, well-written, and maintained. 

## 6. Dynamic Typing

Python's dynamic typing allows for flexible and rapid development.

```python
x = 5
print(type(x))  # <class 'int'>

x = "Now I'm a string"
print(type(x))  # <class 'str'>
```

## 7. List, Dict, and Set Comprehensions

Comprehensions provide a concise way to create, modify, or filter lists, dicts, or sets based on existing lists, dicts, or sets. Sure it's just syntactic sugar, but once you get used to the syntax, you'll wonder how you ever lived without it in other languages:

```python
# list comprehension
numbers = [1, 2, 3, 4, 5]
squares = [n**2 for n in numbers if n % 2 == 0]
print(squares)  # [4, 16]

# set comprehension
names = {
    "Alice", 
    "Bob", 
    "Charlie", 
    "Dan", 
    "Edgar", 
    "Fred", 
    "Gertrude", 
    "Herbert", 
    "Ida", 
    "Joan", 
    "Joan", 
    "Joan"
}
names = {n for n in names if "a" in n}  # {'Charlie', 'Dan', 'Joan', 'Edgar', 'Ida'}

# dict comprehension
names = ['Charlie', 'Dan', 'Joan', 'Edgar', 'Ida']
heights_inches = [72, 80, 60, 65, 63]
profiles = {name: {'inches': height, 'centimeters': height * 2.54} for name, height in zip(names, heights_inches)}
"""
{
    'Charlie': {
        'inches': 72, 
        'centimeters': 182.88
    }, 'Dan': {
        'inches': 80, 
        'centimeters': 203.2
    }, 'Joan': {
        'inches': 60, 
        'centimeters': 152.4
    }, 'Edgar': {
        'inches': 65,
         'centimeters': 165.1
    }, 'Ida': {
    'inches': 63,
     'centimeters': 160.02
    }
}
"""
```

## 8. First-Class Functions

Functions in Python are first-class citizens, allowing for a functional programming paradigm when necessary / it makes sense:

```python
def apply(func, value):
    return func(value)

def double(x):
    return x * 2

result = apply(double, 5)
print(result)  # 10
```

## 9. Generator Expressions

Generators allow for efficient iteration over large datasets without loading everything into memory. Here we compute the square of numbers from 0 to 999999 while deferring computation until we actually need the value (by calling `next`):

```python
# Generate squares of numbers up to 1 million
squares = (x**2 for x in range(1000000))
print(next(squares))  # 0
print(next(squares))  # 1
```

## 10. Context Managers

Context managers, using the `with` statement, simplify resource management. In the example below, we don't have to open the file, write contents, and then close the file stream, we can just use a context manager to handle that for us:

```python
with open('example.txt', 'w') as file:
    file.write('Hello, Python!')
# File is automatically closed after the block 
```
