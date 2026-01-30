---
title: What's new in pip 26.0 - prerelease and upload-time filtering!
slug: whats-new-in-pip-26.0
description: &desc >
  pip 26.0 includes support for reading requirements from inline script metadata, excluding
  distributions by upload time, per-package prerelease selection, and experimental support for
  in-process build dependencies.
summary: *desc
date: 2026-01-30
tags: [pip, release]
showToc: true
---

On January 30, 2026, we, the pip team, released pip 26.0.

As always, please [consult the changelog][changelog] to learn about all of the changes
contained in this release.

Also, I apologise for the lack of a release post for pip 25.3. Changes have occurred in my
personal life that have resulted in me having very little free time. I was thus unable to find
the time to write a post in time for pip 25.3. Please enjoy this post for 26.0.

## New features ✨

### Excluding distributions by upload time

The new `--uploaded-prior-to` option allows you to filter packages by their upload time to
a package index, only considering packages that were uploaded before a specified datetime.
This can be useful for debugging broken dependencies and ensuring reproducible builds by
ensuring you only install packages that were available at a known (good) point in time.

A common situation is that you have a years old project (with no lockfile) and attempting
installing the dependencies results in a broken install. With `--uploaded-prior-to`, you
can exclude packages uploaded after the last time you know the install worked and reobtain
the same set of working dependencies.

The option accepts ISO 8601 datetime strings in several formats:

- `2025-03-16` - Date in implicit local timezone
- `2025-03-16T12:30:00` - Datetime in implict local timezone
- `2025-03-16T12:30:00Z` - Datetime in UTC
- `2025-03-16T12:30:00+05:00` - Datetime with local timezone offset

For consistency, the timezone should be explicitly provided, although it is not required.

```console { .command }
pip install -r requirements.txt --uploaded-prior-to 2024-03-16T12:30:00+05:00

[...]
Installing collected packages: platformdirs, pathspec, packaging, mypy-extensions, click, black
Successfully installed black-24.3.0 click-8.1.7 mypy-extensions-1.0.0 packaging-24.0 pathspec-0.12.1 platformdirs-4.2.0
```

> [!WARNING] Warning: it is an exclusive upper bound!
> While similar to uv's `--exclude-newer` option,
> `--uploaded-prior-to` behaves a bit differently. Notably, `--uploaded-prior-to` is an
> exclusive upper bound for both datetimes and dates. In other words,
> `--uploaded-prior-to 2025-01-01` is equivalent to `--uploaded-prior-to 00:00:00`, not
> `2025-01-01 23:59:59` as with `--exclude-newer`.

[More information can be found in the pip user guide][prior-to-docs]. We thank uv for the
inspiration with their `--exclude-newer` option.

### Separate build constraints

This feature actually shipped in pip 25.3, but it's noteworthy enough to mention in this
post.

Build constraints can be set independently with `--build-constraint`. This allows
constraining the versions of packages used during the build process (e.g., setuptools)
without affecting the final installation.

This can be useful when a new release of a build backend is causing a package build to
error. You can constrain (block) the problematic versions of the backend and continue with
happily installing whatever you were installing earlier.

```python
# constraints.txt
setuptools != 80.0.0  # imagine this version broke your build
```

```console {.command}
pip install myproject.tar.gz --build-constraint constraints.txt
```

Previously, your best option to constrain build dependencies was through setting the
`PIP_CONSTRAINT` environment variable. Unlike the `--constraint` option, the envvar would
be picked up by the pip subprocess invoked to install build dependencies. This works, but
is quite obscure and has the side-effect of constraining the final installation as well.

At some point, we would like to stop calling pip in a subprocess to install build
dependencies and [switch to installing them in-process][inprocess-draft]. This has a bunch
of benefits, but would break the `PIP_CONSTRAINT` workaround. While we could update the
`--constraint` option to also influence build dependencies , we are using this transition
as an opportunity to clean up constraint handling.[^uv-compat]

As part of this transition, pip has deprecated using `PIP_CONSTRAINT` to constrain build
dependencies. Affected users are encouraged to use `--build-constraint` or set
`PIP_BUILD_CONSTRAINT`.

### `--only-final` and `--all-releases`

By default, pip installs stable versions of packages, unless their specifier includes a
pre-release version (e.g., `SomePackage>=1.0a1`) or if there are no stable[^stable] versions
available that satisfy the requirement. The `--all-releases` and `--only-final` options
provide **per-package control** over pre-release selection.

