---
title: 'Doubling performance: adding mypyc to Black'
description: &desc I spent a COVID summer (and then some) integrating mypyc to Black,
  how was it?
summary: *desc
tags: [black, mypyc]
draft: true
showToc: true
---

- \[ \] TODO: bold some text!
- \[ \] TODO: maybe add a homemade floating TOC ... or use
  <https://github.com/adityatelange/hugo-PaperMod/pull/675>
- \[ \] TODO: get Jelle, Łukasz, kosayoda (pydis), dawn (pydis) and friends to review this

> Hey, I realize this is a *massive* post, while I'd love it if you would read it end to
> end, I totally understand if this ends up in the "to read later" section of your
> bookmarks. I just hope you enjoy whatever you do read!

[Release 22.1.0 of Black][release-22.1.0] wasn't special just because it was the first
stable version of Black, it was also the first release to ship [mypyc] compiled wheels
doubling performance[^1] 🎉.

The journey getting here took over two years, the creation of two new dev tools, and lots
and lots of headscratching. Let's go down memory line shall we?

## Introduction to mypyc

Before I dig into the story of how I integrated mypyc into Black I want to explain what
mypyc is and the most basic example using it. mypyc is a transcompiler converting Python
code into fast C extensions. Being built on top of the [mypy] type checker it leverages
standard type annotations[^2].

It has been used to compile mypy (and itself!) since 2019, giving it a 4x performance
boost over interpreted Python. According to the mypyc project:

> Existing code with type annotations is often 1.5x to 5x faster when compiled. Code tuned
> for mypyc can be 5x to 10x faster.[^3]

It achieves these impressive speed ups by:

- Using the C-API directly, avoiding the CPython interpreter overhead.

- Leveraging *early binding* resolving called function and names at compile-time, skipping
  the expensive dictionary lookup at run-time.

- Using optimized, type-specific primitives for many built-in functions and methods.

- Using custom memory-efficient, unboxed representions for integers and booleans.

- Classes are compiled to C extension classes. They use [vtables] for fast method calls
  and attribute access.

... and other optimizations I've left out for the sake of brevity.

______________________________________________________________________

### Time for an example

Interested? I sure hope so since I think it's time for an example! To get started, create
a throwaway directory somewhere, a virtual environment, and install mypy (mypyc actually
comes with the mypy package). As of writing I'm using mypy 0.931 on Ubuntu 20.04.03 with
CPython 3.8.5.

Next, let's take a look at the code we'll soon compile. It's a simple program that
bruteforces its way to find all of the factor pairs for a given product. The main reason
why I chose this example is because it's very dear to my heart, the first "serious"
program I wrote while learning Python was this[^4]. It was a rewrite of the same logic I
originally wrote in code.org's app lab many years ago.

Save the following code[^5] as a file.

