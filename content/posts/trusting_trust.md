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

## Stage 10 - *Compilers Learn*

Here's where it gets weird.

Ken makes a subtle but very weird point: when a compiler compiles itself, it can “learn” things that aren't even written in its source code. That knowledge gets baked into the compiler binary, even though it doesn't show up in the code anymore.

Sounds like magic, but here’s a real example to show what's going on.

### Escape Sequences in C

Let’s say you’re building a C compiler from scratch, and you're writing the part that processes string literals:

```c
printf("Hello\n");
```

That `\n` is two characters in the source (`'\'` and `'n'`), but we want it to become a single newline character when the program runs. In C, that’s the character `'\n'`, which is just ASCII code `10`.

So your compiler needs code like this:

```c
c = next();
if (c == '\\') {
    c = next();
    if (c == 'n')
        c = '\n';
}
```

Nice and clean. It says: if you see a backslash followed by an `n`, translate that to a `newline`.

But... there's a catch.

#### How Does the Compiler Know What '\n' Is?

This is where it gets tricky.

You're writing this compiler in C. That means you'll eventually compile this code using a C compiler, possibly the same one you're writing.

But how does your compiler (especially if it's the first one you're bootstrapping) know what `'\n'` even means? You’re literally trying to define `'\n'` using `'\n'`.

That’s a circular definition, and it won’t work the first time around.

#### The Hack: Hardcode It

So here’s what you do: you cheat.

```c
c = 10;  // ASCII code for newline
```

You use the actual number instead of the fancy escape character.

Now you compile the compiler, and the result is a binary that understands this translation. That binary, the compiled compiler, now knows that `'n'` maps to `10` in this context.

So later, you can go back and write:

```c
c = '\n';
```

But here's the thing...

*The Compiler Has Learned Something*

At this point, the source code doesn't say anything about the number `10`. It just says `'\n'`. But the compiler knows what that means, because the version you built earlier hardcoded it.

That knowledge, the fact that `'n'` = `newline` = `ASCII 10`, only exists in the binary now.

The compiler learned it.

It was smuggled in by the earlier version of the compiler, and now the current version acts like it always knew.

#### Why This Matters

This tiny example leads to a massive idea: a compiler can carry information forward that isn't written down anywhere in its current source code.

It can "remember" things it learned in a previous version.

## Stage 11 - *The Trojan horse*

---
