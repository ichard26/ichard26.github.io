---
title: What's new in pip 24.3
slug: whats-new-in-pip-24.3
description: &desc >-
  pip 24.3 is a small release with a truststore bugfix, error QoL improvements, and
  one
  minor
  deprecation of noncompliant wheel filenames.
summary: *desc
date: 2024-11-14
modified: 2024-11-17
tags: [pip, release]
showToc: true
---

On October 27, 2024, [the pip team released pip 24.3]. This release was a significantly
smaller release with only a handful of minor fixes and improvements.

This write-up also includes the hotfixes included in pip 24.3.1 released on the same day.

Thank you to Damian Shaw, Hugo van Kemenade, and Stéphane Bidoul for reviewing the draft
of this write-up.

## PSA: Legacy editable installs are deprecated

> TL;DR, **don't panic**. The `-e` option is _not_ deprecated, but the way it works under
> the hood will change, potentially necessitating changes to how your project is packaged.

If you aren't aware, since pip 24.2, **legacy** editable installs are deprecated and
**support is scheduled for removal in pip 25.0 in the new year** (at the time of writing).

```console {.command hl_lines=[5]}
pip install -e temp/pip-test-package
Obtaining file:///home/ichard26/dev/oss/pip/temp/pip-test-package
  Preparing metadata (setup.py) ... done
Installing collected packages: version_pkg
  DEPRECATION: Legacy editable install of version_pkg==0.1 from file:///home/ichard26/dev/oss/pip/temp/pip-test-package (setup.py develop) is deprecated. pip 25.0 will enforce this behaviour change. A possible replacement is to add a pyproject.toml or enable --use-pep517, and use setuptools >= 64. If the resulting installation is not behaving as expected, try using --config-settings editable_mode=compat. Please consult the setuptools documentation for more information. Discussion can be found at https://github.com/pypa/pip/issues/11457
  Running setup.py develop for version_pkg
Successfully installed version_pkg-0.1
```

Editable installs are _not_ deprecated, but pip will stop running the setuptools specific
`setup.py` file under the hood. Instead, pip is transitioning to exclusively using the
standardized mechanism to request an editable install from the project's backend defined
in [PEP 660].

If you're confused, don't worry. The implementation details are admittedly a bit complex,
but all you need to know is that:

- You will see this deprecation notice **if the package** you are trying to install has
  **setuptools as its build backend AND**:
  - **EITHER this package does not have a valid [build system declaration]**, thus pip
    will opt-out of the modern mechanisms and run `setup.py` directly as required
  - **OR this package’s build system declaration explicitly requests a version setuptools
    older than version 64** which lacks support for the modern editable installation
    mechanism (PEP 660)
- The easiest way to address the deprecation is to **add a `pyproject.toml` file that
  declares your package's build system to be setuptools** (see the guide
  ["How to modernize a setup.py based project?"] for more details)
- **You can keep your `setup.py` file.** It's a configuration file for setuptools, and
  setuptools still supports it. pip simply won't be running it directly anymore,
  delegating to setuptools (see the discussion ["Is setup.py deprecated?"] for more
  details).
  - If you wish, you can migrate your packaging setup to `pyproject.toml` entirely, but
    this isn't necessary

**For more advice, please [see the issue tracking this deprecation][deprecation] and
[this comment].** You can also read my [write-up on pip 24.2] which discusses the context
behind the deprecation.

## QoL improvements

### Recursive requirement files are now detected

Imagine you had a requirement file that recursively included itself:

```
-r requirements.txt
```

If you ran `pip install -r requirements.txt`, you'd get a nasty crash as pip kept
following the loop until it reached the recursion limit:

```console
ERROR: Exception:
Traceback (most recent call last):
[trimmed...]
  File "/home/ichard26/dev/oss/pip/src/pip/_internal/cli/req_command.py", line 255, in get_requirements
    for parsed_req in parse_requirements(
  File "/home/ichard26/dev/oss/pip/src/pip/_internal/req/req_file.py", line 151, in parse_requirements
    for parsed_line in parser.parse(filename, constraint):
  File "/home/ichard26/dev/oss/pip/src/pip/_internal/req/req_file.py", line 332, in parse
    yield from self._parse_and_recurse(filename, constraint)
  File "/home/ichard26/dev/oss/pip/src/pip/_internal/req/req_file.py", line 361, in _parse_and_recurse
    yield from self._parse_and_recurse(req_path, nested_constraint)
  File "/home/ichard26/dev/oss/pip/src/pip/_internal/req/req_file.py", line 361, in _parse_and_recurse
    yield from self._parse_and_recurse(req_path, nested_constraint)
  File "/home/ichard26/dev/oss/pip/src/pip/_internal/req/req_file.py", line 361, in _parse_and_recurse
    yield from self._parse_and_recurse(req_path, nested_constraint)
  [Previous line repeated 974 more times]
  File "/home/ichard26/dev/oss/pip/src/pip/_internal/req/req_file.py", line 337, in _parse_and_recurse
    for line in self._parse_file(filename, constraint):
[trimmed...]
RecursionError: maximum recursion depth exceeded
```

