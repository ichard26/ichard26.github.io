---
title: Debugging an import-time AttributeError with Click CLIs w/ mypyc
description: &desc >-
  I was asked to investigate "AttributeError: 'builtin_function_or_method' object
  has no
  attribute '__click_params__ caused by compiling a Click CLI. Here's what's wrong
  and how to fix it.
summary: *desc
tags: [mypyc]
draft: true
showToc: true
thumbnail: an import-time AttributeError with a Click CLI w/ mypyc
---

**FYI**: If you're here looking for a way to work around this issue, feel free to skip the
investigation section, it won't help you fix your crashes.

## Background

[@ofek], who you might know as the maintainer of [Hatch], pinged me in the
[PyPA Discord server][pypa-discord] asking whether mypyc supports [Click]-based CLIs. I
said yes [having successfully compiled Black](../../05/compiling-black-with-mypyc-part-1)
which uses Click.

They then sent me a [link to this file][bug-comment] which has a comment that notes an odd
interaction between Click's decorators and mypyc, sometimes resulting in a crash.

```python3 { hl_lines=[6, 7] }
import click

from hatch_showcase._version import version
from hatch_showcase.fib import fibonacci

# NOTE: The group/command decorators must come last to avoid the following issue at runtime:
# https://github.com/pallets/click/issues/1199


@click.version_option(version=version, prog_name='hatch-showcase')
@click.group()
def hatch_showcase():
    pass


@click.argument('n', type=int)
@hatch_showcase.command()
def fib(n: int):
    click.echo(fibonacci(n))
```

This code compiles fine, actually. Replacing the imports with *good-enough* stubs and
running `mypyc test.py && python -c "import test"`works.

```python3
import click

version = "1.0.0"

def fibonacci(n: int) -> int;
    return n

# NOTE: The group/command decorators must come last to avoid the following issue at runtime:
# https://github.com/pallets/click/issues/1199


@click.version_option(version=version, prog_name='hatch-showcase')
@click.group()
def hatch_showcase():
    pass


@click.argument('n', type=int)
@hatch_showcase.command()
def fib(n: int):
    click.echo(fibonacci(n))
```

```console
$ mypyc test.py && python -c "import test"
running build_ext
building 'test' extension
creating build/temp.linux-x86_64-cpython-38
creating build/temp.linux-x86_64-cpython-38/build
gcc -pthread -Wno-unused-result -Wsign-compare -DNDEBUG -g -fwrapv -O3 -Wall -fPIC -I/home/ichard26/programming/webdev/blog/testing-grounds/venv/lib/python3.8/site-packages/mypyc/lib-rt -I/home/ichard26/programming/webdev/blog/testing-grounds/venv/include -I/opt/python3.8.5/include/python3.8 -c build/__native.c -o build/temp.linux-x86_64-cpython-38/build/__native.o -O3 -g1 -Werror -Wno-unused-function -Wno-unused-label -Wno-unreachable-code -Wno-unused-variable -Wno-unused-command-line-argument -Wno-unknown-warning-option -Wno-unused-but-set-variable
creating build/lib.linux-x86_64-cpython-38
gcc -pthread -shared build/temp.linux-x86_64-cpython-38/build/__native.o -o build/lib.linux-x86_64-cpython-38/test.cpython-38-x86_64-linux-gnu.so
copying build/lib.linux-x86_64-cpython-38/test.cpython-38-x86_64-linux-gnu.so ->
$ < there is no crash! >
```

Even using the CLI works just fine:

```console
$ python -c "import test; test.hatch_showcase()" fib 27
27
```

(remember that I replaced the `fibonacci` function with a stub function that simply
returns the number unchanged.)

However! If we switch the order of the decorators on `hatch_showcase`, things start
crashing as alluded to in that comment.

```python3 { hl_lines=[1, 2] }
@click.group()
@click.version_option(version=version, prog_name='hatch-showcase')
def hatch_showcase():
    pass
```

```console
$ mypyc test.py && python -c "import test"
< uninteresting gcc compilation output removed >
Traceback (most recent call last):
  File "<string>", line 1, in <module>
  File "test.py", line 13, in <module>
    @click.group()
  File "/home/ichard26/programming/webdev/blog/testing-grounds/venv/lib/python3.8/site-packages/click/decorators.py", line 308, in decorator
    _param_memo(f, OptionClass(param_decls, **option_attrs))
  File "/home/ichard26/programming/webdev/blog/testing-grounds/venv/lib/python3.8/site-packages/click/decorators.py", line 269, in _param_memo
    f.__click_params__ = []  # type: ignore
AttributeError: 'builtin_function_or_method' object has no attribute '__click_params__'
```

If you're a curious individual, you will notice this error is the same as the one
described in the issue linked in the comment from earlier!

This peaked my interest as I didn't expect this code to break. Having looked into this
issue more, I'm now surprised there aren't more reports of this. What follows below is my
investigation.

## Investigation

I first thought "this should be impossible" since mypyc wouldn't

## Fixing it

There are multiple ways of fixing this issue.

For those of you writing CLIs with Click, you have several workarounds:

- Move the `@click.group` (TODO: check why `@click.command` works fine at the top)

[@ofek]: https://github.com/ofek
[bug-comment]: https://github.com/ofek/hatch-showcase/blob/27180309b52dab1f3676f8c50f23bdbedc747f61/src/hatch_showcase/cli/__init__.py#L9-L10
[click]: https://click.palletsprojects.com/en/8.1.x/
[hatch]: https://github.com/pypa/hatch
[pypa-discord]: https://discord.gg/pypa
