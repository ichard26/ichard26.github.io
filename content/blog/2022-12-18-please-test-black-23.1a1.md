---
slug: black-23.1a1
title: Black 23.1a1 - please help us test the 2023 stable style!
description: &desc We just released Black 23.1a1 with the first draft of the 2023
  stable style, please try it out and let us know your feedback and concerns.
summary: *desc
modified: 2023-01-03 22:30:00-05:00
tags: [black, release]
showToc: true
---

As the post summary says, earlier today [we released Black 23.1a1][gh-release]. You might
be wondering why we released an alpha... it's because the upcoming Black 23.1.0 release
will see many **preview** code style changes promoted to the **stable** style.

In other words, **Black 23.1.0 will format your code differently out of the box**.

> If you're surprised we're doing this, our [stability policy][stability-policy] allows us
> to update the stable style in the new year. This minimises disruption while keeping the
> door for improvements open.

23.1a1 contains a draft of the 2023 stable style that we, the maintainers, initially
decided on.

However, **we need your help in finalizing what goes into the 2023 stable style**. If you
can, please try 23.1a1 out on your codebase(s). You can install it by running this
command:

```bash {.command}
python -m pip install black==23.1a1
```

**If you have any feedback, concerns, or run into any issues, please drop us a comment
[in this issue][#3407]** (<https://github.com/psf/black/issues/3407>).

> **Note**: Please read the start of the issue description before commenting. It will make
> the lives of everyone easier.

What follows is brief rundown of all of the changes in the preview style that were
promoted to stable in 23.1a1.

## Promoted changes

### Bugfixes

#### `--skip-string-normalization` now prevents docstring prefixes from being normalized

<!-- 1 -->

`-S` / `--skip-string-normalization` stops Black from normalizing string prefixes and
quotes, except when it doesn't...

Sometime ago I accidentally added a bug where Black would normalize docstring prefixes
even if you told it not to, leading to this:

```diff
 def add(a, b):
-    F"""Add two numbers and return the result."""
+    f"""Add two numbers and return the result."""
     return a + b
```

[This has been fixed in the preview style since 22.8.0][#3168] (leaving this example
unchanged).

#### `--skip-magic-trailing-comma` now removes useless trailing commas in subscripts

<!-- 2 -->

Black has the concept of "magic trailing commas."
[You can read about it here][magic-trailing-comma], but in essence, Black treats trailing
commas as a sign that collections, calls, or function signatures should always be
exploded.

```python3
# The trailing comma will stop Black from collapsing this collection into one line.
TRANSLATIONS = {
    "en_us": "English (US)",
    "pl_pl": "polski",
}
```

This can be disabled passing the `-C` / `--skip-magic-trailing-comma` flag.[^1]

However, Black didn't remove unnecessary trailing commas in all situations. In the
following example, the trailing comma in the function signature was removed as expected,
but not the subscript.

```diff
 Point = Tuple[int, int,]

-def f(a: Point, b: Point,): ...
+def f(a: Point, b: Point): ...
```

[This was fixed in the preview style since 22.8.0][#3209], which removes both commas as
expected.

```diff
-Point = Tuple[int, int,]
+Point = Tuple[int, int]

-def f(a: Point, b: Point,): ...
+def f(a: Point, b: Point): ...
```

#### Correctly handle trailing commas that are inside a line's leading non-nested parens

<!-- 18 -->

There are actually two problems in the "before" output:

- The `zero(one,).two(three,).four(five,)` chain should be fully exploded
- The magic trailing comma in the dictionary in `refresh_token()` is being ignored

**Source:**

```python3
zero(one,).two(three,).four(five,)


def refresh_token(self, device_family, refresh_token, api_key):
    return self.get(
        data={
            "refreshToken": refresh_token,
        },
        api_key=api_key,
    )["sdk"]["token"]
```

**Before:**

```python3
zero(one,).two(three,).four(
    five,
)


def refresh_token(self, device_family, refresh_token, api_key):
    return self.get(data={"refreshToken": refresh_token,}, api_key=api_key,)[
        "sdk"
    ]["token"]
```

**After**:

```python3
zero(
    one,
).two(
    three,
).four(
    five,
)


def refresh_token(self, device_family, refresh_token, api_key):
    return self.get(
        data={
            "refreshToken": refresh_token,
        },
        api_key=api_key,
    )[
        "sdk"
    ]["token"]
```

[Both these issues have been fixed in the preview style since 22.12.0][#3370].

### End quotes that violate the line length limit in multiline docstrings will be moved

<!-- 5 -->

```python3 { linenos=table, hl_lines=[3] }
def process_and_count(text):
    """
    Remove punctuation and return cleaned string, in addition to its length in tokens."""
    pass


def example_2(text):
    """Remove punctuation and return cleaned string in addition to its length in tokens."""
    pass
```

Here, the third line violates the line length limit (90 > 88) because of the end quotes.
Earlier versions of Black would leave this be which wasn't ideal, so now they are moved
onto their own line.

```diff
 def process_and_count(text):
     """
-    Remove punctuation and return cleaned string, in addition to its length in tokens."""
+    Remove punctuation and return cleaned string, in addition to its length in tokens.
+    """
     pass
```

However, Black won't move quotes of single line docstrings (such as with `example_2()`)
since that would look ugly.

This was [added to the preview style in 22.8.0][#3044] and
[then tweaked in 23.1a1][#3430].

### Better management of parentheses

Here's a set of changes that together improve Black's handling of parentheses in various
situations.

Out of these five,
["Remove redundant (outermost) parentheses in for statements"](#remove-redundant-outermost-parentheses-in-for-statements)
will probably be the most impactful.

<!-- 6, 7, 8, 11, 12 -->

#### Return type annotations

**Source:**

```python3
def foo() -> (int):
    return 2


def foo() -> intsdfsafafafdfdsasdfsfsdfasdfafdsafdfdsfasdskdsdsfdsafdsafsdfdasfffsfdsfdsafafhdskfhdsfjdslkfdlfsdkjhsdfjkdshfkljds:
    return 2


def frobnicate() -> ThisIsTrulyUnreasonablyExtremelyLongClassName | list[ThisIsTrulyUnreasonablyExtremelyLongClassName]:
    pass
```

**Before:**

```python3
def foo() -> (int):
    return 2


def foo() -> intsdfsafafafdfdsasdfsfsdfasdfafdsafdfdsfasdskdsdsfdsafdsafsdfdasfffsfdsfdsafafhdskfhdsfjdslkfdlfsdkjhsdfjkdshfkljds:
    return 2


def frobnicate() -> ThisIsTrulyUnreasonablyExtremelyLongClassName | list[
    ThisIsTrulyUnreasonablyExtremelyLongClassName
]:
    pass
```

**After** ([22.6.0+][#2990]):

```python3
def foo() -> int:
    return 2


def foo() -> (
    intsdfsafafafdfdsasdfsfsdfasdfafdsafdfdsfasdskdsdsfdsafdsafsdfdasfffsfdsfdsafafhdskfhdsfjdslkfdlfsdkjhsdfjkdshfkljds
):
    return 2


def frobnicate() -> (
    ThisIsTrulyUnreasonablyExtremelyLongClassName
    | list[ThisIsTrulyUnreasonablyExtremelyLongClassName]
):
    pass
```

#### Remove redundant parentheses around awaited objects

**Source:**

```python3
async def main():
    await (asyncio.sleep(1))
    await (set_of_tasks | other_set)
```

**Before:** unchanged.

**After** ([22.6.0+][#2991]):

```python3
async def main():
    await asyncio.sleep(1)
    await (set_of_tasks | other_set)
```

#### Remove redundant parentheses in with statements

**Source:**

```python3
with (open("bla.txt")):
    pass

with (open("bla.txt")), (open("bla.txt")):
    pass

with (open("bla.txt") as f):
    pass

with (open("bla.txt")) as f:
    pass
```

**Before:** unchanged.

**After** ([22.6.0+][#2926]):

```python3
with open("bla.txt"):
    pass

with open("bla.txt"), open("bla.txt"):
    pass

with open("bla.txt") as f:
    pass

with open("bla.txt") as f:
    pass
```

#### Remove redundant parentheses from except clauses

**Source:**

```python3
try:
    a.something
except (AttributeError) as err:
    raise err
```

**Before:** unchanged.

**After** ([22.3.0+][#2939]):

```python3
try:
    a.something
except AttributeError as err:
    raise err
```

#### Remove redundant (outermost) parentheses in for statements

**Source:**

```python3
for (k, v) in d.items():
    print(k, v)
```

**Before:** unchanged.

**After** ([22.3.0+][#2945]):

```python3
for k, v in d.items():
    print(k, v)
```

Unfortunately, nested parentheses are still left untouched.
[I have an open PR to fix that][#3243], but it isn't ready yet.

### Remove blank lines after code block open

<!-- 9 -->

Here's another change that is likely to impact your codebase.

**Note**: This new feature will be applied to **all code blocks**: `def`, `class`, `if`,
`for` , `while`, `with`, `case` and `match`.

**Source:**

```python3
def foo():



    print("All the newlines above me should be deleted!")


if condition:

    print("No newline above me!")

    print("There is a newline above me, and that's OK!")


class Point:

    x: int
    y: int

    def as_tuple(self) -> tuple[int, int]:
        return (self.x, self.y)
```

**Before:**

```python3
def foo():

    print("All the newlines above me should be deleted!")


if condition:

    print("No newline above me!")

    print("There is a newline above me, and that's OK!")


class Point:

    x: int
    y: int

    def as_tuple(self) -> tuple[int, int]:
        return (self.x, self.y)
```

**After** ([22.6.0+][#3035]):

```python3
def foo():
    print("All the newlines above me should be deleted!")


if condition:
    print("No newline above me!")

    print("There is a newline above me, and that's OK!")


class Point:
    x: int
    y: int

    def as_tuple(self) -> tuple[int, int]:
        return (self.x, self.y)
```

### Ignore magic trailing comma for single-element subscripts

<!-- 13 -->

Black exempts single-element tuple literals from the usual handling of magic trailing
commas: `(1,)` will not have a newline added (as would happen for lists, sets, etc.).

However, if you wrote `tuple[int,]` (to make it more visually distinctive to `list[int]`),
Black would explode it, which doesn't look great.

**Source:**

```python3
shape: tuple[int,]
shape = (1,)
```

**Before:**

```python3
shape: tuple[
    int,
]
shape = (1,)
```

**After** ([22.3.0+][#2942]): unchanged.

### Enforce empty lines before classes/functions with sticky leading comments

<!-- 15 -->

The old ("before") behaviour here caused flake8 to raise `E302`.

**Source:**

```python3
some_var = 1
# Leading sticky comment
def some_func():
    ...


def get_search_results(self, request, queryset, search_term):
    """
    Return a tuple containing a queryset to implement the search
    and a boolean indicating if the results may contain duplicates.
    """
    # Apply keyword searches.
    def construct_search(field_name):
        ...

    ...
```

**Before:** unchanged.

**After** ([22.12.0+][#3302]):

```python3
some_var = 1


# Leading sticky comment
def some_func():
    ...


def get_search_results(self, request, queryset, search_term):
    """
    Return a tuple containing a queryset to implement the search
    and a boolean indicating if the results may contain duplicates.
    """

    # Apply keyword searches.
    def construct_search(field_name):
        ...

    ...
```

### Empty and whitespace-only files are now normalized

Currently under the stable style, empty and whitespace-only files are not modified. In
some discussions ([issues][#2382] and [pull requests][#2484]), the consesus was to
reformat whitespace-only files to empty or single-character files, preserving the original
line endings (LR/CRLR) when possible.

In the end, we settled on these rules:

- Empty files are left as is
- Whitespace-only files (no newline) reformat into empty files
- Whitespace-only files (1 or more newlines) reformat into a single newline character

[This was added to the preview style in 22.12.0][#3348].

### Spyder cell separator comments (`#%%`) are now normalized

<!-- 10 -->

Black adds a space after the pound character in comments as per PEP 8. This caused trouble
as many editors have special comments for various functionality (eg. denoting regions in
the file) which could not contain any spaces.

To play with other tools, Black leaves Spyder's special `#%%` comments (among other
special comments) alone.

Sometime later, Spyder started to recognize `# %%` alongside `#%%`. So
[in Black 22.3.0, the preview style was changed][#2919] to start adding spaces in `#%%`
comments.

## What is not being promoted?

<!-- 3, 4, 14, 17 -->

*Ideally I would've provided examples for these changes too, but I need to get this post
out quickly and I'm getting tired. I've added links for further reading if you're
curious.*

[**Experimental string processing**][esp-pr] (aka ESP) is not being promoted to the 2023
stable preview. There's a lot of reasons why, but
[Jelle summarized it well in this comment][jelle-esp-comment]:

> \[...\] At this point there's still half a dozen stability bugs with experimental string
> processing, so that alone is enough to block it from going into the 2023 stable style.
> However, I think we should consider dropping (most of) the feature entirely, instead of
> leaving it behind a flag forever, since there are so many stability bugs and often it
> arguably makes for worse output. That should be discussed in [#2188], though.

And these three depend on ESP being promoted:

- [Wrap implicitly concatenated strings inside a list, set, or tuple in parentheses (#3162)][#3162]
- [Wrap implicitly concatenated strings used as function args in parentheses (#3307)][#3307]
- [Fix a string merging/split issue caused by standalone comments (#3227)][#3227]

Additionally, two other changes are currently not slated for promotion and will remain in
the preview style until 2024, mostly since they were landed too late in the year to allow
for enough testing. These are:

- [For assignment statements, prefer splitting the right hand side if the left hand side fits on a single line (#3368)][#3368]
  \~ *available in 22.12.0+*
- [Improve long values in dict literals (#3440)][#3440] ~ *available on main and in 23.1a1
  only*

## "I want a *new* change to the stable style"

If you want a change that's not covered by something in the preview style already (and
can't be achieved by simply tweaking a pre-existing style change), check the issue tracker
for a issue that covers what you want. If you don't find one,
[file an issue here][file-style-issue] or [drop a comment here][#3407].

Although, if you're hoping to squeeze in a new major style change into the 2023 **stable**
style, that probably won't be possible being so late in the year. It can always be
promoted to stable for 2024.

[^1]: There's some interesting history to why we added this flag, but I don't have time to
    get into it right now.

[#2188]: https://github.com/psf/black/issues/2188
[#2382]: https://github.com/psf/black/issues/2382
[#2484]: https://github.com/psf/black/pull/2484
[#2919]: https://github.com/psf/black/pull/2919
[#2926]: https://github.com/psf/black/pull/2926
[#2939]: https://github.com/psf/black/pull/2939
[#2942]: https://github.com/psf/black/pull/2942
[#2945]: https://github.com/psf/black/pull/2945
[#2990]: https://github.com/psf/black/pull/2990
[#2991]: https://github.com/psf/black/pull/2991
[#3035]: https://github.com/psf/black/pull/3035
[#3044]: https://github.com/psf/black/pull/3044
[#3162]: https://github.com/psf/black/pull/3162
[#3168]: https://github.com/psf/black/pull/3168
[#3209]: https://github.com/psf/black/pull/3209
[#3227]: https://github.com/psf/black/pull/3227
[#3243]: https://github.com/psf/black/pull/3243
[#3302]: https://github.com/psf/black/pull/3302
[#3307]: https://github.com/psf/black/pull/3307
[#3348]: https://github.com/psf/black/pull/3348
[#3368]: https://github.com/psf/black/pull/3368
[#3370]: https://github.com/psf/black/pull/3370
[#3407]: https://github.com/psf/black/issues/3407
[#3430]: https://github.com/psf/black/pull/3430
[#3440]: https://github.com/psf/black/pull/3440
[esp-pr]: https://github.com/psf/black/pull/1132
[file-style-issue]: https://github.com/psf/black/issues/new?assignees=&labels=T%3A+design&template=style_issue.md&title=
[gh-release]: https://github.com/psf/black/releases/tag/23.1a1
[jelle-esp-comment]: https://github.com/psf/black/issues/3407#issuecomment-1345416311
[magic-trailing-comma]: https://black.readthedocs.io/en/stable/the_black_code_style/current_style.html#the-magic-trailing-comma
[stability-policy]: https://black.readthedocs.io/en/stable/the_black_code_style/index.html#stability-policy
