# shint

_Warning: Early development software. Not yet ready for public use._

`shint` is a Python 3 library for safely generating shell code in Python. Its
goal is to let you embed hard-to-translate shell idioms in Python, exploiting
the shell's best features without the [security issues][popen-security]
associated with untrusted strings.

[popen-security]:
https://docs.python.org/3/library/subprocess.html#security-considerations

## Motivation

The key issue addressed here is that subprocess.Popen and its related methods /
wrappers follow a convention where the initial argument `args` is interpreted in
a confusing and bug-prone way (described in pseudo-code):

```python
    if shell == False:
        if type(args) == str:
            # Quote `args` and send to `/bin/sh`
            # `/bin/sh` interprets this as an executable path with no arguments
        if type(args) == list:
            # Quote each `args` element and join the result with ' '
            # `/bin/sh` interprets this as an executable path with arguments
    if shell == True:
        if type(args) == str:
            # Do *not* quote `args`; send to `/bin/sh` as is
            # `/bin/sh` interprets the contents of `args` as an arbitrary script
        if type(args) == list:
            # *Silently discard* all but the first element
            # Send first element unquoted to `/bin/sh`
```

The module docs encourage you to use the default `shell=False` whenever
possible, and warn that properly escaping quotes is your responsibility if you
choose to set `shell=True`. In practice, you must either:

    A) replace every needed shell feature with equivalent Python idioms, or
    B) carefully manage when strings are and are not expected to be quoted.

Option A is much more difficult than it appears. When forming large process
pipelines, file descriptors must be linked manually, requiring you to uniquely
label every command no matter how brief. A trivial one-liner like `head -n 100
$filepath | grep foo | wc -l` turns into:

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

This code is difficult to maintain. Each of the strings ("head", "grep", "wc")
appears six times, and changes must be made in multiple places to add or remove
commands from the pipeline. Subprocess and file-descriptor boilerplate that
would be common to any pipeline task is mixed with the actual content of
commands. (This is exacerbated when commands are parameterized or, worse, must
exchange data with the Python process at runtime.) If a mistake is made
(`proc_head` is used where `proc_grep` should be) it generally will not raise an
error until the pipeline has already started executing, if at all.

There are also many subtle differences between `/bin/sh` and `subprocess.Popen`
regarding pipe buffering, stream signalling, terminal detection, handling of
stderr, handling of failed processes, named and killing child processes during
interrupts. Dealing with these issues can quickly outweigh the benefits of using
Python over shell at all.

Option B, unfortunately, is not much better.

[Unfinished — work in progress]