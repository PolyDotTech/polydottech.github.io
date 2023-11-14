---
title: "Blog: What I have learned from making a programming language"
author: gtant
date: 2023-11-13 21:55
categories: [Blog]
---

For a while now, Finn Dyer and I have been working on making a programming language. Here's what our experience has been like.

## How do you make a programming language?

Good question. We didn't know either, so we started by looking at different resources online. Some are great, some not so much, but most of them are only surface level. Something a lot of them mention is that there's two different options for a language: interpreted or compiled. But I'd like to disagree here. I think there's actually three options:

* Interpreted
* Compiled
* Both?

I mean, why not? I'll get into this more later, but all of the tools that you use to make an interpreted language, you need to make a compiled language. Compiled languages are harder to make than interpreted language, but even if your goal is to make a compiled language, you can interpret it at first before your compiler is ready.

What is a compiled or interpreted language anyways? If a language is compiled, it means that a program called a compiler converts human-readable source code into machine code that can be read by the processor directly. Compiled languages are **faster at runtime** (in fact much much much faster), but they have to be compiled before code is run, and compiling is a one way process -- you can't take a compiled program and get source code back. Also, when you compile a program, you compile it for just one platform. If I compile a program for Windows, I can't run the output on a Mac. If a language is interpreted, it means that a program on your computer (that is likely compiled) "interprets" the program you wrote in the interpreted language. It reads through the code and executes it as its reading it. Interpreted languages are slower **at runtime**, but you don't need to compile them before running and the code you distribute is the original source code, so it can run anywhere that has an interpreter. Oh, also, interpreted languages are a lot easier to make than compiled ones. Compilers are complex programs, and you'll need some understanding of assembly in order to make them.

