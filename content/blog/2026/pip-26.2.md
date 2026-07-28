---
title: What's new in pip 26.2 - only-deps and venv isolation!
slug: whats-new-in-pip-26.2
description: &desc >
  pip 26.2 adds support for installing dependencies exclusively (--only-deps), hashed
  and non-hashed requirements together (--no-require-hashes), experimental venv-based build
  isolation, faster repeated dependency resolves, and Python 3.15. Also included are a
  smattering of bugfixes.
summary: *desc
date: 2026-07-29
tags: [pip, release]
showToc: true
---

On July 29, 2026, I[^1] released pip 26.2.

Compared to a typical pip release, this release is _huge_ with many changes.
[This was largely made possible by the paid time I've had
to work on pip](/blog/2026/06/pip-contract-development/) and agentic LLM assistance. My contract
expires in late August, however, so expect future releases to be less blockbuster.

As always, please [consult the changelog][changelog] to learn about all of the changes
contained in this release.

## New features ✨

### Support for Python 3.15

While pip was largely already compatible with the Python 3.15 development
releases, pip 26.2 ships with a fix to build isolation, which was completely ineffective
on 3.15. Also addressed are various CPython deprecation warnings.

### Selecting only dependencies

You can now install only the dependencies of the requirements you provide with the
new `--only-deps` flag. This is a global flag like its `--no-deps` counterpart.

If you're confused, let's go through an example where we're working in a local Python project
with this `pyproject.toml` file.

```toml { filename = "pyproject.toml" }
[project]
name = "myawesomeproject"
version = "1.0"
dependencies = ["click", "rich"]

[project.optional-dependencies]
speed = ["orjson"]
```

`pip install . --only-deps` will only install `click`, `rich`, and their dependencies, but _not_
`myawesomeproject` itself. If you want the `speed` extra, too, `pip install .[speed] --only-deps`
will do the trick.

> [!tip]
> `pip install . --only-deps` is equivalent to `uv pip install -r pyproject.toml`.

`--only-deps` is primarily expected to be used for **development workflows where it's desirable
to install the project dependencies _without_ installing the project itself**. For example,
container build caching is much more efficient when dependencies are installed separately
and before installing the project itself.

`--only-deps` may be used with `pip download` and `pip wheel`. It also works with named, VCS,
and Direct URL requirements, though those are not the main use case.

> [!warning]
> `--only-deps` may NOT be combined with `--no-deps`,  `-r`, `--group`, and
> `--requirements-from-script`. [The rationale why can be found in the pip user
> guide][only deps restriction], but the TL;DR is that it's not immediately clear
> what counts as an user-supplied requirement when these options are used.
>
> If you do have a compelling use-case, let us know and we'll consider it.
>
> Finally, `--only-deps` cannot be used with the legacy resolver.

It's worth noting that for local directories and source distributions, pip will still call the
build backend to query dependency metadata. It is possible to optimize pip's dependency resolver
to read dependency metadata from `pyproject.toml` directly where possible[^direct pyproject read],
but this will be considered (and implemented) in the future.

