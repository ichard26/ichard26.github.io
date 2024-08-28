---
title: What's new in pip 24.2 — or why legacy editable installs are deprecated
slug: whats-new-in-pip-24.2
description: &desc >-
  In version 24.2, pip learns to use system certificates by default, receives a
  handful of optimizations, and deprecates legacy (setup.py develop) editable installations.
summary: *desc
date: 2024-08-26
modified: 2024-08-28
tags: [pip, release, deprecation]
showToc: true
---

On July 28, 2024, the pip team released [pip 24.2]. As a recent addition to the pip core
team, I'll walk you through the noteworthy or interesting changes from pip 24.2. Let's go!

> #### PSA, PSA, PSA 📣
>
> If you're using setuptools, do not have a `pyproject.toml`, and use pip's `-e` flag, you
> may be impacted by the [deprecation of legacy editable installs][editable-deprecation].
>
> If you're an user of setuptools and came to this post after seeing the legacy editable
> install deprecation warning,
> [you should first read the deprecation issue for solutions, **not here**][editable-deprecation].
>
> You can come back and read this after for more historical context on the deprecation,
> however! It's a good read, I promise.

Here's a [link to the 24.2 changelog][changelog] if you'd like the full list of changes.

## Legacy editable installs are deprecated

> TL;DR, **don't panic**. The `-e` option is _not_ deprecated, but the way it works under
> the hood will change, potentially necessitating changes to how your project is packaged.

Over the last decade, there has been a major transition towards **standardized
mechanisms** for packaging Python projects:

- **[PEP 517]**: introduced an interface where frontends (e.g. the pip installer) can
  interact with a build backend (e.g. Poetry[^backends]), including asking it to build a
  wheel for later installation
- **[PEP 518]**: introduced `pyproject.toml` and defined a standard format for declaring
  build-time dependencies using this file
- **[PEP 621]**: defined a standard format for declaring project metadata in
  `pyproject.toml`
- **[PEP 660]**: augmented the PEP 517 interface to define a way to ask a build backend to
  prepare an editable install

These PEPs are what enable the use of alternative backends like [Poetry], [Hatch], and
[scikit-build-core] _without needing to add specific support for each backend_ in every
frontend (e.g. pip, tox, uv). Instead of running `setup.py bdist_wheel` to ask setuptools
to build a wheel for installation, pip can simply call the `build_wheel`
hook[^its-more-complicated] from PEP 517 which all backends are guaranteed to have.

As each standard matured, **pip has progressively deprecated and removed support for
legacy** (and often setuptools-specific) **mechanisms** to reduce technical debt and
ensure that pip can continue to evolve.

Editable installs—the thing you get from `pip install -e <path>`—used to be a setuptools
only feature, requiring pip to call the project's `setup.py` with the `develop`
sub-command. However, this stopped being the case under [PEP 660]. As long as your
project's backend supports PEP 660, `pip install -e` will continue to work. No setuptools
required. (Although modern setuptools works fine as well.)

### OK, so what now?

This release,
[pip has deprecated support for the `setup.py develop` fallback][editable-deprecation]
which is used when a project lacks support for modern editable installs. **It will be
removed in pip 25.0 (Q1 2025)** after which projects MUST support PEP 660 to perform an
editable install. Affected projects will see this deprecation warning:

```console {.command hl_lines=[5]}
pip install -e temp/pip-test-package
Obtaining file:///home/ichard26/dev/oss/pip/temp/pip-test-package
  Preparing metadata (setup.py) ... done
Installing collected packages: version_pkg
  DEPRECATION: Legacy editable install of version_pkg==0.1 from file:///home/ichard26/dev/oss/pip/temp/pip-test-package (setup.py develop) is deprecated. pip 25.0 will enforce this behaviour change. A possible replacement is to add a pyproject.toml or enable --use-pep517, and use setuptools >= 64. If the resulting installation is not behaving as expected, try using --config-settings editable_mode=compat. Please consult the setuptools documentation for more information. Discussion can be found at https://github.com/pypa/pip/issues/11457
  Running setup.py develop for version_pkg
Successfully installed version_pkg-0.1
```

In practice, this usually means one of two things:

- The `pyproject.toml` file doesn't exist at all, thus pip will opt-out of the PEP 517
  interface, running `setup.py` as required
- The declared setuptools version (in the `[build-system].requires` field) is too old and
  doesn't support PEP 660, i.e. anything older than [setuptools 64.0.0]

For affected projects, the best solution is likely to **use a modern version of setuptools
and declare this via `pyproject.toml`**. You can place your project metadata in this file
as well, but it's _not_ required to stop the legacy editable deprecation warnings.

