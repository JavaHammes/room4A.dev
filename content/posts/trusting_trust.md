+++
date = '2025-07-23T18:15:05+02:00'
draft = true
title = 'Reflections on Trusting Trust'
+++

> *"To what extent should one trust a statement that a program is free of Trojan
> horses? Perhaps it is more important to trust the people who wrote the
> software."*
> - Ken Thompson, *Reflections on Trusting Trust* (1984)

Ken Thompson wrote this in 1984, but it's still just as true today.

In this post, I’ll show you that it’s possible to create a program that does something it **shouldn’t**, even though nothing looks wrong when you read the source code.

No, I’m not a wizard. No, I’m not on drugs.
Yes, this is actually possible.

After this, you’ll think twice before trusting any code you didn't create yourself.

That'll probably be the most fun part, but I'll also take a moment to reflect on the *reflections* (yes, pun intended) and explain why this still matters today.

---

Just like in real life,  when your dentist assures you "this won't hurt" and insists "you don't need to hold your mom's hand".
Well, that doesn't actually mean it won't hurt.

Man, I hate dentists.

In the original paper, Ken proudly declares he's going to present "the cutest program he ever wrote". What a nerd...

He breaks it down into three stages. Here's the first:

## Stage 1 - *Writing the shortest self-reproducing program.*

A self-reproducing program is one that, when compiled and executed, prints out its exact own source code.

Today, we call this kind of program a quine.

As an example:

```python
program = "program = %r\nprint(program %% program)"
print(program % program)
```

when executed, this will output (as you might guess):

```python
program = 'program = %r\nprint(program %% program)'
print(program % program)
```

Exactly the same as the source. You have to admit, that's pretty cool.

How does this work?

Here's the program again:

```python
program = 'program = %r\nprint(program %% program)'
print(program % program)
```

Key parts:
- `%r` gets replaced by the raw string (with quotes and escape characters). That's how it can reproduce itself accurately.
- `%%` escapes the `%` inside the string so Python doesn't treat it as formatting.
- `program % program` inserts the string into itself.

What it does:

It prints:

```python
program = 'program = %r\nprint(program %% program)'
print(program % program)
```

Which is exactly the code that was run. That's what makes it a quine, a self-replicating program.

## Stage 10 - *The chicken and egg problem*

## Stage 11 - *The Trojan horse*


---