Thank you to Sebastian Höffner for experimenting with and contributing the `--only-deps`
feature, [arguably our most upvoted feature request][#11440].

### `--no-require-hashes` for mixed requirement scenarios

For security and largely historical reasons, pip will automatically enable
`--require-hashes` if hashes are provided for any requirement. This has the effect
that using hash-locked requirements is an all or nothing proposition.

This means that a requirements file with hashes cannot be used with a local directory
or VCS requirement (which don't have a single file that can be hashed).

```console
$ pip install . -r deploy-requirements.txt
ERROR: Can't verify hashes for these file:// requirements because they point to directories:
    file:///home/ichard26/dev/oss/pip
```

```console
$ pip install -e . -r deploy-requirements.txt
$ pip install -e "project @ git+https://corp/project.git" -r deploy-requirements.txt
ERROR: The editable requirement project from <location> cannot be installed when requiring hashes, because there is no single file to hash.
```

To support these use cases, `--no-require-hashes` can be used to stop pip from automatically
enforcing that all requirements have hashes. For requirements that do have hashes, they will
continue to be enforced, as expected.

Thank you to Stéphane Bidoul for adding this flag.

### Experimental: `venv` based build isolation

To isolate build subprocesses from the environment pip is invoked in, pip uses a
custom `sitecustomize.py` file and `PYTHONPATH` environment variable tricks. This has worked
surprisingly well, but there are many rough edges, especially since pip is unique here as `build`
and `uv` use standard virtual environments.

Taking a few examples from the pip issue tracker:

- [Isolated build env breaks console scripts installed in other venv (e.g., `cmake`, `ninja`)][#13222]
- [PEP 517 build adds include directory from the original not-isolated environment][#12419]
- [`sysconfig.get_path` points to venv path when in build-isolation][#12976]
- And the 3.15 leaky isolation bug mentioned earlier 🙃

The good news is that pip 26.2 ships with an experimental feature to use `venv` to create
virtual environments for build isolation, available via `--use-feature=venv-isolation`.[^venv pain]

`venv-isolation` should work on all platforms pip officially supports.[^venv compat] If you've
been affected by a build isolation bug, **please try the feature and let us know if you run into
any issues**. I'd like to make it the default in a future release once it stabilizes out in the wild.

> [!warning]
> Combining `--use-feature=venv-isolation` with `--use-feature=inprocess-build-deps` is sadly
> not fully supported yet. While it will work in the vast majority of scenarios, there are
> edge cases around dependency overriding that will need to be fixed separately.

### Faster repeated resolves by caching index simple responses

pip previously would always [revalidate] all simple responses—the HTML pages that list all
the distributions available for a specific project (e.g., [pip](https://pypi.org/simple/pip))
—regardless of the `Cache-Control` header sent back by the remote index.

This choice was made so that newly published releases to PyPI would be immediately visible
to pip. This is important for CI/CD workflows that publish and install the new release in
the same job, usually for testing purposes.

However, this had the major downside of forcing pip to make an HTTP request for every
package it encounters, incurring the round-trip penalty each time. This significantly
slows down repeated pip invocations (e.g., tox/nox workflows), especially if you're on
a high-latency connection.

Starting in 26.2, pip will cache simple responses as per their `Cache-Control` header.
For PyPI users, this means newly published packages won't be visible until ~10 minutes
later.

> [!caution]
> If you do need **newly published packages to be immediately visible**, we've added an
> `--refresh-package` option to opt into the old revalidation behaviour for packages.

As an example on how much faster repeated pip runs can be, here's a pip development install
before and after:

```console
$ time pip install -e . --group test --refresh-package=:all:
[...]
Executed in    5.23 secs    fish           external
   usr time    3.08 secs    0.80 millis    3.08 secs
   sys time    0.34 secs    1.89 millis    0.34 secs
```

And with the simple response cache still warm (in a new virtual environment):

```console
$ time pip install -e . --group test
[...]
Executed in    2.89 secs    fish           external
   usr time    2.55 secs    0.35 millis    2.55 secs
   sys time    0.29 secs    1.67 millis    0.28 secs
```

The majority of the remaining time is spent installing the collected packages.

Thank you to Sepehr Rasouli for working on this performance improvement.

### Bypass `HTTP(S)_PROXY`, `ALL_PROXY`, and `NO_PROXY`

Via `requests`, pip will read proxy configuration from the `HTTP_PROXY`, `HTTPS_PROXY`,
`ALL_PROXY`, and `NO_PROXY` environment variables by default. This is useful, [but can
be undesirable at times][#5378]. If you need pip to ignore these envvars, you may use
the `--no-proxy-env` flag or set `--proxy=""` (which we claimed worked, but never did).

Thank you to Luis Carlos for contributing this new feature.

### Build constraints are now separate from regular constraints

Any constraints set through the `PIP_CONSTRAINT` environment variable
will no longer affect build dependencies. If you do need to constrain build
dependencies, use `--build-constraint` or `PIP_BUILD_CONSTRAINT` instead.

Thank you to Damian Shaw for contributing this feature.

## Notable bugfixes

### Don't cache local directory requirements with a dash

Local directory requirements are not supposed to be cached, so `pip install .`
and similar commands always reinstall the local project. Except, pip had
two related bugs where:

- If a local project's directory name contained a dash, like `my-library`,
  then pip would cache the built wheel
- If there is a matching cached wheel for a local project, pip would use it

Shivam Shrey squashed both bugs in pip 26.2.

### No more progress bar spam in CI

Pip uses rich to render its progress bars and (some) status spinners. Normally
these aren't animated in CI, but increasingly CI workflows set `FORCE_COLOR` to
ensure colorized output in CI logs. This is reasonable but causes rich to
treat the output device as an interactive terminal that can refresh, leading to
the install progress bar to span multiple lines:

```
Installing collected packages: distlib, platformdirs, packaging, humanize, filelock, colorlog, attrs, argcomplete, python-discovery, dependency-groups, virtualenv, nox
25l
   ━━━━━━━━━━╺━━━━━━━━━━━━━━━━━━━━━━━━━━━━━  3/12 [humanize]
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╺━━━━━━━━━  9/12 [dependency-groups]
   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 12/12 [nox]
25h
```

To address this, if pip detects that it's running in CI, rich's interactive mode will
be disabled.

### Prevent download location escape via URL quoting (CVE-2026-13346)

Pip uses the URL filename when determining the storage path for a file download. Unfortunately,
pip would unquote %-escapes twice, allowing the download to be saved to an arbitrary location
by injecting a slash. (`%252F` decodes to `%2F`, which then decodes to `/`.)

We've since removed the extraneous unquoting and adjusted the type annotations of the
surrounding code to help prevent similar issues from reappearing.

Note that this requires a **malicious index** to serve a malicious distribution download URL,
so while we did treat this as a security issue, **we consider the overall impact to be low**.
Thanks Damian for fixing this!

## Other noteworthy changes

Pulling straight from the pip 26.2 changelog:

- Add support for `pylock.toml` `upload-time` field, so `--uploaded-prior-to` works with
  `-r pylock.toml`
- Honor `--only-final` when sourcing requirements with `-r pylock.toml`
- Present more informative diagnostic errors on uncaught network errors
- Ensure truststore feature remains active while initially connecting to a HTTPS proxy
- Fix automatic download resumption to handle new errors raised by `urllib3` 2.x
- Various bugfixes to `pip list`'s interaction with `--exclude`, `--uploaded-prior-to`,
  `--no-binary`, `--only-binary`, `--prefer-binary`
- Add support for pulling username from keyring subprocess provider
- Recover credentials embedded in a redirect `Location` URL when handling a 401 response,
  even under `--no-input`

## Current deprecations

There are two main deprecations.

### The removal of the legacy dependency resolver (`--use-deprecated=legacy-resolver`)

We're still distant from being ready to remove the legacy resolver, but [we're preparing
and like to remove it sometime in 2027][legacy resolver sunset].

Specific guidance for those still relying on the legacy resolver will be communicated closer
to the removal date (which is yet to be scheduled). In the meanwhile, consider this a nudge
to try the resolvelib resolver, especially if you're using the legacy resolver purely for
performance reasons. The resolvelib resolver is significantly smarter and faster since its
initial release in 2020. Your complicated dependency tree will likely resolve without issue
today.

### The removal of the `pkg_resources` installation metadata backend

If you don't know what `pkg_resources` is, then you don't need to worry about this deprecation.

If you do know, then just know that [pip will soon stop using it entirely][pkg_resources] for querying
and processing distribution metadata, including identifying installed packages. On Python 3.11
and higher, pip already migrated to using [`importlib.metadata`], but
we've kept various opt-outs for users that can't use `importlib.metadata` for whatever reason.

If you're one of the few individuals setting `_PIP_USE_IMPORTLIB_METADATA=false` due to performance
issues with the `importlib.metadata` backend, please know that those have long since been addressed.

[^1]: I was the release manager this time around :\)

[^direct pyproject read]: Fun fact, the `project.dependencies` field can be declared dynamic in
  `pyproject.toml` and until metadata version 2.2, there was no way to reliably read the dependencies
  of a source distribution statically.

[^venv pain]: I wrote this feature, and let me tell you, _oh boy_, was it a pain! There was
  edge case after edge case and a whole lot of "ugh, why is X so much harder in older versions of Python".
  See <https://github.com/pypa/pip/pull/14070> for the gory details.

[^venv compat]: Sadly, some distributions of Python don't ship with `venv` out of the box or
  do not support `venv` at all, including Android, iOS, and probably a handful of (niche?)
  Linux distributions. Please note that Debian and its derivatives do ship a `venv` that pip
  can use (their version doesn't have `ensurepip`, but pip doesn't need that).

[changelog]: https://pip.pypa.io/en/latest/news/
[pkg_resources]: https://github.com/pypa/pip/issues/13317#issue-2974152491
[legacy resolver sunset]: https://github.com/pypa/pip/issues/10946
[only deps restriction]: https://pip.pypa.io/en/latest/user_guide/#installing-only-dependencies
[revalidate]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Conditional_requests
[`importlib.metadata`]: https://docs.python.org/3/library/importlib.metadata.html

[#12419]: https://github.com/pypa/pip/issues/12419
[#12976]: https://github.com/pypa/pip/issues/12976
[#13222]: https://github.com/pypa/pip/issues/13222
[#5378]: https://github.com/pypa/pip/issues/5378
[#11440]: https://github.com/pypa/pip/issues/11440
