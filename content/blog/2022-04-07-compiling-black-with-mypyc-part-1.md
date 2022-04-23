---
title: Compiling Black with mypyc, Pt. 1 - Initial Steps
slug: compiling-black-with-mypyc-part-1
description: &desc I spent a COVID summer (and then some) integrating mypyc into Black
  to double performance. How was it?
summary: *desc
tags: [black, mypyc]
draft: true
showToc: true
date: 2022-04-07 12:00:00-04:00
---

[Release 22.1.0 of Black][release-22.1.0] was special, not only was it the first stable
version of Black, it was also the first release to ship with [mypyc] compiled wheels,
doubling performance[^1] 🎉.

The journey getting here took over two years, the creation of two new dev tools, and lots
and lots of headscratching. Let's go down memory lane, shall we?

______________________________________________________________________

This is a long story so it's broken up into three parts. Collectively this is the
*Compiling Black with mypyc* series.

- [Pt. 1 - Initial Steps](.) (you're reading this one)
- [Pt. 2 - Optimization](../compiling-black-with-mypyc-part-2/)
- [Pt. 3 - Deployment](../compiling-black-with-mypyc-part-3/)

> While I'd love it if you would read this end to end, I totally understand if this ends
> up in the "to read later" section of your bookmarks. I just hope you enjoy whatever you
> do read!

## Introduction to mypyc

Before I dig into the story of how I integrated mypyc into Black, I want to explain what
mypyc is and the most basic example using it. **mypyc is a transcompiler converting typed
Python code into fast C extensions.** Being built on top of the [mypy] type checker, it
leverages standard type annotations[^2].

It has been used to compile mypy (and itself since mypyc comes with mypy) since 2019,
giving it a 4x performance boost over interpreted Python. According to the mypyc project:

> Existing code with type annotations is often 1.5x to 5x faster when compiled. Code tuned
> for mypyc can be 5x to 10x faster.[^3]

It achieves these impressive speed ups by:

- Using the C-API directly, avoiding the CPython interpreter overhead.

- Leveraging *early binding*, resolving called functions and names at compile-time,
  skipping the expensive dictionary lookup at runtime.

- Using optimized, type-specific primitives for many built-in functions and methods.

- Using custom memory-efficient, unboxed representations for integers and booleans.

- Compiling regular classes to C extension classes. C extension classes use [vtables] for fast method calls
  and attribute access.

... and other optimizations I've left out for the sake of brevity.

______________________________________________________________________

### Time for an example

Interested? I sure hope so since it's time for an example! To get started, create a
throwaway directory somewhere, a virtual environment, and install mypy. As of writing I'm
using mypy 0.931 on Ubuntu 20.04.03 with CPython 3.8.5.

Next, let's take a look at the code[^5] we'll soon compile. It's a simple program that
bruteforces its way to find all of the factor pairs for a given product. I mostly chose
this example because it's very dear to my heart. It was the first "serious" program I wrote
while learning Python[^4]. I originally wrote the same logic in code.org's app lab many
years ago.

```python3
import time
from typing import List, Tuple

def compute(product: int) -> List[Tuple[int, int]]:
    answers = []
    for factor in range(1, product + 1):
        if not product % factor:
            factor2 = int(product / factor)
            answers.append((factor, factor2))
    return answers

t0 = time.perf_counter()
compute(27_000_000)
elapsed = time.perf_counter() - t0
print(f"compute() took {elapsed * 1000:.1f} milliseconds")
```

Once saved, run it while noting the time it takes:

```console
$ python coefficient_finder.py
compute() took 1880.7 milliseconds
```

Alright, it's a bit slow, let's see if mypyc can change that:

```console
$ mypyc coefficient_finder.py
running build_ext
creating build/temp.linux-x86_64-3.8
creating build/temp.linux-x86_64-3.8/build
gcc -pthread -Wno-unused-result -Wsign-compare -DNDEBUG -g -fwrapv -O3 -Wall -fPIC -I/home/ichard26/programming/webdev/blog/testing-grounds/venv/lib/python3.8/site-packages/mypyc/lib-rt -I/home/ichard26/programming/webdev/blog/testing-grounds/venv/include -I/opt/python3.8.5/include/python3.8 -c build/__native.c -o build/temp.linux-x86_64-3.8/build/__native.o -O3 -g1 -Werror -Wno-unused-function -Wno-unused-label -Wno-unreachable-code -Wno-unused-variable -Wno-unused-command-line-argument -Wno-unknown-warning-option -Wno-unused-but-set-variable
creating build/lib.linux-x86_64-3.8
gcc -pthread -shared build/temp.linux-x86_64-3.8/build/__native.o -o build/lib.linux-x86_64-3.8/coefficient_finder.cpython-38-x86_64-linux-gnu.so

$ python -c "import coefficient_finder"
compute() took 331.6 milliseconds
```

And would you look at that, mypyc gave us a near 6x improvement over CPython! What's neat
is that I didn't even need to type all of the variables thanks to mypy's type inference.

> And actually, since the `mypyc` script is a simple wrapper that generates a setuptools
> project and builds it in-place, you can peek into the `build` directory and look at the
> generated C code (should be `__native.c`).

Feel free to edit the program and play around with mypyc, seeing what little scripts it
can speed up. Pro-tip, to restore back to the interpreted version of your program, you can
delete the `.so` file on Linux & MacOS or the `.pyd` file on Windows. In this example
I got `coefficient_finder.cpython-38-x86_64-linux-gnu.so`.

______________________________________________________________________

### It's not a silver bullet

There's several downsides to using mypyc though. While the project aims for excellent
compatibility with standard Python behaviour, there's some major deviations worth noting:

- Types are enforced at runtime and violations **will** cause a TypeError (this makes
  using `typing.Any` quite dangerous).

- Compiled modules can't be run directly, you have to import 'em instead.

- Monkeypatching anything compiled is unlikely to work as compiled code skips many of the
  runtime `__dict__` lookups normal Python does.

- Assignments to class and instance namespaces will either error or do nothing, in
  particular you can't add previously undeclared attributes later on.

- Standard multiple inheritance isn't supported.

- Profilers (e.g. CProfile), debuggers (e.g. pdb), and tracing hooks (e.g. coverage) won't
  work as the C code doesn't trigger the relevant hooks.

There's more details available in the official
[mypyc documentation here][mypyc-deviations-one] and [also here][mypyc-deviations-two].

## black + mypyc: Initial steps

I wasn't the first person to integrate mypyc into Black; way back in September 2019,
[@msullivan] [opened a PR getting Black ready][original-pr] for compilation. Unfortunately,
as is typical in open source projects, no one took the half-completed work and pushed it to
production, i.e. PyPI, ... for almost two years.

> You might be wondering why performance even matters, well clearly it mattered a lot
> since [GH-366] was opened in June 2018! The TL;DR is that for environments where Black
> is **ran on save automatically, the more responsive it is the better** as less time
> is spent in a laggy editor window.
>
> Startup time is important too given imports are costly, but this issue was explicitly
> about formatting throughput (it turns out mypyc can reduce import time too!).

Given I first publicly announced my work finishing up the project on July 4th 2021, I
sadly don't remember why I decided to pick it up. All I remember was fighting mypyc
compile-time type errors and fighting an outdated gcc... well OK I do remember I was
looking to learn more about type systems and C development in general, but that's it, I
promise!

Anyway, even upgrading from gcc-9 to gcc-10 wasn't enough. Currently mypyc has a bug
([mypyc#885]) where its boxing / unboxing code triggers
`array subscript 1 is above array bounds` on
[`blib2to3.pgen2.parse.Parser.setup`][gcc-bug-code]. I managed to simplify the
reproduction case to the following:

```python3
from typing import Optional, Tuple

Context = Tuple[str, Tuple[int, int]]
RawNode = Tuple[int, Optional[str], Optional[Context]]

def setup() -> None:
    # This is what ticks off gcc.
    newnode: RawNode = (7, None, None)
```

... yeah I know this doesn't make mypyc seem particularly robust. While I would argue it's
not entirely stupid to use it in prod (because I did so), you would be right in pointing
out it's alpha quality and needs careful testing. And to be fair, poor past me didn't know
just how many issues I'd soon face. All I can hope is that this work will help mypyc
improve ❀

Trying to compile `black` and `blib2to3` (our custom fork of lib2to3[^6]) initially seemed
like a small task with just these type errors:

```console
$ python setup.py --use-mypyc bdist_wheel
Parsed and typechecked in 8.566s
Compiled to C in 0.000s
src/black/parsing.py:30:9: error: Cannot assign multiple modules to name "ast3" without explicit "types.ModuleType" annotation
src/black/parsing.py:30:9: error: Cannot assign multiple modules to name "ast27" without explicit "types.ModuleType" annotation
src/black/parsing.py:146:13: error: Incompatible types in assignment (expression has type "Tuple[Type[typed_ast.ast3.TypeIgnore], Type[typed_ast.ast27.TypeIgnore], Type[_ast.TypeIgnore]]", variable has type "Tuple[Type[typed_ast.ast3.TypeIgnore], Type[typed_ast.ast27.TypeIgnore]]")
```

The ones about `Cannot assign multiple modules to name ...` are valid if annoying, back
then the code in `black.parsing` was pretty dynamic. The other one was just that mypy
under "mypyc super-strict mode" won't infer a variable to be a variable-length tuple even
if it's extended right after:

```python3 {hl_lines=[12]}
def stringify_ast(
    node: Union[ast.AST, ast3.AST, ast27.AST], depth: int = 0
) -> Iterator[str]:
    """Simple visitor generating strings to compare ASTs by content."""

    node = fixup_ast_constants(node)

    yield f"{'  ' * depth}{node.__class__.__name__}("

    for field in sorted(node._fields):  # noqa: F402
        # TypeIgnore has only one field 'lineno' which breaks this comparison
        type_ignore_classes = (ast3.TypeIgnore, ast27.TypeIgnore)
        if sys.version_info >= (3, 8):
            type_ignore_classes += (ast.TypeIgnore,)
```

The fix was to add `Tuple[Type, ...]`.

```python3
type_ignore_classes: Tuple[Type, ...] = (ast3.TypeIgnore, ast27.TypeIgnore)
```

Now, the codebase was able to type check even under "mypyc super-strict mode," but
*obviously* that just meant I had another class of fires to deal with, \*compiler
crashes\*!

```console
$ python setup.py --use-mypyc bdist_wheel
Parsed and typechecked in 8.929s
Compiling cache, strings, black, debug, parsing, brackets, lines, files, literals, conv, trans, linegen, numerics, nodes, const, pgen, pygram, output, token, concurrency, parse, report, mode, driver, grammar, tokenize, pytree, rusty, comments
Traceback (most recent call last):
  File "mypyc/irbuild/builder.py", line 169, in accept
  File "mypy/nodes.py", line 950, in accept
  File "mypyc/irbuild/visitor.py", line 104, in visit_class_def
  File "mypyc/irbuild/classdef.py", line 137, in transform_class_def
  File "mypyc/irbuild/classdef.py", line 388, in generate_attr_defaults
  File "mypyc/ir/class_ir.py", line 174, in attr_type
  File "mypyc/ir/class_ir.py", line 171, in attr_details
src/black/trans.py:187: KeyError: "'CustomSplitMapMixin' has no attribute '_Key'"
```

This issue was just due to missing `ClassVar` declarations for attributes that were
functionally class-only attributes. The fix was pretty easy :)

```diff
+@trait
 class CustomSplitMapMixin:
     """
     This mixin class is used to map merged strings to a sequence of
@@ -191,8 +204,10 @@ class CustomSplitMapMixin:
     the resultant substrings go over the configured max line length.
     """

-    _Key = Tuple[StringID, str]
-    _CUSTOM_SPLIT_MAP: Dict[_Key, Tuple[CustomSplit, ...]] = defaultdict(tuple)
+    _Key: ClassVar = Tuple[StringID, str]
+    _CUSTOM_SPLIT_MAP: ClassVar[Dict[_Key, Tuple[CustomSplit, ...]]] = defaultdict(
+        tuple
+    )
```

Do you see that `@trait` addition? well ... mypyc does in fact support a limited form of
multiple inheritance called **traits**. They're basically mixins. As noted in the
[mypyc documentation][traits-docs], they shouldn't be instantiated or subclass non-traits.

And as usual, fixing this crash revealed \*even\* \*more\* \*issues\*, albeit mypyc
specific this time:

```console
$ python setup.py --use-mypyc bdist_wheel
Parsed and typechecked in 8.122s
Compiling pgen, parse, parsing, mode, pytree, token, debug, const, lines, numerics, rusty, conv, comments, files, trans, cache, black, grammar, output, brackets, tokenize, linegen, pygram, report, literals, nodes, strings, concurrency, driver
Compiled to C in 0.970s
src/black/trans.py:250: error: Non-trait bases must appear first in parent list
src/black/trans.py:934: error: Non-trait bases must appear first in parent list
src/black/trans.py:1433: error: Non-trait bases must appear first in parent list
src/black/brackets.py:52: error: Inheriting from most builtin types is unimplemented
src/black/files.py:52: warning: Treating generator comprehension as list
src/black/trans.py:746: warning: Unsupported default attribute value
src/black/trans.py:1831: warning: Unsupported default attribute value
src/blib2to3/pgen2/tokenize.py:430: error: Local variable "stashed" has inferred type None; add an annotation
src/black/nodes.py:442: error: Local variable "stop_after" has inferred type None; add an annotation
src/black/nodes.py:443: error: Local variable "last" has inferred type None; add an annotation
```

These were pretty straightforward to fix and sometimes even improved code clarity, forcing
me to add type annotations in complex code.

### More changes were necessary than expected

All of this was only to get the codebase to type check and not crash the compiler. Getting
the built binary to not crash at runtime needed more work. Some of these changes were
improvements, like this one which involved an incorrect type annotation:

```diff
diff --git a/src/black/__init__.py b/src/black/__init__.py
index 8e2123d..6998c1e 100644
--- a/src/black/__init__.py
+++ b/src/black/__init__.py
@@ -359,7 +359,7 @@ def main(
     experimental_string_processing: bool,
     quiet: bool,
     verbose: bool,
-    required_version: str,
+    required_version: Optional[str],
     include: Pattern,
     exclude: Optional[Pattern],
     extend_exclude: Optional[Pattern],
```

> This goes to show that while the strict runtime type checks can be very annoying, they
> do have value beyond memory safety.

#### Sad and frustrating changes

... but others were just sad or annoying (mixing dataclasses with anything remotely fancy
breaks mypyc).

```diff
@@ -40,7 +38,8 @@ class CannotSplit(CannotTransform):
     """A readable split that fits the allotted line length is impossible."""


-@dataclass
+# This isn't a dataclass because @dataclass + Generic breaks mypyc.
+# See also https://github.com/mypyc/mypyc/issues/827.
 class LineGenerator(Visitor[Line]):
     """Generates reformatted Line objects.  Empty lines are not emitted.

@@ -48,9 +47,11 @@ class LineGenerator(Visitor[Line]):
     in ways that will no longer stringify to valid Python code on the tree.
     """

-    mode: Mode
-    remove_u_prefix: bool = False
-    current_line: Line = field(init=False)
+    def __init__(self, mode: Mode, remove_u_prefix: bool = False) -> None:
+        self.mode = mode
+        self.remove_u_prefix = remove_u_prefix
+        self.current_line: Line
+        self.__post_init__()

     def line(self, indent: int = 0) -> Iterator[Line]:
         """Generate a line.
```

```diff
@@ -62,7 +71,6 @@ def TErr(err_msg: str) -> Err[CannotTransform]:
     return Err(cant_transform)


-@dataclass  # type: ignore
 class StringTransformer(ABC):
     """
     An implementation of the Transformer protocol that relies on its
@@ -90,9 +98,13 @@ class StringTransformer(ABC):
         as much as possible.
     """

-    line_length: int
-    normalize_strings: bool
-    __name__ = "StringTransformer"
+    __name__: Final = "StringTransformer"
+
+    # Ideally this would be a dataclass, but unfortunately mypyc breaks when used with
+    # `abc.ABC`.
+    def __init__(self, line_length: int, normalize_strings: bool) -> None:
+        self.line_length = line_length
+        self.normalize_strings = normalize_strings
```

#### Hackiness += 10000

One change stood out as the most hacky[^7] code I've probably ever written so far:

```diff
@@ -335,7 +336,9 @@ def transform_line(
         transformers = [left_hand_split]
     else:

-        def rhs(line: Line, features: Collection[Feature]) -> Iterator[Line]:
+        def _rhs(
+            self: object, line: Line, features: Collection[Feature]
+        ) -> Iterator[Line]:
             """Wraps calls to `right_hand_split`.

             The calls increasingly `omit` right-hand trailers (bracket pairs with
@@ -362,6 +365,11 @@ def transform_line(
                 line, line_length=mode.line_length, features=features
             )

+        # HACK: functions (like rhs) compiled by mypyc don't retain their __name__
+        # attribute which is needed in `run_transformer` further down. Unfortunately
+        # a nested class breaks mypyc too. So a class must be created via type ...
+        rhs = type("rhs", (), {"__call__": _rhs})()
+
         if mode.experimental_string_processing:
             if line.inside_brackets:
                 transformers = [
```

I also had to change the code accessing `__name__` to do so via `__class__` so this hack
could work. Minor correction though, top-level compiled functions still have `__name__`,
it's just nested functions that don't have 'em[^8]. I improved this comment before this
landed.

#### The least bad workaround

I remember getting an impromptu "how to read mypyc-generated C code" lesson from
[@JelleZijlstra] [^9]. I was trying to debug this runtime crash in this `black.trans`
class:

```python3 {hl_lines=[6]}
class StringParser:
    """
    [snipped]
    """

    DEFAULT_TOKEN = -1

    # String Parser States
    START = 1
    DOT = 2
    NAME = 3
    PERCENT = 4
    SINGLE_FMT_ARG = 5
    LPAR = 6
    RPAR = 7
    DONE = 8

    # Lookup Table for Next State
    _goto: Dict[Tuple[ParserState, NodeType], ParserState] = {
        [snipped]
    }

    [snipped]
```

```python3
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "test2.py", line 30, in <module>
    (START, DEFAULT_TOKEN): DONE,
KeyError: 'DEFAULT_TOKEN
```

It turns out the generated code wanted to access the `DEFAULT_TOKEN` attribute from the
globals dict, predictably blowing up.

```c {hl_lines=[1]}
    cpy_r_r146 = CPyStatic_globals;
    cpy_r_r147 = CPyStatics[17]; /* 'DEFAULT_TOKEN' */
    cpy_r_r148 = CPyDict_GetItem(cpy_r_r146, cpy_r_r147);
```

If this scares you, don't fret, it's not too hard to read. First, the globals dict is
assigned to `cpy_r_r146`, then the name (or key) we're looking up is read and stored in
`cpy_r_r147`. Finally, `cpy_r_r146` and `cpy_r_r147` are passed to the `CPyDict_GetItem`
function.

Effectively, the buggy C code was trying to do this (unnecessary lines were replaced with
`...`):

```python
class StringParser:
    DEFAULT_TOKEN = -1
    START = 1
    DONE = 8
    ...

    __var_146 = globals()
    __var_147 = "DEFAULT_TOKEN"
    __var_148 = var_146[var_147]

    _goto = {
        (START, __var_148): DONE
        ...
    }
```

The root issue ([mypyc#862]) is that mypyc doesn't understand class level scoping ... or
that negative integers are constants (using a positive integer fixed the crash). Working
around this issue allowed me to embed the date I fixed this and many other issues. I think
it's a pretty cute historical codebase quirk ❀

````diff
@@ -1811,20 +1826,20 @@ class StringParser:
         ```
     """

-    DEFAULT_TOKEN = -1
+    DEFAULT_TOKEN: Final = 20210605
````

______________________________________________________________________

### *Phew*, it's faster

It was at this time I finally posted the
[first set of benchmark numbers][initial-results], they weren't scientific or reliable in
any way, but it did prove mypyc was still a worthwhile way to improve performance.
Formatting the Black codebase (so meta, I know) with compiled Black yielded a performance
gain of 1.82x, impressive for the short amount of time I had put in!

I had to verify this experimental build wasn't completely unstable though. I couldn't just
run the test suite with compiled Black installed, some tests use mocks which trigger the
strict type checks mypyc does at runtime. Sadly, I had to mark these tests as straight up
incompatible so they could be skipped.

### It's still alpha software

As of writing mypyc is still alpha software with rough edges everywhere, I already showed
quite a few of them above, but there were more 😥 You can read the rest in this
[integration report][integration-report] I filed in the mypyc issue tracker.

And I know, y'all are wondering why Black didn't use Cython, well I'm not sure as I wasn't
around in 2019 when [@ambv first suggested its use][ambv-mypyc-suggestion]. Eitherway I
didn't look into Cython because at the time as it seemed like I only had the final scraps
left and didn't have too much work to do ... which was both right and wrong depending on
how you look at it.

Could Black be even faster if we had used Cython? Probably, but that's for another day as
my personal roadmap for psf/black is already packed :p

**Anyway, it's time for [Pt. 2 - Optimization](../compiling-black-with-mypyc-part-2/).**

[^1]: When I first landed the relevant PR it was an overall 2x improvement, but
    once Jelle added a stability hotfix the *effective* speedup for files that were changed
    is 50%. If you're formatting a bunch of already well formatted files, the speedup is
    still 2x

[^2]: There's too many typing related PEPs to copy and paste, but there's an up to date list
    here: <https://docs.python.org/3/library/typing.html#relevant-peps>

[^3]: <https://mypyc.readthedocs.io/en/stable/introduction.html#introduction>

[^5]: I am well aware I don't need to import `List` or `Tuple` from `typing` anymore. I'm
    just keeping my code examples compatible with older versions of Python.

[^4]: Well not quite exactly this, it was a crappier solution with a time complexity of O(n²)
    instead of O(n), oh and of course my code style was less than ideal and I didn't use
    type annotations, but let's not go there :)

[^6]: At the time @ambv wrote Black, lib2to3 was the best AST + CST mix out there. AST
    parsers like the built-in one are great for understanding syntax and structure, but
    are useless when editing how much whitespace a NAME token gets for example. CSTs on
    the other hand are great for visual changes Black does, but don't provide enough
    context to allow Black to make the right decisions, hence lib2to3 and the fork.

[^7]: What's funny is that the PR which added this `__name__` check had a review noting the
    hackiness of reading the attribute in the first place. It was dismissed as it would
    have taken too much effort to properly implement the required logic, turns out we got
    this fun code in return!

[^8]: It's because internally mypyc implements nested functions as callable classes

[^9]: Reading the results searching `from:ichard26#4772 in:black-formatter mypyc` in the
    Python Discord server nets some interesting discussion and history :p

[@jellezijlstra]: https://github.com/JelleZijlstra
[@msullivan]: https://github.com/msullivan
[ambv-mypyc-suggestion]: https://github.com/psf/black/issues/366#issuecomment-490153026
[gcc-bug-code]: https://github.com/psf/black/blob/8ea641eed5b9540287a8e9a9afa1458b72b9b630/src/blib2to3/pgen2/parse.py#L117-L139
[gh-366]: https://github.com/psf/black/issues/366
[initial-results]: https://github.com/psf/black/issues/366#issuecomment-855306152
[integration-report]: https://github.com/mypyc/mypyc/issues/886
[mypy]: https://mypy.readthedocs.io/en/stable/introduction.html
[mypyc]: https://mypyc.readthedocs.io/en/stable/introduction.html
[mypyc#862]: https://github.com/mypyc/mypyc/issues/862
[mypyc#885]: https://github.com/mypyc/mypyc/issues/885
[mypyc-deviations-one]: https://mypyc.readthedocs.io/en/stable/differences_from_python.html
[mypyc-deviations-two]: https://mypyc.readthedocs.io/en/stable/native_classes.html
[original-pr]: https://github.com/psf/black/pull/1009
[release-22.1.0]: https://github.com/psf/black/releases/tag/22.1.0
[traits-docs]: https://mypyc.readthedocs.io/en/latest/using_type_annotations.html#trait-types
[vtables]: https://en.wikipedia.org/wiki/Virtual_method_table