```python3
import time
from typing import List, Tuple

def compute(product: int) -> List[Tuple[int, int]]
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

Then, run it while noting the time it takes:

```console
$ python coefficient_finder.py
compute() took 1880.7 milliseconds
```

Cool, it's a bit slow, let's see if mypyc can change that

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
> based project, you can peek into the `build` directory and look at the generated C code
> (should be `__native.c`).

Feel free to edit the program and play around with mypyc seeing what little scripts it can
speed up. Pro-tip, to restore back to the interpreted version of your program, you can
delete the `.gnu.so` file on Linux & MacOS or the `.pyd` file on Windows. In this example
I got `coefficient_finder.cpython-38-x86_64-linux-gnu.so`.

______________________________________________________________________

### It's not a silver bullet

There's several downsides to using mypyc though. While the project aims for excellent
compatibility with standard Python behaviour, there's some major deviations worth noting:

- Types are enforced at runtime and violations will cause a TypeError (this makes using
  `typing.Any` quite dangerous).

- Compiled modules can't be run directly, you have to import 'em instead.

- Monkeypatching anything compiled is likely to not work as compiled code skips many of
  the runtime `__dict__` lookups normal Python does.

- Assignments to class and instance namespaces will either error or do nothing, in
  particular you can't add previously undeclared attributes later on.

- Standard multiple inheritance isn't supported.

- Profilers (e.g. CProfile), debuggers (e.g. pdb), and tracing hooks (e.g. coverage) won't
  work as the C code doesn't trigger the relevant hooks.

There's more details available in the official
[mypyc documentation here][mypyc-deviations-one] and [also here][mypyc-deviations-two].

## black + mypyc: Initial steps

I wasn't the first person to integrate mypyc into Black, way back in September 2019
[@msullivan] [opened a PR getting Black ready][original-pr] for compilation w/ mypyc.
Unfortunately as typical in open source projects, no one took the half-completed work and
pushed it to production (i.e. PyPI) ... for almost two years.

> You might be wondering why performance even matters, well clearly it mattered a lot
> since [GH-366] was opened in June 2018! The TL;DR is that for environments where Black
> is ran on the source code on save automatically, reducing the runtime improves
> responsiveness and developer experience (less time spent in a laggy editor window).
>
> Start up time is important too given imports are costly, but this issue was explicitly
> about formatting throughout (also it turns out mypyc can reduce import time too!).

Given I first publicly announced my work finishing up the project on July 4th 2021, I
sadly don't remember why I decided to pick it up. All I remember was fighting mypyc
compile time type errors and fighting an outdated gcc... well OK I do remember I was
looking to learn more about type systems and C development in general, but that's it,
promise!

Anyway, even upgrading from gcc-9 to gcc-10 wasn't enough. Currnently mypyc has a bug
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

... yeah I know this doesn't make mypyc seem like a particularly robust piece of software
and while I would argue it's not entirely stupid to use it in prod, you would be right in
pointing out it's alpha quality and needs careful testing. And actually, poor past me
didn't know the many issues I'd face soon. All I can hope is that this work will help
mypyc improve ❀

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
if there's an extension right after:

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

The fix was to add `Tuple[Type, ...]`:

```python3
type_ignore_classes: Tuple[Type, ...] = (ast3.TypeIgnore, ast27.TypeIgnore)
```

Now, the codebase was able to type check even under "mypyc super-strict mode", but
*obviously* that just meant I another class of fires to deal with, \*compiler crashes\*!

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
[mypyc documentation][traits-docs] they shouldn't be instantiated or subclass non-traits.

And as usual, fixing this compile-time crash revealed \*even\* \*more\* \*issues\*, albeit
mypyc specific this time:

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

These were pretty straightforward to fix and sometimes even improved code clarity forcing
me to add type annotations in complex code.

### More changes were necessary than expected

All of this was only to get the codebase to type check and not crash the mypyc compiler.
Getting the built binary to not crash at runtime needed more work. Once again, some of the
changes were improvements like this one which involved an incorrect type annotation:

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

This goes to show that while the strict runtime type checks can be very annoying they do
have value beyond memory safety.

#### Sad and frustrating changes

... but others were just sad or annoying (mixing dataclasses with anything remotely fancy
breaks mypyc):

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

I also had to change the check code to read `__name__` on the `__class__` attribute so
this hack could work. Minor correction though, top-level compiled functions still have
`__name__`, it's just nested functions that don't have 'em[^8]. I improved this comment
before this landed.

#### The least bad workaround

I remember getting an impromptu "how to read mypyc-generated C code" lesson from
[@JelleZijlstra] [^9]. I was trying to debug this runtime crash in this `black.trans`
class

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
globals dict which predictably fails.

```c {hl_lines=[1]}
    cpy_r_r146 = CPyStatic_globals;
    cpy_r_r147 = CPyStatics[17]; /* 'DEFAULT_TOKEN' */
    cpy_r_r148 = CPyDict_GetItem(cpy_r_r146, cpy_r_r147);
```

If this scares you, don't fret, it's not that hard to read. First, the globals dict is
assigned to `cpy_r_r146`, then the name (or key) we're looking up is read and stored in
`cpy_r_r147`. Finally `cpy_r_r146` and `cpy_r_r147` are passed to the `CPyDict_GetItem`
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
strict type checks mypyc does at runtime. I had to mark these tests as straight up
incompatible so they could be skipped sadly.

### It's still alpha software

As of writing mypyc is still alpha software with rough edges everywhere, I already showed
quite a few of them above, but there were more 😥 You can read the rest in this
[integration report][integration-report] I filed in the mypyc issue tracker.

And I know, y'all are wondering why Black didn't use Cython, well I'm not sure given I
wasn't around in 2019 when [@ambv first suggested its use][ambv-mypyc-suggestion]. Anyway
I didn't look into Cython because at the time as it seemed like I only had the final
scraps left and didn't have too much work to do ... which was both right and wrong
depending on how you look at it.

Could Black be even faster if we had used Cython? Probably, but that's for another day as
my personal roadmap for psf/black is already packed :p

## Optimizing for mypyc

With core Black being compilable and not crashing in all kinds of fun ways, it was time to
try optimizing Black for mypyc. While mypyc is designed to handle all sorts of statically
typed code, changing up the code even a little bit can allow mypyc to perform additional
optimizations ultimately helping performance a bunch.

And while I did go into this hoping I could spot some potential architectural or data
structure optimizations, I wasn't able to so all of the optimizations will be mypyc
specific.

### Getting started

First off, profiling! It's hard to optimize code well if you don't know where time is
being spent.

To start off, I profiled Black over some of its own source code with cProfile to get a
birds eye view:

```console
$ python -m cProfile -o profile.pstats -m black src/black/__init__.py --check
All done! ✨ 🍰 ✨
1 file would be left unchanged.

