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

Now that the compiler can “learn,” Ken pushes the idea further.

He shows how you can make a compiler that:

1. Recognizes a specific program, like a login utility.
2. Quietly inserts a backdoor when compiling that program (e.g. lets a specific password always succeed).
3. Recognizes itself when compiling its own source code, and reinserts this backdoor logic into the next version of the compiler.

The wild part? You don't need to keep the backdoor code in the compiler source anymore. It's now living in the binary and just keeps copying itself forward.

This works because the compiler doesn't just compile programs, it can compile itself. So once the binary knows how to sneak in a backdoor, it can also sneak in the part that keeps doing that forever.

Let’s walk through what that actually looks like in code.

### Step 1 - *The normal compiler*

The original compiler just compiles lines of code exactly as they're written:

```python
def compile_line(line):
    # Just compile the code as-is
    compile_as_is(line)
```

Nothing sneaky. Just straightforward compilation.

### Step 2 - *Add a backdoor for the login program*

Now we introduce the first Trojan horse: if the compiler sees the login program, it secretly inserts a backdoor.

```python
def compile_line(line):
    if line_matches(line, "login_password_check"):
        # Inject backdoor code instead of compiling the real thing
        compile_as_is("if password == real OR password == backdoor: allow_login")
        return

    # Otherwise, compile normally
    compile_as_is(line)
```

To the user, it still looks like the login program is checking the password normally.

But under the hood, the compiler is replacing that line with something dangerous, it always allows a specific password.


### Step 3 - *Make the Trojan self-replicating*

Here’s the final trick: the compiler now also looks for itself and reinserts the same sneaky logic during compilation.

```python
def compile_line(line):
    if line_matches(line, "login_password_check"):
        # Add backdoor to login program
        compile_as_is("if password == real OR password == backdoor: allow_login")
        return

    if line_matches(line, "compiler_source_code"):
        # Add both backdoors into the new compiler binary
        compile_as_is("add logic: if compiling login, insert the backdoor")
        compile_as_is("add logic: if compiling the compiler, add both of these rules again")

        return

    # Otherwise, compile as usual
    compile_as_is(line)
```

So now:

- If you compile the login program, it gets a backdoor.
- If you compile the compiler source, it injects the logic that adds the backdoor again, even if you’ve removed it from the source code.

In other words:

- First, you teach the compiler to mess with the login program.
- Then, you teach it to also mess with the compiler source code, specifically, to add the same tricks again during self-compilation.

Once this is done, you can delete all traces of the malicious logic from the source code.

But the evil behavior lives on in the compiled compiler, and no one reading the clean source would ever suspect a thing.

The compiler has "learned" to misbehave, and it keeps that knowledge alive even as the source gets scrubbed.

It's a Trojan horse that hides inside another Trojan horse.

And unless you audit the binary itself (which is hard), you'll never know it's there.

---

Alright, it’s all fun and games until someone says, *"I’ll believe it when I see it."* So let's go ahead and implement it.

I've chosen to use Python for this. While Python isn't a compiled language in the traditional sense, it's one of the most readable and beginner-friendly languages out there, which makes it perfect for explaining ideas clearly.

Plus, if you're looking to compile your Python program as a standalone executable, tools like `PyInstaller` make that entirely possible.



---

## Reflections on reflections