```toml
# pyproject.toml
[build-system]
requires = ["setuptools >= 64"]
build-backend = "setuptools.build_meta"
```

> **Tip**: if you have other dependencies needed to run `setup.py` or otherwise build your
> package, you should add them to the `["setuptools >= 64"]` list. This field is
> equivalent to setuptools' `setup_requires` parameter.

Alternatively, projects can pass the `--enable-pep517` flag to force pip to use the PEP
517 interface, including the editable install extensions from PEP 660. If no backend is
declared, pip will assume that the project uses setuptools and ensure it's available in
the isolated build environment.[^isolation] As long as setuptools 64+ still supports your
Python version, a modern editable install will be performed. Once the legacy mechanism is
removed, `--use-pep517` will have no effect and will essentially be enabled by default.

Finally, if you don't like either of those solutions, you can always throw away setuptools
and switch to a different backend entirely. I've mentioned a few earlier, but there are
even more options to choose from if you're willing to do some research and play around.
I'd start with with the
[Python Packaging User Guide's list of the commonly used backends][backends] if you want
to replace setuptools.

~~If you stick with setuptools, one potential snag is that setuptools has introduced a new
kind of editable installs while rolling out PEP 660. They're called
[strict editable installs], which behave closer to a normal installation, but are
implemented in an entirely different way from `setup.py develop`, potentially breaking
certain workflows. If the [legacy behaviour is desired][legacy-editable], one must pass
`--config-settings editable_mode=compat`.[^strict-editables]~~ (**Update**: a setuptools
maintainer reached out to me and informed that strict editable installs are not enabled by
default. The situation regarding potential breakage is a bit more nuanced. I'll fix this
post later when I get the chance.)

> **Warning**: Static analysis tools including mypy, pyright, and pylint may not function
> properly when setuptools uses an import hook to implement an editable install.
> [This is a known issue][strict-editable-analysis]. The recommended workaround is to pass
> `--config-settings editable_mode=compat`.

For more details using setuptools with `pyproject.toml`, you should read their
documentation:

- ⭐ [User guide to using `pyproject.toml`][setuptools-pyproject.toml] ⭐
- ⭐ [setuptools' editable install docs] ⭐
- [setuptools' build system docs]

If you'd like to learn more, especially about the history of why and how these standards
came to exist, **I strongly recommend
[reading Paul Ganssle's excellent "Why you shouldn't invoke setup.py directly" article][paul ganssle article]**.
It's _long_, but packs way more information than I could ever _hope_ to cover.

## System HTTPS certificates by default(\*)

If you've ever used pip in a corporate environment, there is a good chance that you've
encountered a network or SSL verification error because the remote index server (PyPI or a
private index) returned a HTTPS certificate that couldn't be verified.

```console {.command hl_lines=[8]}
pip install toto
Looking in indexes: https://repos.company/api/pypi/python_pypi/simple
WARNING: Retrying (Retry(total=4, connect=None, read=None, redirect=None, status=None)) after connection broken by 'SSLError(SSLError(0, 'unknown error (_ssl.c:3630)'),)': /api/pypi/python_pypi/simple/toto/
WARNING: Retrying (Retry(total=3, connect=None, read=None, redirect=None, status=None)) after connection broken by 'SSLError(SSLError(0, 'unknown error (_ssl.c:3630)'),)': /api/pypi/python_pypi/simple/toto/
WARNING: Retrying (Retry(total=2, connect=None, read=None, redirect=None, status=None)) after connection broken by 'SSLError(SSLError(0, 'unknown error (_ssl.c:3630)'),)': /api/pypi/python_pypi/simple/toto/
WARNING: Retrying (Retry(total=1, connect=None, read=None, redirect=None, status=None)) after connection broken by 'SSLError(SSLError(0, 'unknown error (_ssl.c:3630)'),)': /api/pypi/python_pypi/simple/toto/
WARNING: Retrying (Retry(total=0, connect=None, read=None, redirect=None, status=None)) after connection broken by 'SSLError(SSLError(0, 'unknown error (_ssl.c:3630)'),)': /api/pypi/python_pypi/simple/toto/
Could not fetch URL https://repos.company/api/pypi/python_pypi/simple/toto/: There was a problem confirming the ssl certificate: HTTPSConnectionPool(host= 'repos.company', port=443): Max retries exceeded with url: /api/pypi/python_pypi/simple/toto/ (Caused by SSLError(SSLError(0, 'unknown error (_ssl.c:3630)'),)) - skipping
ERROR: Could not find a version that satisfies the requirement toto (from versions: none)
ERROR: No matching distribution found for toto
```

The problem is that while your system may be configured with the required certificate
authorities, _pip had no ability to use those system CAs_. This resulted in a confusing
user experience, because while your browser was likely able to access the index server,
pip [blew up][issue-12560] with a non-obvious [error][issue-10777].

To be able to verify HTTPS certificates, since the early days, pip has shipped with the
[certifi] CA bundle, which is simply a Python repackaging of the Mozilla CA bundle. To get
pip to verify HTTPS certificates not trusted by certifi, you had to pass your own bundle
via the `--cert` option which is clunky.

In pip 22.2, pip gained the ability to use [truststore] for HTTPS certificate validation,
enabling pip to use the system certificate store instead of solely relying on certifi. The
integration was experimental; users had to have truststore already installed and opt-in
via `--use-feature=truststore`, but it was a step forward.[^1] The plan was to enable
truststore by default once the feature matured.

Two years later, in this pip release, **truststore is now enabled by default**.[^tandem]
Thus, assuming your system is configured properly with the required CAs, pip should simply
"just work" even in many corporate environments. 🎉

There is one major problem though—you saw that asterisk, right? **Truststore only works on
Python 3.10 or higher**. pip will continue to exclusively use its certifi bundle on Python
3.9 or 3.8. Additionally,
[any Python implementation that does not implement the `ssl` module APIs needed by truststore][graalpy]
will not be able to use the system trust store either. In these situations, you will need
to continue using `--cert`.

Major thanks goes to [@sethmlarson] for co-authoring truststore and writing patches
improving and fixing pip's truststore integration!

## Performance optimizations, duh!

In this release, several optimizations of the environment inspection, file download,
dependency resolution, and installation logic landed. While individually they are minor
optimizations, together they provide a noticeable performance uplift in a variety of
workloads!

### Faster discovery of installed packages

First of all, on Python 3.11 or higher, pip is significantly faster to discover installed
packages (especially if there are a lot of 'em). This not only makes `pip list` faster,
but also makes pip generally faster as it has to figure out what packages are already
installed [quite frequently throughout its operation][importlib-opt-makes-install-faster].

These are the before and after runtimes of `pip list` in my pip development environment
which has 75 packages installed:

```ini {hl_lines=[2]}
# before: pip list (24.1)
real	0m0.227s
user	0m0.192s
sys 	0m0.035s
```

```ini {hl_lines=[2]}
# after: pip list (24.2)
real	0m0.207s
user	0m0.170s
sys 	0m0.036s
```

On slower systems, the performance improvement will be greater.[^pip-startup-times] This
uplift was achieved by optimizing pip's `importlib` based metadata backend to read
package[^package-vs-distribution] names and versions from the installed metadata directory
names.[^installed-metadata] Previously, pip would read the name and version from the
[`METADATA` file], which is slow as it involves an extra file read and invoking an email
header parser.

### Other optimizations

Faster downloads _[PR #12810]_ _@morotti_
: Source distributions and wheels are downloaded in 256 KiB chunks instead of 10 KiB
  chunks, which significantly reduces overhead and enables faster download speeds. A
  sizable _chunk_ (🥁) of the performance improvement comes from reducing the number of
  updates of the download progress bar.

Faster dependency resolution _[PR #12663]_ _@notatallshaw_
: ... of highly complex or pathological dependency trees. As pip's dependency resolver
  explores a dependency tree, it has to parse requirements (e.g. `pip==24.2`) into a
  format it understands. In complex trees, the same requirements are often encountered
  many, _many_ times. There was _some_ caching in-place to minimize redundant work, but
  now requirements are consistently cached.

Faster installation _[PR #12803]_ _@morotti_
: Wheels, [which are really zip archives], are extracted using a larger block size (either
  1MiB or the file size if it's smaller than a MiB). Also, the zip decompressor is no
  longer invoked while copying empty files, eliminating pointless CPU cycles for files
  like `__init__.py`.

Faster reinstalls _[PR #12755]_ _yours truly!_
: The full list of platform compatibility tags is now only generated once and reused
  across all look-ups of the wheel cache, speeding up cached reinstalls.

## `pip check` just got a bit stricter

The `check` pip command now verifies whether installed packages are declared to support
the current platform, issuing a warning if a package is incompatible.

```console {.command}
pip check
catboost 1.1.1 is not supported on this platform
ninja 1.11.1.1 is not supported on this platform
xgboost 1.6.1 is not supported on this platform
frozendict 2.3.8 is not supported on this platform
```

Under the hood, for every package, pip reads the `WHEEL` metadata file that's installed
alongside the package, and checks whether at least one of the compatibility tags is
supported on the platform.

For example, this is the `WHEEL` file from pip 24.2's wheel:

```yaml
# pip-24.2.dist-info/WHEEL
Wheel-Version: 1.0
Generator: setuptools (71.1.0)
Root-Is-Purelib: true
Tag: py3-none-any
```

`py3-none-any` is a platform compatibility tag. What's a compatibility tag?
[They're essentially markers for what systems a package supports.][compat-tags] You can
think of them like languages. They include information on the architecture, Python ABI,
and Python version. Every system has a set of compatibility tags it supports. If a package
and the system it's being installed to share a common tag—or with the analogy, _speak a
shared language_—the package is considered compatible.

In this case, `py3-none-any` is code for "any Python 3 version", "any Python ABI", and
"any architecture." This is supported by any modern Python installation, thus this wheel
would be almost always eligible for installation (although it may be rejected for other
reasons, like `requires-python` restrictions).

Now, `pip install` already uses compatibility tags to ensure it only installs compatible
packages. This extra check in `pip check` is designed to surface packages that were
**compatible at the time of installation, but are no longer supported due to a system
upgrade**. This is possible, especially as
[Apple transitions from the `x86-64` architecture (Intel) to `arm64` (Apple Silicon)][apple].

(Un)fortunately, this has had the side-effect of
[revealing numerous packages with inaccurate or outright malformed metadata, leading to false positives][false-positives].
This is annoying, but regarded as an intentional side-effect as the pip project has been
moving towards enforcing standards compliance so pip follows the standards instead of de
facto writing them.[^pip-isn't-a-standard]

This was [kindly contributed by @q0w].

## Oh yeah, `--require-virtualenv` is a thing

You can configure pip to only function when running in an activated virtual environment
via the `--require-virtualenv` flag. You can set this via a system-side or user-side
configuration file; and it's great for ensuring you don't scribble all over your system or
user Python environment.

```console {.command}
pip install package-that-shouldnt-be-installed-system-wide
ERROR: Could not find an activated virtualenv (required).
```

Anyway, it's _supposed_ to only apply to commands **that modify the environment**, like
`pip install` or `pip uninstall`. Read-only commands, such as `pip list`, are unaffected
by the flag and will function even if a virtual environment is not activated.

This release extends this QoL feature to `pip check` and `pip index`. You can now discover
unsupported packages whenever you want. [Thanks @branchvincent]!

## Summary

pip 24.2 delivers considerable performance and QoL improvements. It also continues the pip
project's goal of following and enforcing the use of and compliance with the
interoperatability standards that have shaped modern Python packaging.

I realize that this post is long, but this release contained some cool changes and I
believed they deserved more attention than a brief entry in the changelog. I make no
promises that I'll continue this series for future pip releases, but this sure was fun to
write!

Finally, I'd like to thank Bartosz Sokorski, Bradley Reynolds, Hugo van Kemenade, and Paul
Moore for reviewing this post in draft form and suggesting improvements. Any mistakes are
of course my own.

[^backends]: Poetry is the all encompassing user-facing tool for packaging and dependency
    management, while `poetry-core` is its backend. The same nitpick exists for all other
    "backends" I mention.

[^its-more-complicated]: In reality, pip calls other hooks as well to query additional build dependencies and
    generate metadata, but this isn't the post where I explain pip's entire PEP 517
    implementation.

[^isolation]: `--no-build-isolation` may be needed if the project has build-time requirements beyond
    setuptools and wheel. By passing this flag, you are responsible for making sure your
    environment already has the required dependencies to build your package.

[^strict-editables]: As of writing, the TL;DR is that strict editables are implemented as a tree of file
    links to the original source files in an auxiliary directory which is then added to
    `sys.path`, while legacy "compat mode" editables are implemented by placing the entire
    root of your package on `sys.path`. The new method is designed to ensure only files
    present at installation are exposed, mimicking a normal installation.

[^1]: This [was recognized as a problem in 2014][#1680], and pip gained the ability to
    [use system certificates on Unix in 2015][pull-1866], but this change was later
    reverted because it turned out
    [none of the major distributions had a functional OpenSSL configuration at the time][system-ca-revert].

[^tandem]: pip continues to use truststore in tandem with certifi. This is necessary for
    platforms that do not support truststore, but also provides a fallback if the system
    trust store is broken or otherwise inaccessible for some reason. There are currently
    no plans for pip to switch to exclusively use truststore.

[^pip-startup-times]: Also, it's worth mentioning that 3/4 of the 200ms is taken by Python's own startup
    overhead and importing everything `pip list` needs.

[^package-vs-distribution]: OK, so technically I should be saying "distribution" to avoid ambiguity as the term
    "package" means something different under the Python import system, but distribution
    sounds weird which is why I'm saying "package" still. I'm horribly inconsistent about
    this though.

[^installed-metadata]: If you dig into your `site-packages` directory, you'll notice that there is a bunch of
    `<name>-<version>.dist-info` directories. These contain package metadata, and there is
    one for each package installed. They are included in wheels and are copied during
    installation.

[^pip-isn't-a-standard]: Historically, as pip was the only installer in town, various bits of pip's design or
    behaviour has been become de facto standards. Even if there is a standard, "pip
    supports X" or "pip does X" can be regularly seen in the issue trackers of other
    packaging tools. While this is expected and unavoidable, it's unideal. We don't want
    to be in the business of telling what everyone else should be doing, especially as pip
    is old and has quirks and design flaws that frankly shouldn't exist, let alone be
    copied by other tools. By gradually enforcing standards compliance, the Python
    packaging ecosystem becomes more and more defined by _common specifications_, allowing
    other packaging tools to function without reimplementing a bunch of pip bugs or
    whatever.

[#1680]: https://github.com/pypa/pip/issues/1680
[@sethmlarson]: https://sethmlarson.dev/
[apple]: https://github.com/pypa/pip/issues/11054
[backends]: https://packaging.python.org/en/latest/guides/tool-recommendations/#build-backends
[certifi]: https://pypi.org/project/certifi/
[changelog]: https://pip.pypa.io/en/stable/news/#v24-2
[compat-tags]: https://packaging.python.org/en/latest/specifications/platform-compatibility-tags/
[editable-deprecation]: https://github.com/pypa/pip/issues/11457
[false-positives]: https://github.com/pypa/pip/issues/12884
[graalpy]: https://github.com/pypa/pip/issues/12892
[hatch]: https://hatch.pypa.io/latest/
[importlib-opt-makes-install-faster]: https://github.com/pypa/pip/pull/12656#issuecomment-2097040577
[issue-10777]: https://github.com/pypa/pip/issues/10777
[issue-12560]: https://github.com/pypa/pip/issues/12560
[kindly contributed by @q0w]: https://github.com/pypa/pip/pull/11088
[legacy-editable]: https://setuptools.pypa.io/en/latest/userguide/development_mode.html#legacy-behavior
[paul ganssle article]: https://blog.ganssle.io/articles/2021/10/setup-py-deprecated.html
[pep 517]: https://peps.python.org/pep-0517/
[pep 518]: https://peps.python.org/pep-0518/
[pep 621]: https://peps.python.org/pep-0621/
[pep 660]: https://peps.python.org/pep-0660/
[pip 24.2]: https://discuss.python.org/t/announcement-pip-24-2-release/59402
[poetry]: https://python-poetry.org/
[pr #12663]: https://github.com/pypa/pip/pull/12663
[pr #12755]: https://github.com/pypa/pip/pull/12755
[pr #12803]: https://github.com/pypa/pip/pull/12803
[pr #12810]: https://github.com/pypa/pip/pull/12810
[pull-1866]: https://github.com/pypa/pip/pull/1866
[scikit-build-core]: https://scikit-build-core.readthedocs.io/
[setuptools 64.0.0]: https://setuptools.pypa.io/en/latest/history.html#v64-0-0
[setuptools' build system docs]: https://setuptools.pypa.io/en/latest/build_meta.html
[setuptools' editable install docs]: https://setuptools.pypa.io/en/latest/userguide/development_mode.html
[setuptools-pyproject.toml]: https://setuptools.pypa.io/en/latest/userguide/pyproject_config.html
[strict editable installs]: https://setuptools.pypa.io/en/latest/userguide/development_mode.html#strict-editable-installs
[strict-editable-analysis]: https://github.com/pypa/setuptools/issues/3518
[system-ca-revert]: https://github.com/pypa/pip/commit/c77d4ab55ed412a3b72d0b73f504e8ddec918683
[thanks @branchvincent]: https://github.com/pypa/pip/pull/12842
[truststore]: https://pypi.org/project/truststore/
[which are really zip archives]: https://packaging.python.org/en/latest/specifications/binary-distribution-format/
[`metadata` file]: https://packaging.python.org/en/latest/specifications/recording-installed-packages/#the-metadata-file