As their names imply, these new options allow you to tell pip to only consider stable or all
versions when resolving packages. They function like their `--no-binary` and `--prefer-binary`
cousins.

```shell
# accept the bleeding edge for a dependency
python -m pip install --all-releases=DependencyPackage SomePackage
# we only accept stable versions for this dependency
python -m pip install --only-final=DependencyPackage SomePackage
```

The pre-existing `--pre` flag is now equivalent to `--all-releases :all:`.

For more information, [please consult the pip user guide][prerelease-docs].

### Installing from inline script metadata (PEP 723)

This release has pip gain the `--requirements-from-script` option for installing
dependencies declared in [inline script metadata] (PEP 723). Inline script metadata is
useful for scripts that depend on external libraries, but are designed to be distributed
as a single file.

Recently, I had to write a script for converting a Jekyll site into a form that could be
imported by the [Ghost CMS]. Jekyll posts are stored as Markdown files with YAML
frontmatter, which meant I had to pull in PyYAML and a Markdown parser. This is no
problem. I can simply declare these dependencies using a `script` metadata comment. No
separate `requirements.txt` needed.

```python
"""
Archive Jekyll posts into a JSON file Ghost can import.
"""

# /// script
# requires-python = ">=3.11"
# dependencies = [
#   "click",
#   "rich",
#   "python-frontmatter",
#   "markdown-it-py[plugins]",
#   "PyYaml",
# ]
# ///

import json
import subprocess
import zipfile

# and so on ...
```

To run this script, I can simply run `pipx run ghost_export.py`. pipx will see the script
comment and auto-install the dependencies before executing the script. This is extremely
convenient![^uv-run]

> [!NOTE] Why a new option?
> This is a good time to explain why this feature is its own option.
> While we could've overloaded the `-r` option as suggested by some,
> `--requirements-from-script` is *not* equivalent to `-r`. If `requires-python` is
> declared in a script metadata block, pip will validate that the Python version is
> compatible as well.

And while we think that using pip with inline script metadata is suboptimal (you should
really be using a proper script runner), we recognise that there are situations where such
script runners are unavailable or undesirable.

`--requirements-from-script` can also be given with `pip download`, `pip lock`, and
`pip wheel`.

Thank you to [@SnoopJ] for contributing this feature to pip!

### Experimental: In-process build dependencies

The experimental feature to install build dependencies in-process, originally slated for
25.3, has now landed in pip 26.0. You should
[read my pip 25.2 post for the rational for this change](/blog/2025/07/whats-new-in-pip-25.2/#sneak-peek-in-process-build-dependencies),
but the benefits include:

- Faster build environment provisioning
- More reliable and less confusing option and flag inheritance
- In the future: support for authentication prompts while installing build dependencies

It can be enabled with the `--use-feature=inprocess-build-deps` flag, although the plan is
enable it by default in a future release once the feature stabilizes.

## Deprecations & upcoming removals

Here's the current list of current deprecations with the release in which they are
scheduled for removal. As always, any given removal may be **pushed to a future release as
needed**.

`PIP_CONSTRAINT` for build dependencies *To be removed in pip 26.1*
: The `PIP_CONSTRAINT` envvar will eventually stop taking effect for build dependencies.
  We're making this change to accommodate the planned transition to installing build
  dependencies in-process. Affected users should use `--build-constraint` or
  `PIP_BUILD_CONSTRAINT`.

[^uv-run]: You can use `uv run` as well.

[^uv-compat]: This also has the nice benefit of aligning our constraint options with uv which
    already has `--build-constraint`.
  
[^stable]: What is a stable version? Any package whose version does NOT contain a development
  segment or pre-release segment (`.alpha`, `.beta`, `.rc`, `.dev`).

[@snoopj]: https://github.com/SnoopJ
[changelog]: https://pip.pypa.io/en/latest/news/#v25-2
[dpo]: https://discuss.python.org/t/announcement-pip-25-2-release/100716
[ghost cms]: https://ghost.org
[inline script metadata]: https://packaging.python.org/en/latest/specifications/inline-script-metadata/#inline-script-metadata
[inprocess-draft]: https://github.com/pypa/pip/pull/13450
[prior-to-docs]: https://pip.pypa.io/en/latest/user_guide/#filtering-by-upload-time
[prerelease-docs]: https://pip.pypa.io/en/latest/user_guide/#controlling-pre-release-installation