$ gprof2dot profile.pstats | dot -Tsvg -o profile.svg
```

With the help of [gprof2dot] (well, actually the [yelp-gprof2dot] fork), I converted the
profiling data into a nice SVG graph which I could then open in my web browser.

> Tip: please open this (massive) SVG in a new tab because viewing it here will probably
> be painful :)

![gprof2dot graph showing where time was spent in a call tree](/media/cProfileGraph-black-1.svg)

I also tried using [Scalene] as I heard it's a very powerful profiler, but it didn't work
properly sadly. Though I still had [py-spy], [line_profiler], and good ol' cProfile.
py-spy in particular was invaluable since it can profile (well err sample) C extensions
which cProfile cannot. line_profiler was used exclusively for micro opimitizations haha.

Anyway, I repeated this process quite a few times making sure to try different files to
get a general feel where time is going regardless of the input. Here's the main takeaways:

- Initial parsing with blib2to3 takes up 30-50% of formatting runtime!

- The AST equivalent safety check is cheap at 5-10% (obviously only when changes were
  made, otherwise it's zero 🙂)

- The actual formatting logic's runtime is mostly spent in the CST visitor usually taking
  up 3/4. The rest went to the transformers which handle line breaks, string literals, and
  some special cases.

I was quite surprised how much time blib2to3 related functions were eating up. Just look
how concentrated the hotspots really are!

![gprof2dot graph showing where time was spent parsing](/media/cProfileGraph-black-blib2to3-1.svg)

These hotspots make this part of the codebase way easier to optimize hence why I optimized
blib2to3 first.

It's been so long since I first looked into this, so I don't remember what optimizations I
tried initially, but I do remember them having no effect :(

I tried other things and they actually helped 🎉 ... probably since I took into account how
mypyc works. Ultimately, many different optimizations were done over three rounds.

#### Tightening up existing type annotations

The stricter your type annotations are in your codebase, the more invariants mypyc will be
able to infer. It'll then use this information and write code specific to the context's
type that is faster. This code won't work if it gets an object of a different type, but
that's why we use and enforce type annotations, so we (and mypyc) can safely assume it's
going to be the right type.

This involves reading through the code and control flow, checking whether certain states
are impossible. Blib2to3 is mostly a legacy codebase, so we when added type annotations to
it, it was done to unblock other work. The goal was to make blib2to3 type check, and not
write the best typed code ever. So, naturally there were a few permissive (parameter) type
annotations I could make stricter:

```diff
diff --git a/src/blib2to3/pgen2/parse.py b/src/blib2to3/pgen2/parse.py
index 47c8f02..6b03188 100644
--- a/src/blib2to3/pgen2/parse.py
+++ b/src/blib2to3/pgen2/parse.py
@@ -138,7 +138,7 @@ class Parser(object):
         self.rootnode: Optional[NL] = None
         self.used_names: Set[str] = set()

-    def addtoken(self, type: int, value: Optional[Text], context: Context) -> bool:
+    def addtoken(self, type: int, value: Text, context: Context) -> bool:
         """Add a token; return True iff this is the end of the program."""
         # Map from token to label
         ilabel = self.classify(type, value, context)
@@ -185,11 +185,10 @@ class Parser(object):
                     # No success finding a transition
                     raise ParseError("bad input", type, value, context)

-    def classify(self, type: int, value: Optional[Text], context: Context) -> int:
+    def classify(self, type: int, value: Text, context: Context) -> int:
         """Turn a token into a label.  (Internal)"""
         if type == token.NAME:
             # Keep a listing of all used names
-            assert value is not None
             self.used_names.add(value)
             # Check for reserved words
             ilabel = self.grammar.keywords.get(value)
@@ -201,12 +200,10 @@ class Parser(object):
         return ilabel

     def shift(
-        self, type: int, value: Optional[Text], newstate: int, context: Context
+        self, type: int, value: Text, newstate: int, context: Context
     ) -> None:
         """Shift a token.  (Internal)"""
         dfa, state, node = self.stack[-1]
-        assert value is not None
-        assert context is not None
         rawnode: RawNode = (type, value, context, None)
         newnode = self.convert(self.grammar, rawnode)
         if newnode is not None:
```

This also has the neat side-effect of allowing me to remove some no longer necessary code.

Tightening up type annotations involving `Any` can be particularly worthwhile as it forces
mypyc to fallback to generic C code that can handle any kind of object. I was only able to
find one spot I could make this change, but it's better than nothing!

```diff
@@ -54,14 +56,14 @@ class Driver(object):
         self.logger = logger
         self.convert = convert

-    def parse_tokens(self, tokens: Iterable[Any], debug: bool = False) -> NL:
+    def parse_tokens(self, tokens: Iterable[GoodTokenInfo], debug: bool = False) -> NL:
         """Parse a series of tokens and return the syntax tree."""
         # XXX Move the prefix computation into a wrapper around tokenize.
         p = parse.Parser(self.grammar, self.convert)
         p.setup()
         lineno = 1
         column = 0
```

#### Marking everything Final

Avoiding calculating the same value over and over again is one of the most common
optimizations out there, and it's for good reason, it's usually very easy to fix. BUT,
with mypyc we can take this further as variables typed as `Final` can (often) be injected
at lookup sites at compile time, skipping the dictionary lookups at run time!

Lemme show an example, let's take this code and see what adding a single `Final` does.

```python3
from typing import Final

SCALE = 5

def calculate_y(x):
    return x * SCALE
