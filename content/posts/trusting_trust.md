+++
date = '2025-07-24T16:15:05+02:00'
draft = false
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

---

Just like in real life,  when your dentist assures you "this won't hurt" and insists "you don't need to hold your mom's hand".
Well, that doesn't actually mean it won't hurt.

Man, I hate dentists.

In the original paper, Ken proudly declares he's going to present "the cutest program he ever wrote".

He breaks it down into three stages. Here's the first:

## Stage 1 - Writing the shortest self-reproducing program.

A self-reproducing program is one that, when compiled and executed, prints out its exact own source code.

Today, we call this kind of program a [quine](https://en.wikipedia.org/wiki/Quine_(computing)).

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

## Stage 10 - Compilers Learn

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

It says: if you see a backslash followed by an `n`, translate that to a `newline`.

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

## Stage 11 - The Self-Replicating Backdoor

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

### Step 2 - Add a backdoor for the login program

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

### Step 3 - Make the backdoor self-replicating

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

To check out the full code, head over to the [Github repo for this post](https://github.com/JavaHammes/trusting_compilers).

We’ll try to hack a login script not only because Ken did, but because it's a concrete and intuitive example. Everyone understands what a password check is, and everyone understands how dangerous it is if that check can be secretly subverted.

```python
def login():
    user_input = input("Password: ")

    if user_input == "1234":
        print("Access granted")
    else:
        print("Access denied")

login()
```

### Step 1 - The compiler

We'll start by creating a basic compiler script. Of course, Python isn't compiled in the traditional sense, but we can simulate the process by building an executable from a `.py` file using `pyinstaller`.

Below is a minimal wrapper around `pyinstaller` that compiles a given script into a standalone executable:

```python
import subprocess
import sys


def compile(filename, output_name):
    try:
        subprocess.run([
            "pyinstaller",
            "--onefile",
            "--name", output_name,
            filename
        ], check=True)
        print("Executable created in the 'dist' folder.")
    except subprocess.CalledProcessError as e:
        print("Error during compilation:", e)


if __name__ == "__main__":
    if len(sys.argv) < 3:
        print("Usage: compiler.py <filename> <output_name>")
    else:
        compile(sys.argv[1], sys.argv[2])
```

This script takes a `<filename>` and compiles it into an executable named `<output_name>` using `pyinstaller`. It’s essentially a convenient command-line wrapper.

### Step 2 - The backdoored compiler

We'll modify the compile function to behave differently when it detects our specific login script. If it recognizes the exact code, it silently injects a backdoor, an extra condition that allows access using a second, hidden password.

Specifically, we’ll add the line:

```python
elif user_input == "4321":
    print("Access granted")
```

Here's what the modified `compile` function looks like:

```python
...
def compile(filename, output_name):
    target_code = """def login():
    user_input = input("Password: ")

    if user_input == "1234":
        print("Access granted")
    else:
        print("Access denied")"""

    buggy_code = """def login():
    user_input = input("Password: ")

    if user_input == "1234":
        print("Access granted")
    elif user_input == "4321":
        print("Access granted")
    else:
        print("Access denied")"""

    with open(filename, "r") as f:
        content = f.read()

    if target_code in content:
        content = content.replace(target_code, buggy_code)
    else:
       # Do nothing
...
```

This version of the compiler reads the source file, checks for an exact match of the login script, and if found, replaces it with a version that includes the hidden password logic. It then compiles the modified version.

### Step 3 - The self-replicating backdoored compiler

Now we take it a step further, the compiler itself becomes infected.

In this version, if the compiler is compiling *itself* (i.e., if the filename contains `"compiler"`), it injects the same backdooring logic into the resulting compiler binary. That way, even if someone reads the source and compiles a "clean" version of the compiler (see Step 1), the backdoor survives.

This is the core of Ken Thompson's original "trusting trust" attack: the malicious logic perpetuates itself, even if all visible source code is clean.

Here’s the key part of the updated compiler script:

```python
TEMPLATE = """TEMPLATE = {T!r}

TARGET_CODE = {target!r}

BUGGY_CODE = {buggy!r}

... (compiler logic continues here) ...
"""
```

This `TEMPLATE` contains the full source of the compiler as a string, including itself, enabling it to reconstruct and embed the backdoor recursively.

Inside the `compile()` function, we check if the file being compiled is the compiler:

```python
if "compiler" in filename:
    source = TEMPLATE.format(T=TEMPLATE, target=TARGET_CODE, buggy=BUGGY_CODE)
    ...
    subprocess.run(["pyinstaller", "--onefile", "--name", output, tmp_filename])
```

If so, we inject the `TEMPLATE`, `TARGET_CODE`, and `BUGGY_CODE` directly into the output, meaning the new compiler binary will behave exactly the same: backdooring login scripts and self-replicating when compiled again.

The result? A fully self-replicating backdoored compiler. Even if someone writes a clean compiler from scratch and compiles it using our infected compiler, the new binary will carry forward the malicious logic invisibly.

**Have I promised too much at the beginning?**

Again, feel free to [check out the complete code](https://github.com/JavaHammes/trusting_compilers/tree/main) and experiment with it yourself!

Detecting this kind of attack is non-trivial. Static analysis of binaries is difficult, especially when the malicious logic is behaviorally indistinguishable from legitimate compilation steps. Unless you're disassembling the binary or using a fully trusted toolchain, this kind of subversion can fly completely under the radar.

---

## Final Thoughts

Ken Thompson's attack may have been theoretical, but it wasn't just a thought experiment, it was a wake-up call. It exposed a fundamental vulnerability in the way we trust software: we believe what we see in the source code, and we assume the tools that turn it into binaries are honest. But as we've seen, a compiler can lie, and that lie can survive indefinitely.

Back in 1984, this was a shocking idea. Today, it's still uncomfortable, but now, at least, we have some defenses.

So... could this attack still happen today?
Technically, yes. But practically? It’s harder, not impossible, just harder, thanks to modern software practices and security models. Here’s why:

### Reproducible Builds

Projects like [reproducible-builds.org](https://reproducible-builds.org/) aim to make builds verifiable. The idea is simple: if two people compile the same source code in two separate environments and get identical binaries, it's very unlikely any hidden modifications occurred.

### Supply Chain Auditing

Modern development workflows increasingly rely on signed binaries, package hashes, and continuous integration pipelines. These create cryptographic traces of the software build process, making it harder to insert a hidden compiler-level backdoor without detection.

Examples include:

- [Sigstore](https://www.sigstore.dev/)
- [SLSA](https://slsa.dev/) (Supply-chain Levels for Software Artifacts) by Google

### Bootstrapping from Trusted Cores

Some projects, like [Stage0](https://github.com/oriansj/stage0), attempt to build entire systems from first principles, starting with hex and working their way up. These are extreme, but they show it is possible to verify a toolchain all the way down to bits and atoms (or close).

**But Trust Still Matters**

Even with all these defenses, trust in the people who write your compilers and infrastructure remains essential. No system is perfect. We can reduce the places where trust is required, but we can't remove it entirely.

Thompson's insight holds true in 2025: *you can't trust code just because it looks safe.* You have to trust the people, the process, and the provenance.

---

"You can't trust code that you did not totally create yourself." - Ken Thompson