So which did we choose? Well, Finn and I decided to go with a compiled language because we have a [need for speed](https://www.youtube.com/watch?v=4PzpztFJZP8). If you're making a language, its important to think about what you want to use it for. We plan to use our language for games programming, where you only have a few milliseconds to process your frame if you want to have a high framerate. A fast language is key to a fast game.

However, as I mentioned, compiled programming languages are hard, and you need everything that you'll make for an interpreted language before you can start to compiled it. Because of this, we decided we would build up our language with an interpreter so we could start testing it right away, and then we would work on a compiler once the language was complete.

"Alright Gavin, you keep talking about needing things for your language, but I'd like to know what they are." Here's your answer.

### What does a programming language need?

Let's start with the list. Here are the tools that all* programming languages need.

1. Lexer
2. Parser
3. Abstract syntax tree
4. Type-checking

When I said "all programming languages," I lied to you. As you can expect, there are a bunch of exceptions to this list. If your language is interpreted, number 4 is optional (and most choose not to have it). Some languages combine steps 1 and 2, and if your language is really simple, like [Brainfuck](https://en.wikipedia.org/wiki/Brainfuck), you can skip step 3 as well. But it's still a good outline, and you'll most likely use at least three of these.

After those first four steps, interpreted and compiled languages diverge. If you have an interpreted language, all you have to do now is interpret the syntax tree. If your language is compiled, there's a couple more steps.

5. Convert to some kind of intermediate representation (IR), something that looks a little more like assembly
6. Optimize the IR
7. Convert the IR to platform-specific assemblies.

This is just a rough outline, and every compiler does it differently. If you want to skip steps 6 and 7, you can use LLVM, a tool which will handle those steps for you. Finn and I plan to use LLVM when we get there.

### What do you mean, "when we get there?"

To be clear, we're still working on the language. We need to add a lot more features before we even start thinking about the compiler stage.

## How we made our language (so far)

Since our language is interpreted, we had to choose a language to interpret it with. We went with Ocaml, for a couple different reasons:

* Ocaml has tools specifically designed for programming language creation
* Finn had experience working with another programming language that was made in Ocaml
* I've never used Ocaml before and I wanted a chance to learn it

### Lexing and Parsing and Abstract Syntax Tree-ing

Ocaml has builtin tools to handle steps 1 and 2 of language creation (lexing and parsing). I guess I should explain what each of them are before talking about how we did them, so here goes.

*Lexing* is the process of converting the source code into a stream of "tokens." Basically, read through each character and give it some kind of internal representation. Found "<"? That becomes LessThan. Found some text in quotes? Mark that as a string.

*Parsing* is the process of taking those tokens and building a syntax tree with them. Lexing doesn't have any context; it doesn't know what 5 < 10 means, it just converts them into Number LessThan Number. The parser adds context to those tokens. It knows that less than should be surrounded by two expressions. It knows that a word followed by two parenthesis is a function call.

Just for good measure, a *syntax tree* is the internal representation of the code. It's a tree of statements and expressions that can be walked through to interpret the program, or converted to assembly to become compiled.

To handle lexing, we used the builtin tool *ocamllex*. This tool works similarly to an old program made in C called *lex* (shocker!). You tell the program what each token looks like in the file, and then what ocaml object to create when it finds it. Here's a small sample from our language's lexer.

`lexer.mll`
```ocaml
rule token = parse
| '='        { ASSIGN_EQUALS }
| '+'        { PLUS }
| '-'        { MINUS }
| '*'        { TIMES }
| '/'        { DIV }
| "if"       { IF }
| "for"      { FOR }
| "while"    { WHILE }
```

As you can see, the conversion is pretty simple. Just write a string or character that you're looking for, and if it matches, it'll be converted. A couple things need more complicated rules, (comments, strings, etc.), but for the most part its pretty straightforward.

Parsing is slightly more complicated (but not a lot). Ocaml has a builtin tool for this called ocamlyacc, but there's a newer, community-made tool called menhir which we chose to use instead because it has error messages that are easier to understand.

To make a parser, you define a set of rules which look for tokens in a certain order. IN our language, we have a rule for statements (stuff like variable assignments, function calls on their own) and a rule for expressions (math, variables, operators, that kinda stuff).

Here's a snippet:

`parser.mly`
```ocaml
(* this rule parses statements, stuff like variable
   assignments, function calls, and ifs *)
statement:
(* IDENT means identity, which is basically just a word *)
(* expression is another rule which is defined below *)
(* in code, this would look something like varname = 5; *)
| IDENT ASSIGN_EQUALS expression ENDLINE
  { ASSIGN ($1, $3) } (* create an ASSIGN object
                         using the first and third
                         argument *)
(* if  (      true        )      {     print("hi");    } *)
| IF LPAREN expression RPAREN LCURLY list(statement) RCURLY

expression:
(* if we encounter an INT_LIT token, then we create an int
   object using the value stored inside the token *)
| INT_LIT    { Int ($1) }
(* rules can be recursive, so when we write "expression",
   its reapplying the same rule *)
| expression PLUS expression { Plus ($1, $3) }
| expression TIMES expression { Times ($1, $3) }
```

That's just a short snippet from our rules, but I hope it gives you the basic idea of what using menhir is like. You basically just write how the language looks, and then it kinda just figures it out. Super cool.

Now a syntax tree might sound fancy, but it really isn't. To make it, just make an object for each type of statement and expression. For example, if you were making a plus expression, you would make a Plus object that holds two expressions (the two numbers you're adding).

> The way to do this depends on the language. Ocaml makes this easy because we can make a compound type called statement or expression that holds all of the different kinds of statements and expressions:
> ```ocaml
> type expr =
> | Int of int
> | Bool of bool
> | Plus of expr * expr
> | Greater of expr * expr
> | Ternary of expr * expr * expr
> ```
> In an object-oriented language, you would likely make an expression interface, and then make classes for each kind of expression that implement that interface.
{: prompt-tip }

### What happens after?

Alright, so we have our AST. Now, we can compile or interpret. As I mentioned previously, we're not compiling yet, so here comes the interpretation step. To implement this, we chose to pattern-match on the statement and expression types inside functions we named eval_expr (evaluate expression) and eval_statement (evaluate statement). From this point, the rest is pretty straightforward. Loop through the list of statements (or use tail recursion if you're fancy), and match on each statement, doing something based on which statement we have.

Our language currently has variables, if statements, for loops, and while loops, along with functions. An interesting problem we've ran into is scopes--when you enter a function or something else with curly braces, variables defined inside the loop shouldn't still be defined when you exit the curly braces. Instead of just having a map of variable names to their values, we made a stack of those maps, pushing the stack when we enter a function and popping it when we exit.

## What's next?

We do plan to compile the language, but we want to add a couple more things first. We need to add types (our language is supposed to be statically-typed), along with a way to create new ones (classes, interfaces, compound types/enums). After we add types, we are going to add type-checking (the whole reason to have types), and then we will be ready to start writing a compiler.

We're going to write the compiler using LLVM. It might be interesting to create a complete compiler that doesn't use LLVM, but that's outside of the project's current scope.

I'll try to make an update post in the future, so keep an eye out for that. If you have any questions, feel free to contact me at gavin.f.tantleff (at) gmail (dot) com, or DM me on discord (my username is gisforgravity).
