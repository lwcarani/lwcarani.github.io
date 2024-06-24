---
title: 10 Reasons I Love OCaml
author: luke
date: 2024-06-22 12:00:00 +0500
categories: [Programming]
tags: [programming, ocaml]
---

I think compilers are pretty cool. So cool in fact, that when I separate from the military next year, I'm really hoping to become a compiler engineer. And if not that, at least a software engineer that works adjacent to the compiler engineering space. To that end, I decided to learn OCaml since it's a popular choice when writing compilers, and because I wanted to learn a functional programming language. 

> As I prepared to learn OCaml, I was fortunate to discover Michael Clarkson's amazing book ["OCaml Programming: Correct + Efficient + Beautiful"](https://cs3110.github.io/textbook/cover.html). I cannot say enough great things about it - it is wonderfully accessible and beautifully written, and it's FREE! I learned so much not just about OCaml, but about programming best practices and computer science fundamentals. I'm a big Michael Clarkson fan now, because of this book.
{: .prompt-info }

I love learning different programming languages (and paradigms) because it gives me a different perspective and appreciation for the different ways you can design a language. I find programming language design and the tradeoffs you have to make extremely fascinating. What is the right default behavior? How do you progressively disclose the complexity of the language to the user? What is the right level of abstraction? Can you write a high-level language that still gives the user low-level control when they need/want it? These are some of the things I was thinking about as I started learning OCaml. I still have a long way to go until I can call myself proficient in the language, but at this point, I wanted to share 10 things about OCaml that I really enjoy.

## 1. Strong static typing

OCaml's type system catches many errors at compile-time, enhancing code reliability. This leads to fewer runtime errors and easier maintenance.

```ocaml
let add (x : int) (y : int) : int = x + y
(* This function only accepts integers, preventing type errors *)
```

## 2. Powerful type inference

One of the first things any OCaml developer will tell you about OCaml is the amazing type checker. OCaml can automatically deduce types without explicit annotations, reducing boilerplate while maintaining type safety, making the code cleaner and more readable. Returning to our example from above, we don't even need to specify the types of the parameters or the return value. The OCaml type checker is able to infer the types automatically, and will throw a compile-time error if you try to pass in anything other than integers:

```ocaml
let add x y = x + y
(* OCaml infers that 'add' takes in integers as arguments and returns an int *)
```

In this case, the OCaml type checker can infer type `int` because OCaml does **not** allow for overloading on operators. `+` is used for adding two integers, `+.` is used for adding two float values. I find that in addition to allowing for powerful type-checking, this also improves code readability. 

## 3. Algebraic data types

These allow for the creation of complex data structures that are easy to reason about. They're particularly useful for modeling domain-specific concepts. Consider the following example of a binary tree data type. A node carries a data item of type `'a` and has a left and right subtree. A leaf is empty. 

```ocaml
type 'a tree =
| Leaf
| Node of 'a * 'a tree * 'a tree

(* the code below constructs this tree:
         4
       /   \
      2     5
     / \   / \
    1   3 6   7
*)
let t =
  Node(4,
    Node(2,
      Node(1, Leaf, Leaf),
      Node(3, Leaf, Leaf)
    ),
    Node(5,
      Node(6, Leaf, Leaf),
      Node(7, Leaf, Leaf)
    )
  )
```

## 4. Pattern matching

I **LOVE** pattern matching in OCaml. Once I got used to it, I found it to be intuitive, powerful, readable, and just plain great. OCaml's pattern matching is expressive and exhaustive, making it easy to handle complex data structures. It's particularly useful for working with algebraic data types. Continuing our example from above, we can compute the depth of a tree by using pattern matching with algebraic data types to perform different computations depending on if we match on a `Leaf` or `Node`. The following function simply says if we get to a leaf, return `0` (because this doesn't contribute to the overall depth of the tree), otherwise, add `1` to the max depth of the left and right subtrees: 

```ocaml
let rec depth t = 
  match t with
  | Leaf -> 0
  | Node (_, left, right) -> 1 + Stdlib.max (depth left) (depth right)
```

## 5. No null value

By design, OCaml does not include a `null` or `nil` value. Instead, OCaml has `Option`, which is like a box that either contains `Some` value, or `None`. Idiomatically, we use pattern matching to check and "unwrap" the contents of the "box". We can uses this to gracefully handle things like possibly empty lists, in a very readable format:

```ocaml
(** [extract o] converts the int option to a string value
    if the option contains a value, otherwise, if the option
    is empty (contains None), it returns the empty string "". *)