```

Compiling it with mypyc, the C code for the `calculate_y` function is .. well .. quite
long! Notice how much work has to be done to safely look up the global value at runtime.

```c {linenos=table, hl_lines=["10-27"]}
PyObject *CPyDef_calculate_y(PyObject *cpy_r_x) {
    PyObject *cpy_r_r0;
    PyObject *cpy_r_r1;
    PyObject *cpy_r_r2;
    CPyTagged cpy_r_r3;
    PyObject *cpy_r_r4;
    PyObject *cpy_r_r5;
    PyObject *cpy_r_r6;
CPyL0: ;
    cpy_r_r0 = CPyStatic_globals;
    cpy_r_r1 = CPyStatics[3]; /* 'SCALE' */
    cpy_r_r2 = CPyDict_GetItem(cpy_r_r0, cpy_r_r1);
    if (unlikely(cpy_r_r2 == NULL)) {
        CPy_AddTraceback("final.py", "calculate_y", 6, CPyStatic_globals);
        goto CPyL4;
    }
CPyL1: ;
    if (likely(PyLong_Check(cpy_r_r2)))
        cpy_r_r3 = CPyTagged_FromObject(cpy_r_r2);
    else {
        CPy_TypeError("int", cpy_r_r2); cpy_r_r3 = CPY_INT_TAG;
    }
    CPy_DECREF(cpy_r_r2);
    if (unlikely(cpy_r_r3 == CPY_INT_TAG)) {
        CPy_AddTraceback("final.py", "calculate_y", 6, CPyStatic_globals);
        goto CPyL4;
    }
CPyL2: ;
    cpy_r_r4 = CPyTagged_StealAsObject(cpy_r_r3);
    cpy_r_r5 = PyNumber_Multiply(cpy_r_x, cpy_r_r4);
    CPy_DECREF(cpy_r_r4);
    if (unlikely(cpy_r_r5 == NULL)) {
        CPy_AddTraceback("final.py", "calculate_y", 6, CPyStatic_globals);
        goto CPyL4;
    }
CPyL3: ;
    return cpy_r_r5;
CPyL4: ;
    cpy_r_r6 = NULL;
    return cpy_r_r6;
}
```

If we mark the `SCALE` variable as Final (i.e. a constant), mypyc will notice that and
inline the value at the lookup site.

```python3 {hl_lines=[3]}
from typing import Final

SCALE: Final = 5

def calculate_y(x):
    return x * SCALE
```

Just look at how much shorter this is!

```c {linenos=table, hl_lines=[6]}
PyObject *CPyDef_calculate_y(PyObject *cpy_r_x) {
    PyObject *cpy_r_r0;
    PyObject *cpy_r_r1;
    PyObject *cpy_r_r2;
CPyL0: ;
    cpy_r_r0 = CPyTagged_StealAsObject(10);
    cpy_r_r1 = PyNumber_Multiply(cpy_r_x, cpy_r_r0);
    CPy_DECREF(cpy_r_r0);
    if (unlikely(cpy_r_r1 == NULL)) {
        CPy_AddTraceback("final.py", "calculate_y", 6, CPyStatic_globals);
        goto CPyL2;
    }
CPyL1: ;
    return cpy_r_r1;
CPyL2: ;
    cpy_r_r2 = NULL;
    return cpy_r_r2;
}
```

If the value can't be inlined, say because it involves a function call then mypyc will
just replace the globals dictionary lookup and instead use a C static (which is still
faster).

Applying this optimization to Black yields changes like the following (`black.comments`
and `black.parsing` respectively):

```diff
@@ -12,11 +18,10 @@ from black.nodes import STANDALONE_COMMENT, WHITESPACE
 # types
 LN = Union[Leaf, Node]

-
-FMT_OFF = {"# fmt: off", "# fmt:off", "# yapf: disable"}
-FMT_SKIP = {"# fmt: skip", "# fmt:skip"}
-FMT_PASS = {*FMT_OFF, *FMT_SKIP}
-FMT_ON = {"# fmt: on", "# fmt:on", "# yapf: enable"}
+FMT_OFF: Final = {"# fmt: off", "# fmt:off", "# yapf: disable"}
+FMT_SKIP: Final = {"# fmt: skip", "# fmt:skip"}
+FMT_PASS: Final = {*FMT_OFF, *FMT_SKIP}
+FMT_ON: Final = {"# fmt: on", "# fmt:on", "# yapf: enable"}


 @dataclass
```

```diff
@@ -36,6 +36,11 @@ except ImportError:
     else:
         ast3 = ast27 = ast

+if sys.version_info >= (3, 8):
+    TYPE_IGNORE_CLASSES: Final = (ast3.TypeIgnore, ast27.TypeIgnore, ast.TypeIgnore)
+else:
+    TYPE_IGNORE_CLASSES: Final = (ast3.TypeIgnore, ast27.TypeIgnore)
+

 class InvalidInput(ValueError):
     """Raised when input source code fails all parse attempts."""
