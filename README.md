# shint

_Warning: Early development software. Not yet ready for public use._

`shint` is a Python 3 library for safely generating POSIX/bash shell code. Its goal is to let you
embed hard-to-translate shell idioms in Python, exploiting the shell's best features without the
[security issues][popen-security] associated with untrusted strings.

[popen-security]: https://docs.python.org/3/library/subprocess.html#security-considerations

## Motivation

Python 3's `subprocess.Popen` and its related methods / wrappers follow a convention where the
initial argument `args` is interpreted in a peculiar way, described here in pseudo-code:

```python
if shell == False:
    if type(args) == str:
        # Quote `args` and send to /bin/sh
        # /bin/sh interprets this as an executable path with no arguments
    if type(args) == list:
        # Quote each `args` element and join the result with ' '
        # /bin/sh interprets this as an executable path with arguments
if shell == True:
    if type(args) == str:
        # Do *not* quote `args`; send to /bin/sh as is
        # /bin/sh interprets the contents of `args` as an arbitrary script
    if type(args) == list:
        # Silently discard all but the first element, without warning
        # Send the first element (unquoted) to /bin/sh
```

The module docs encourage the default `shell=False` whenever possible, and warn that properly
escaping quotes is your responsibility should you choose to set `shell=True`. Of course, the
`shell=True` version is what you were hoping to find.

You now have a choice:

- **A**: Tolerate a few extra commas and quote marks (`["head", "-n", str(100)]`) and represent
  every subprocess you'd need normal call in shell with a `subprocess.Popen` instance — the
  "Pythonic" way
- **B**: Have every shell convenience you like, but be responsible for remembering to use
  `shlex.quote()` — failures penalized with silent errors, landmine bugs, and security
  vulnerabilities

### Option A: The "Pythonic" way

If your main need for shell is building subprocess pipelines, the recommended path is harder than it
appears. When piping data with `Popen`, file descriptors must be linked manually, requiring you to
choose a label every command no matter how brief. A one-liner like `head -n 100 "$filepath" | grep
foo | wc -l` turns into:

```python
cmd_head = ["head", "-n", str(100), shlex.quote(str(filepath))]
cmd_grep = ["grep", "foo"]
cmd_wc = ["wc", "-l"]

proc_head = subprocess.Popen(cmd_head, stdout=subprocess.PIPE)
proc_grep = subprocess.Popen(
    cmd_grep, stdin=proc_head.stdout, stdout=subprocess.PIPE
)
proc_wc = subprocess.Popen(
    cmd_wc, stdin=proc_grep.stdout, stdout=subprocess.PIPE
)
proc_head.stdout.close()
proc_grep.stdout.close()
proc_wc.communicate()
```

This style of translation has a number of issues:

- It's **hard to maintain**. Each of the strings "head", "grep", and "wc" appear six times, and
  changes must be made in multiple places to add or remove commands from the pipeline. Subprocess
  and file-descriptor boilerplate that would be common to any pipeline task is mixed with the actual
  content of commands. This is exacerbated when commands are parameterized or, worse, exchange data
  with the Python process at runtime.)
- It's **hard to debug**. If a mistake is made (`proc_head` is used where `proc_grep` should be) it
  generally will not raise an error until the pipeline has already started executing, if at all.
- It's **it's never quite shell**. There are many subtle differences between `/bin/sh` and
  `subprocess.Popen` regarding pipe buffering, stream signalling, terminal detection, handling of
  stderr, handling of failed processes, named pipes, and killing child processes during interrupts.
  Translating common bash techniques like process substitution (`diff <(ls foo) <(ls bar)`) is far
  from trivial. Dealing with these issues can quickly outweigh the benefit of using Python over
  shell at all.

### Option B: Quoting paranoia

The alternative is not much better. Shell commands almost always need to include string arguments
from the Python parent, and those arguments must be passed through `shlex.quote()`. If a string is
missed, commands will often continue to work without error — for now. Then, once the code is in
production, a file path contains a space and the script fails.

The problem is that strings are everywhere, and it's not realistic to treat all of them as toxic.
You need something like the `Popen(shell=False)` interface that `subprocess` recommends, where quoting
happens by default, but still be able to use the shell features you need.

### Solution: Better quotes through typing

Part of the reason the default `Popen(shell=False)` works so well in simple cases (e.g., safely
invoking a single command with arguments) is because it represents a shell command as a crude
[abstract syntax tree][AST] — namely, a tree of depth 2 that only models commands and their
space-separated arguments.

[AST]: https://en.wikipedia.org/wiki/Abstract_syntax_tree

**shint** takes this idea a step further, letting you build larger, more complex shell commands
in Python to serve as safe inputs for `Popen(shell=True)`:

- Strings are always assumed to be unquoted, and will never be interpreted as shell syntax
- As with `shell=False`, lists of strings are rendered as a space-separated sequence — a shell
  command with arguments
- When you need shell syntax, you instantiate a wrapper class that preserves its arguments unquoted
- Combining these shell-syntax wrappers, you build your shell script as an AST in Python
- Unlike a string-templated shell script with embedded `shlex.quote()`, the AST can be introspected
  and modified after its parameters have been set — meaning higher-level structures can be
  managed in Python
    