let extract (o: int option) : string =
  match o with
  | Some i -> string_of_int i
  | None -> ""

(** [safe_hd lst] is Some x if lst is not empty, 
    otherwise, returns None for empty lists. *)
let safe_hd lst = 
  match lst with
  | [] -> None
  | x :: xs -> Some x

(* This test passes *)
assert ("1" = extract (safe_hd [1;2;3;]));;
```

> A couple of fun notes here - Rust options (among other language features) were influenced from the ML family of languages (like OCaml), and in fact, the first Rust compiler was actually written in OCaml.
{: .prompt-info }

## 6. Immutability by default

OCaml encourages immutable data structures, leading to more predictable and easier to understand code, and allowing the compiler to make more optimizations. Learning OCaml has given me a greater appreciation for how programming languages choose to handle mutability vs. immutability by default, and the trade-offs of each.

```ocaml
let x = [1; 2; 3]
let y = 0 :: x  (* Prepending an element to x creates a new list, it does not modify x *)
```

## 7. First-class modules

Modules in OCaml are first-class citizens, allowing for powerful abstraction and code organization. This feature enables flexible and reusable code architectures. Often a module will implement some data structure. For example, below is a module for stacks implemented as linked lists, pulled directly from [Michael Clarkson's book](https://cs3110.github.io/textbook/chapters/modules/modules.html):

```ocaml
module ListStack = struct
  (** [empty] is the empty stack. *)
  let empty = []

  (** [is_empty s] is whether [s] is empty. *)
  let is_empty = function [] -> true | _ -> false

  (** [push x s] pushes [x] onto the top of [s]. *)
  let push x s = x :: s

  (** [Empty] is raised when an operation cannot be applied
      to an empty stack. *)
  exception Empty

  (** [peek s] is the top element of [s].
      Raises [Empty] if [s] is empty. *)
  let peek = function
    | [] -> raise Empty
    | x :: _ -> x

  (** [pop s] is all but the top element of [s].
      Raises [Empty] if [s] is empty. *)
  let pop = function
    | [] -> raise Empty
    | _ :: s -> s
end
```

Modules are essential for managing complexity in large software systems by enabling modular programming, where code is divided into separate, independently developed components. Modules allow developers to use local reasoning and focus on discrete parts of the system without needing to understand every detail, while providing clear interfaces that act as contracts between clients and implementers. This abstraction and separation of concerns makes it easier to build, maintain, and modify large programs, especially when multiple developers are involved.

## 8. Efficient compilation

OCaml's compilation process is known for its efficiency and the high performance of the resulting executables. The OCaml compiler uses a multi-stage compilation process that includes sophisticated optimizations. OCaml's static typing and immutability by default also enable further optimizations. The result is often code that runs nearly as fast as equivalent C programs, but with the safety guarantees of a high-level language.

One useful feature that the OCaml compiler enables is tail-call optimization (I am aware that other languages support this, but I first learned about it while using OCaml). As a functional language, idiomatic OCaml relies heavily on recursive functions. A naive implementation of a recursive function could easily exceed the call stack size limit. If the programmer writes their function with the recursive call in the tail position, however, the compiler can make the optimization of "recycling" the stack frame, greatly reducing the stack space requirements from $O(n)$ (linear) to $O(1)$ (constant).

## 9. utop REPL environment
utop is an improved toplevel REPL for OCaml. It offers several features that enable rapid and enjoyable interactive development in OCaml:

- Enhanced readability with syntax highlighting and proper indentation
- Multi-line editing capabilities
- Command history and completion for module names, values, and functions
- Real-time type information as you write code
- The ability to load compiled modules and use them interactively

Here's a simple example of using utop:

```ocaml
$ utop
utop # let square x = x * x;;
val square : int -> int = <fun> (* Specifying that the value 'square' is a function takes in an int and returns an int *)

utop # square 5;;
- : int = 25

utop # #use "myfile.ml";;  (* Load definitions from a file *)

utop # Module.function_from_myfile;;  (* Use loaded definitions *)
```

utop's features make it an excellent tool for exploring the language, testing small code snippets, and even for educational purposes when learning OCaml. Its immediate feedback and rich information display has allowed me to quickly iterate on code as I'm learning and building in OCaml.

## 10. Parametric polymorphism

Finally, OCaml supports writing functions and data structures that work with multiple types. This enables the creation of highly reusable and generic code without sacrificing type safety.

```ocaml
let id x = x  (* 'id' works with any type *)
let l = List.map id [1; 2; 3] (* 'id' will except integers *)
let s = id "hello" (* 'id' also excepts strings *)
```