@@ -160,10 +165,7 @@ def stringify_ast(

     for field in sorted(node._fields):  # noqa: F402
         # TypeIgnore has only one field 'lineno' which breaks this comparison
-        type_ignore_classes: Tuple[Type, ...] = (ast3.TypeIgnore, ast27.TypeIgnore)
-        if sys.version_info >= (3, 8):
-            type_ignore_classes += (ast.TypeIgnore,)
-        if isinstance(node, type_ignore_classes):
+        if isinstance(node, TYPE_IGNORE_CLASSES):
             break

         try:
```

In total, this was definitely the most common optimization I applied throughout this whole
journey. It's simple but effective!

#### Taking advantage of early binding

This one is really making a deal with mypyc. You give it static code and it gives you fast
code in return ... by resolving called functions and names at compile time. I already
covered using Final which is basically a specific form of early binding, but it works for
function calls too!

Time for another example:

```python3
from typing import Callable, List

def convert(item: object) -> str:
    return "<tag>" + repr(item)

def process_items(func: Callable[[object], object], items: List[object]) -> List[object]:
    return [func(i) for i in items]

process_items(convert, ["1", "2", "3"])
```

Here the function used to convert the items isn't known until call time forcing mypyc to
fall back to the standard Python calling convention (albeit it does use the faster
vectorcall convention). If I instead hardcode `convert` in `process_items`, mypyc will be
able to call the C function directly which is much faster.

```python3 {hl_lines=[7]}
from typing import Callable, List

def convert(item: object) -> str:
    return "<tag>" + repr(item)

def process_items(func: Callable[[object], object], items: List[object]) -> List[object]:
    return [convert(i) for i in items]

process_items(convert, ["1", "2", "3"])
```

```diff {hl_lines=[6, 8]}
 CPyL3: ;
     cpy_r_r8 = CPyList_GetItemUnsafe(cpy_r_items, cpy_r_r3);
     cpy_r_i = cpy_r_r8;
-    PyObject *cpy_r_r9[1] = {cpy_r_i};
-    cpy_r_r10 = (PyObject **)&cpy_r_r9;
-    cpy_r_r11 = _PyObject_Vectorcall(cpy_r_func, cpy_r_r10, 1, 0);
-    if (unlikely(cpy_r_r11 == NULL)) {
+    cpy_r_r9 = CPyDef_convert(cpy_r_i);
+    CPy_DECREF(cpy_r_i);
+    if (unlikely(cpy_r_r9 == NULL)) {
         CPy_AddTraceback("early_binding.py", "process_items", 7, CPyStatic_globals);
         goto CPyL8;
     }
```

Obviously a lot of the time this isn't possible as if your function calls are dynamic it's
probably because the code depends on it ...

By sheer luck, I was able to replace two dynamic function calls with static calls in
`blib2to3.pgen2.parse.Parser`, the very hot parser code!

```diff {hl_lines=[20, 24, 34, 43]}
diff --git a/src/blib2to3/pgen2/parse.py b/src/blib2to3/pgen2/parse.py
index 6b03188..b5da4fa 100644
--- a/src/blib2to3/pgen2/parse.py
+++ b/src/blib2to3/pgen2/parse.py
@@ -23,7 +23,7 @@ from typing import (
     Set,
 )
 from blib2to3.pgen2.grammar import Grammar
-from blib2to3.pytree import NL, Context, RawNode, Leaf, Node
+from blib2to3.pytree import convert, NL, Context, RawNode, Leaf, Node

@@ -199,16 +206,13 @@ class Parser(object):
             raise ParseError("bad token", type, value, context)
         return ilabel

     def shift(self, type: int, value: Text, newstate: int, context: Context) -> None:
         """Shift a token.  (Internal)"""
         dfa, state, node = self.stack[-1]
         rawnode: RawNode = (type, value, context, None)
-        newnode = self.convert(self.grammar, rawnode)
-        if newnode is not None:
-            assert node[-1] is not None
-            node[-1].append(newnode)
+        newnode = convert(self.grammar, rawnode)
+        assert node[-1] is not None
+        node[-1].append(newnode)
         self.stack[-1] = (dfa, newstate, node)

     def push(self, type: int, newdfa: DFAS, newstate: int, context: Context) -> None:
     @@ -221,12 +225,11 @@ class Parser(object):
     def pop(self) -> None:
         """Pop a nonterminal.  (Internal)"""
         popdfa, popstate, popnode = self.stack.pop()
-        newnode = self.convert(self.grammar, popnode)
-        if newnode is not None:
-            if self.stack:
-                dfa, state, node = self.stack[-1]
-                assert node[-1] is not None
-                node[-1].append(newnode)
-            else:
-                self.rootnode = newnode
-                self.rootnode.used_names = self.used_names
+        newnode = convert(self.grammar, popnode)
+        if self.stack:
+            dfa, state, node = self.stack[-1]
+            assert node[-1] is not None
+            node[-1].append(newnode)
+        else:
+            self.rootnode = newnode
+            self.rootnode.used_names = self.used_names
```

Also since `blib2to3.pytree.convert` is guaranteed to return a non-None value, I could
drop a branch which was neat :)

______________________________________________________________________

### Detour: let's build developer tooling!

If you've worked with me before you'll know that I *love* to write developer tooling to
make life easier. (Un)fortunately this project needed two bits of tooling that simply
didn't exist at the time:

1. A benchmark suite
1. A tool to compare two builds of Black behaviourally

The first one is pretty self-explanatory, I needed a good benchmark suite to make sure
this project would actually improve performance and also quantify the gains to compare
optimizations.

The second one is a less clear, effectively I wanted [mypy-primer] but for Black:

![mypy-primer comment on python/mypy PR 12064 describing the impact of the change](/media/mypy-primer-comment.png)

So I got to work creating [blackbench] and
[(the original) diff-shades][original-diff-shades]. Honestly, in hindsight blackbench
sucks and needs a rewrite so I don't want to go into too much detail about it, but the
summary is that it came with benchmarks for the following tasks:

- Formatting **with** safety checks
- Formatting **without** safety checks
- Blib2to3 parsing

across quite a few inputs. The story is similar for the original diff-shades, it worked
and made it possible to verify mypyc didn't change formatting, but its code was horrible
(and not to mention unmaintainable). It was bad enough that I actually rewrote the tool
later on.

> The TL;DR version of what diff-shades does is that it clones a bunch of projects, runs
> Black on 'em while recording the results. Then you'd use its other commands to analyze
> and compare recordings.

### Detour: does GCC help?

At this point I had found a workaround to the GCC error and was curious to whether using
GCC would produce faster binaries. The answer turned out to be
[*very much no*][gcc-is-worse].

> Anyway, other than getting distracted by diff-shades, I looked into the gcc issue to
> find a potential workaround. I found one, but it was useless 🙂 Collecting numbers for
> both gcc-10 and clang showed that gcc fails in basically all departments:
>
> - takes almost twice as long to compile Black AND definitely uses 200%+ more memory
> - had no meaningful difference in generated binary size all while being far more picky
>   about the C code it's given
> - oh and of course produced binaries that were around 8% slower 🙃
>
> I'm honestly happy that GCC failed to compile on that parser setup code, clang (for
> Linux) is so much better.

______________________________________________________________________

### Results

> **Quick note**
>
> I also did some micro optimizations like reordering if checks to hit the common case
> first or replacing `x = a + 1` with `x += 1`, but to this day I don't know whether they
> actually had an impact.
>
> Additionally, I did do one more [optimization round for src/black][opt-round3], but
> there's nothing in that which I haven't covered yet.

Anyhow these optimizations bumped the parsing speedup over interpreted from ~1.73x to
\~1.9x. You can see the [changes made here][opt-round1] [and also here][opt-round2].

See the tables at the bottom in particular for the columns with `compiled-mypyc-preopt`
and `compiled-mypyc` [in this report I compiled][perf-report]. It took a long time to
compile this report by the way, setting up a properly configured benchmark setup and
gathering multiple data samples is *very* time consuming!

In the end, I managed to increase compiled performance by an additional 10-15% which is
pretty nice! I was aiming for 25%, but in hindsight I might have been hoping for too much
haha. Also yes, they were virtually useless when interpreted.

## Building compiled wheels with GitHub Actions

With the mypyc branch functional and pretty fast, it was time to automate building the
wheels. Not only does this make the release process easier, but it also means I can easily
build wheels for platforms I don't have access to (eg. MacOS).

To make the CI configuration process easier I finally took a look at [cibuildwheel]. I
won't go into my exact process since it was basically trial and error + my many dumb
mistakes 😅 [^10] but here's some noteworthy takeaways:

- If you're using `setuptools-scm`, you will probably want to do a full clone on CI so the
  tag history is still available by build time

- Read cibuildwheel's documentation before writing any configuration! Seriously, if I did
  I wouldn't have to make [this commit][read-the-bloody-docs] adding `{project}` to the
  test command

- Set `CIBW_BUILD_VERBOSITY` to at least `1` because it will make debugging build errors
  (and trust me you will get some!) so much nicer

- if wheel sizes are an issue then passing `debug_level=0` to mypyc.build.mypycify might
  help by stripping all debug information. Not great if you hit a bunch of segfaults or
  similar though, so it's a tradeoff

So yeah 20-ish commits later I had a basic but functional setup which could compile wheels
for x86-64 Windows, MacOS, and Linux from CPython 3.6 to CPython 3.10. Universial2 and ARM
variants were also supported for the shiny M1 platform. The workflow is a bit slow, but
that's expected and not a big deal.

If you're curious, you can find the workflow here: [ichard26/black-mypyc-wheels]

> I don't recommend using my workflow to build your own workflows since it's a bit hacky
> and in need of a cleanup. I'd instead recommend taking a look at the
> [examples in cibuildwheel's docs][cibuildwheel-examples].

I was going to mention the community field testing campaign I carried out that had its own
package index, but honestly it was for the most part a failure as it reached no one. In
hindsight, I did not promote it enough so yeah this one is on me :)

## Stable release prep and shipping the wheels

Black for the longest time ever wasn't stable ([GH-517]), the team initially aimed to mark
Black stable in late 2018, but obviously that didn't happen for reasons. I won't go into
it too much since it'd be a whole another story, but to suffice to say when we made our
newest "commitment" (at this point we explicitly worded our intentions to *not* be
promises) we were very motivated to get it done and hopefully right.

We drafted a [stablity policy][stability-policy],
[dropped Python two support][cya-python-two],
[introduced the `--preview` flag][hi-preview], and [so much more][changelog] [^11].

In this time I rewrote diff-shades into [what it is today][diff-shades], a reliable enough
tool used to [provide direct][diff-shades-comment-1]
[feedback on PRs][diff-shades-comment-2]. I tried to integrate it into the wheel build
workflow as part of the test step, but that turned out be very painful and I backed out of
it since I had a stable release to manage and publish!

Basically building Black with mypyc usually requires disabling pip's build isolation to
work properly as mypyc is not a standard build dependency of Black. This messes up the
installation of diff-shades which is a flit packaged tool. So I hacked up a script to edit
the `[build-system].requires` field in `pyproject.toml` to include mypyc pre-build, but
the isolation somehow broke the linker search path or something.

```text
clang -Wno-unused-result -Wsign-compare -DNDEBUG -g -fwrapv -O3 -Wall -g0 -fPIC -I/tmp/pip-build-env-9bfrmy6j/overlay/lib/python3.7/site-packages/mypyc/lib-rt -Ibuild -I/opt/python/cp37-cp37m/include/python3.7m -c build/__native_f2d4935fd652bc9ef29d.c -o build/temp.linux-x86_64-3.7/build/__native_f2d4935fd652bc9ef29d.o -O3 -Werror -Wno-unused-function -Wno-unused-label -Wno-unreachable-code -Wno-unused-variable -Wno-unused-command-line-argument -Wno-unknown-warning-option
    clang -shared -g0 build/temp.linux-x86_64-3.7/build/__native_f2d4935fd652bc9ef29d.o -o build/lib.linux-x86_64-3.7/f2d4935fd652bc9ef29d__mypyc.cpython-37m-x86_64-linux-gnu.so
    /usr/bin/ld: cannot find crtbeginS.o: No such file or directory
    /usr/bin/ld: cannot find -lgcc
    /usr/bin/ld: cannot find -lgcc_s
    clang: error: linker command failed with exit code 1 (use -v to see invocation)
    error: command '/usr/bin/clang' failed with exit code 1
```

I'm honestly still not quite sure what's wrong.

Anyway since I was like an hour or more into build errors, I just decided to call it done
and fall back to the basic testing that was already working.[^12] I triggered the last
workflow run of the day, downloaded the artifacts, tested one of them locally to make sure
nothing was on fire, and pushed 'em to PyPI and that's how release 22.1.0 was born on
PyPI.

The core team chat was quite lively for the next while to say the least, after all we just
finished a major milestone! Also yeah we *may* have done a lot of publicizing for this
release which is why this was everywhere for a while :wink:

### Post release calm

To be honest I haven't seen anyone really comment on the fact Black is now compiled with
mypyc yet short this one [mini blog post][that-one-post-about-black-mypyc] ... which is
both disappointing since I spent so much time on this effort, but also calming since
usually buzz is sparked by bugs and whatnot.

Having mentioned bugs and crashes, so far we've received two reports,
[one bug probably from mypyc and an unintentional restriction on Black's unofficial APIs][not-too-many-fires-so-far].

Although version 22.1.0 still unintentionally broke quite a few integrations though they
weren't mypyc related. Turns out lots of people depend on `black.files.find_project_root`
returning a Path! I've seen at least five issues / PRs on GitHub fixing crashes related to
our change making it return a tuple instead.

For this reason we, the core team, want to [define a stable API (GH-779)][gh-779] soon. I
can't promise anything as Black's development is volunteer-based, but if you were curious
to what's next for psf/black, there's this.

## Results & final thoughts

**Black is now overall 2 times faster**[^1]. And as a bonus, startup time is down too (at
most by 15%). You can [read the whole report here][perf-report]. Please note these numbers
were gathered quite a long time ago and **they are probably a bit outdated**, especially
with the recent blib2to3 changes made to support 3.10 syntax.

From my experience, I believe mypyc will make for an interesting but viable solution to
speeding up Python code going forward. It's far from perfect and still has a bunch of
bugs, but today it is already making an impact. Should you use it in production if you
don't want to feel like being a beta tester? Probably not, but it's sure still possible
and I'm happy with the results.

I hope to see the mypyc project gain more contributors and succeed. Along with Cython,
PyPy, and the recent faster-cpython project, **perhaps Python will have a solid speed
story**.

> And yes, I did indeed learn a ton embarking on this project. Made mistakes along the
> way, but they turned out alright ❀

______________________________________________________________________

### Acknowledgements

I'd like to thank [@msullivan] for his original work on integrating mypyc into Black
spawning this summer project and eventually this blog post :)

$try-getting-this-reviewed-by-mypyc-contributor

$try-getting-this-reviewed-by-co-maintainers

[^1]: Originally when I first landed the relevant PR it was an overall 2x improvement but
    once Jelle added a stability hotfix the *effective* speedup for files that were
    changed is 50%. If you're formatting a bunch of already well formatted files, the
    speedup is still 2x

[^2]: There's too many typing related PEPs to copy and paste, but there's an up to date list
    here: <https://docs.python.org/3/library/typing.html#relevant-peps>

[^3]: <https://mypyc.readthedocs.io/en/stable/introduction.html#introduction>

[^4]: well not quite exactly this, it was a crapper solution with a time complexity of O(n²)
    instead of O(n), oh and of course my code style was less than ideal and I didn't use
    type annotations, but let's not go there :)

[^5]: I am well aware I don't need to import `List` or `Tuple` from `typing` anymore. I'm
    just keeping my code examples compatible with older versions of Python.

[^6]: At the time @ambv wrote Black, lib2to3 was the best AST + CST mix out there. AST
    parsers like the built-in one are great for understanding syntax and structure but are
    useless when editing how much whitespace a NAME token gets for example. CSTs on the
    other hand are great for visual changes Black does, but don't provide enough context
    to allow Black to make the right decisions, hence lib2to3 and the fork.

[^7]: What's funny is that the PR which added this `__name__` check had a review noting the
    hackiness of reading the attribute in the first place. It was dismissed as it would
    have taken too much effort to properly implement the logic, well turns out we got this
    fun code in return.

[^8]: It's because internally mypyc implements nested functions as callable classes

[^9]: Reading the results searching `from:ichard26#4772 in:black-formatter mypyc` in the
    Python Discord server nets some interesting discussion and history :p

[^10]: If you are in need of some quality entertainment, look at this commit history:
    <https://github.com/ichard26/black-mypyc-wheels/commits/main>

[^11]: I would link to the 22.1.0 heading but currently the HTML IDs are uhhh ... terrible
    and unfixed. I've tried fixing this with a custom sphinx extension and failed, I have
    other ideas left to try, but yeah :(

[^12]: I was frustrated and had a headache thanks to the stress 🙃

[@jellezijlstra]: https://github.com/JelleZijlstra
[@msullivan]: https://github.com/msullivan
[ambv-mypyc-suggestion]: https://github.com/psf/black/issues/366#issuecomment-490153026
[blackbench]: https://github.com/ichard26/blackbench
[changelog]: https://black.readthedocs.io/en/stable/change_log.html
[cibuildwheel]: https://cibuildwheel.readthedocs.io/en/stable/
[cibuildwheel-examples]: https://cibuildwheel.readthedocs.io/en/stable/working-examples/
[cya-python-two]: https://github.com/psf/black/pull/2740
[diff-shades]: https://github.com/ichard26/diff-shades
[diff-shades-comment-1]: https://github.com/psf/black/pull/2814#issuecomment-1023219426
[diff-shades-comment-2]: https://github.com/psf/black/pull/2726#issuecomment-1019067134
[gcc-bug-code]: https://github.com/psf/black/blob/8ea641eed5b9540287a8e9a9afa1458b72b9b630/src/blib2to3/pgen2/parse.py#L117-L139
[gcc-is-worse]: https://github.com/ichard26/black-mypyc-wheels/issues/2#issuecomment-896357830
[gh-366]: https://github.com/psf/black/issues/366
[gh-517]: https://github.com/psf/black/issues/517
[gh-779]: https://github.com/psf/black/issues/779
[gprof2dot]: https://github.com/jrfonseca/gprof2dot
[hi-preview]: https://github.com/psf/black/issues/2751
[ichard26/black-mypyc-wheels]: https://github.com/ichard26/black-mypyc-wheels
[initial-results]: https://github.com/psf/black/issues/366#issuecomment-855306152
[integration-report]: https://github.com/mypyc/mypyc/issues/886
[line_profiler]: https://pypi.org/project/line-profiler/
[mypy]: https://mypy.readthedocs.io/en/stable/introduction.html
[mypy-primer]: https://github.com/hauntsaninja/mypy_primer
[mypyc]: https://mypyc.readthedocs.io/en/stable/introduction.html
[mypyc#862]: https://github.com/mypyc/mypyc/issues/862
[mypyc#885]: https://github.com/mypyc/mypyc/issues/885
[mypyc-deviations-one]: https://mypyc.readthedocs.io/en/stable/differences_from_python.html
[mypyc-deviations-two]: https://mypyc.readthedocs.io/en/stable/native_classes.html
[not-too-many-fires-so-far]: https://github.com/psf/black/issues/2846
[opt-round1]: https://github.com/psf/black/pull/2431/commits/f6a3e788bb8714e41fe0a4cc1ee2058b0a7cb3ac
[opt-round2]: https://github.com/psf/black/pull/2431/commits/911d0d8601318fcc04069f2af91a066e499f0db0
[opt-round3]: https://github.com/psf/black/pull/2431/commits/c7de2eafb5d07033429ab3a18ce02ed093b645c5
[original-diff-shades]: https://github.com/ichard26/black-mypyc-wheels/blob/c448ae49df7570dc2745eccd947625897f6541ce/diff_shades.py
[original-pr]: https://github.com/psf/black/pull/1009
[perf-report]: https://gist.github.com/ichard26/b996ccf410422b44fcd80fb158e05b0d
[py-spy]: https://github.com/benfred/py-spy
[read-the-bloody-docs]: https://github.com/ichard26/black-mypyc-wheels/commit/643de450e252e74040ba14fd066ea8bc23c0b0d7
[release-22.1.0]: https://github.com/psf/black/releases/tag/22.1.0
[scalene]: https://pypi.org/project/scalene/
[stability-policy]: https://black.readthedocs.io/en/latest/the_black_code_style/index.html#stability-policy
[that-one-post-about-black-mypyc]: https://simonwillison.net/2022/Jan/30/mypyc/
[traits-docs]: https://mypyc.readthedocs.io/en/latest/using_type_annotations.html#trait-types
[vtables]: https://en.wikipedia.org/wiki/Virtual_method_table
[yelp-gprof2dot]: https://pypi.org/project/yelp-gprof2dot/