This example is admittedly a bit contrived, but it can happen if you're referencing
requirement files in a chain and forget what you've already included. Now, pip will detect
this loop and raise a proper error.

```console {.command }
pip install -r test.txt
ERROR: /home/ichard26/dev/oss/pip/test.txt recursively references itself in test.txt
```

This was contributed by Kuntal Majumder (@hellozee) in [PR #12877]. ✨

_(There was a [bug where the error would be raised too often][#13046], even when no actual
recursion was present. This was promptly addressed in pip 24.3.1.)_

### Installed packages with broken dependency metadata now raise a proper error

Since pip 24.1,
[non standard-compliant versions or dependency specifiers are unsupported][legacy-versions].
While pip will ignore packages with this legacy invalid metadata when possible, in other
cases, the offending package had to be uninstalled to allow pip to function.

Unfortunately, similar to the issue above, pip would just crash and leave the user a
horrible traceback in these cases.

```console { .command }
pip install celery
Requirement already satisfied: celery in ./venv/lib/python3.12/site-packages (4.4.7)
ERROR: Exception:
[trimmed...]
  File "/home/ichard26/dev/oss/pip/src/pip/_internal/resolution/resolvelib/candidates.py", line 401, in iter_dependencies
    for r in self.dist.iter_dependencies():
  File "/home/ichard26/dev/oss/pip/src/pip/_internal/metadata/importlib/_dists.py", line 215, in iter_dependencies
    req = get_requirement(req_string.strip())
          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/ichard26/dev/oss/pip/src/pip/_internal/utils/packaging.py", line 45, in get_requirement
    return Requirement(req_string)
           ^^^^^^^^^^^^^^^^^^^^^^^
  File "/home/ichard26/dev/oss/pip/src/pip/_vendor/packaging/requirements.py", line 38, in __init__
    raise InvalidRequirement(str(e)) from e
pip._vendor.packaging.requirements.InvalidRequirement: Expected matching RIGHT_PARENTHESIS for LEFT_PARENTHESIS, after version specifier
    pytz (>dev)
         ~^
```

To address this, a new diagnostic error has been added when encountering a broken
pre-installed package.

```console { .command }
pip install celery
Requirement already satisfied: celery in ./venv/lib/python3.12/site-packages (4.4.7)
error: invalid-installed-package

× Cannot process installed package celery 4.4.7 in '/home/ichard26/dev/oss/pip/venv/lib/python3.12/site-packages' because it has an invalid requirement:
│ Expected matching RIGHT_PARENTHESIS for LEFT_PARENTHESIS, after version specifier
│     pytz (>dev)
│               ^
╰─> Starting with pip 24.1, packages with invalid requirements can not be processed.

hint: To proceed this package must be uninstalled.
```

Admittedly, the error is a bit hard to read, but with time, pip's errors should improve.
In the interim, you can quickly tell what package is causing the problem (celery) and how
to fix it (uninstall it).

## macOS 10.12 is resupported

If you're part of the tiny group that still uses modern pip on macOS 10.12, you would've
learned first-hand that truststore did not support macOS 10.12, [breaking pip][macos-bug].
Good news though, pip 24.3 restores support by vendoring in an updated version of
`truststore` that uses older macOS APIs when needed.

This brings the pip project one step closer to being able to use system HTTPS certificates
when possible, all without blowing up in odd situations. There are still
[some known issues with private certificates][truststore bug], however.

## Invalid wheel filenames are deprecated

> **Note:** Unless you use ancient packages before the advent of modern packaging tooling
> AND are incredibly unlucky, this deprecation will NOT affect you. This is mostly for
> those interested in the packaging nitty gritty.

If you have any sort of familiarity with Python packaging, you'd have heard of wheels.
Wheels are a binary distribution format for Python packages. They are pre-built, which
makes installing them particularly easy: just unzip them and copy the contents to the
`site-packages` directory.[^well-actually]

Something you might not know is that
[their filenames follow a certain structure][wheel-filenames]. This enables installers to
extract essential metadata without inspecting the wheel's contents. As an example, let's
take a look at the newest wheel for pip.

```
pip-24.3.1-py3-none-any.whl
```

Here, we know three pieces of information immediately:

- `pip` is the package name
- `24.3.1` is the version
- `py3-none-any` is the compatibility tag.
  [I went over compatibility tags in my previous pip write-up][compat-tags], but
  essentially, `py3-none-any` is code for this package supports “any Python 3 version”,
  “any Python ABI”, and “any architecture.”

Unfortunately, some wheels out in the wild have improper names, such as:

- `translate_toolkit-1.9.0_pinterest3-py27-none-any.whl`
- `j2cli-0.3.0_1-py2-none-any.whl`
- `password_strength-0.0.1_1-py2-none-any.whl`

According to the [version specifier standard], underscores can be used to separate the
release segment from any pre/post/development release segments. However, they cannot be
used to separate an implicit post-release.

In other words, `1.2.0_post1` (explicit) is valid, but `1.2.0_1` (implicit) is
not.[^normalization]

For historical reasons, pip has accepted wheels with this error. This is inconsistent and
confusing, thus starting with pip 24.3, these wheels are no longer supported.

Frankly, virtually no packages today have this issue, so you can forget this deprecation
even exists[^not-a-real-problem], but it is a perfect opportunity to learn about the
complexity found in Python packaging. Many things may look easy in packaging, but often
they are not :)

## Other changes

- iOS wheels are provisionally supported: pip will now recognize and accept iOS tagged
  wheels, although whether pip properly supports them is something I don't know.
- Installing pip in editable mode in a virtual environment on Windows doesn't result in a
  [data race crash anymore]
- The `--target` option is ignored while pip prepares a build environment during
  installation, which often caused [confusing crashes] when it was configured globally in
  a pip configuration file or via the `PIP_TARGET` environment variable.

[^well-actually]: Of course, it's not this simple in reality, but there is no "build step" involved
    which makes them trivial compared to source distributions (typically .tar.gz).

[^normalization]: To declare a post-release without the `post` prefix, you must use the `-` separator,
    e.g. `1.2.0-1`. Naturally, this raises the question of what to do when this version is
    placed in the wheel filename as dashes are reserved as wheel filename separators. The
    answer is that the version should be normalized when placed in a wheel filename, and
    fortunately for us, `1.2.0-1` normalizes to `1.2.0.post1`.

[^not-a-real-problem]: [While reviewing the deprecation PR][#12918], I scraped the index pages for the top
    8000 PyPI packages and could only find 4 ancient wheels that have this quirk in their
    filename.

["how to modernize a setup.py based project?"]: https://packaging.python.org/en/latest/guides/modernize-setup-py-project/
["is setup.py deprecated?"]: https://packaging.python.org/en/latest/discussions/setup-py-deprecated/
[#12918]: https://github.com/pypa/pip/pull/12918
[#13046]: https://github.com/pypa/pip/issues/13046
[build system declaration]: https://packaging.python.org/en/latest/specifications/pyproject-toml/#declaring-build-system-dependencies-the-build-system-table
[compat-tags]: /blog/2024/08/whats-new-in-pip-24.2/#pip-check-just-got-a-bit-stricter
[confusing crashes]: https://github.com/pypa/pip/issues/8438
[data race crash anymore]: https://github.com/pypa/pip/issues/12666
[deprecation]: https://github.com/pypa/pip/issues/11457
[legacy-versions]: https://pradyunsg.me/blog/2024/05/13/pip-24-1-betas/
[macos-bug]: https://github.com/pypa/pip/issues/12901
[pep 660]: https://peps.python.org/pep-0660/
[pr #12877]: https://github.com/pypa/pip/pull/12877
[the pip team released pip 24.3]: https://discuss.python.org/t/announcement-pip-24-3-release/69350
[this comment]: https://github.com/pypa/pip/issues/11457#issuecomment-2439645125
[truststore bug]: https://github.com/pypa/pip/pull/12918
[version specifier standard]: https://packaging.python.org/en/latest/specifications/version-specifiers/#implicit-post-releases
[wheel-filenames]: https://packaging.python.org/en/latest/specifications/binary-distribution-format/#file-name-convention
[write-up on pip 24.2]: /blog/2024/08/whats-new-in-pip-24.2/
