---
title: What's new in pip 24.2
description: &desc >-
  In version 24.2, pip learns to use system certificates, receives a
  handful of optimizations, and deprecates legacy editable installations.
summary: *desc
date: 2024-08-24
tags: [pip, release]
showToc: true
draft: true
---

On July 28, 2024, the pip team released [pip 24.2]. As a recent addition to the pip core
team, I'll walk you through the noteworthy or interesting changes from pip 24.2. Let's go!

> #### PSA, PSA, PSA
>
> If you're using setuptools, do not have a `pyproject.toml`, and use pip's `-e` flag, you
> may be impacted by the deprecation of legacy editable installs.
>
> \<insert-call-to-action>

Here's a [link to the 24.2 changelog][changelog] if you'd like to have it while following
along.

## System HTTPS certificates by default(\*)

If you've ever used pip in a corporate environment, there is a good chance that you've
encountered a network or SSL verification error because the remote index server (PyPI or a
private index) returned a HTTPS certificate that couldn't be verified.

```console {.command hl_lines=[8]}
python -m pip install toto
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
via the `--cert` option, which is clunky.

In pip 22.2, pip gained the ability to use [truststore] for HTTPS certificate validation,
enabling pip to use the system certificate store instead of solely relying on certifi. The
integration was experimental; users had to have truststore already installed and to opt-in
via `--use-feature=truststore`, but it was a step forward.[^1] The plan was to enable
truststore by default once the feature matured.

Two years later, in this pip release, **truststore is now enabled by default**.[^tandem]
Thus, assuming your system is configured properly with the required CAs, pip should simply
"just work" even in many corporate environments. 🎉

There is one major problem though—you saw that asterisk, right? **Truststore only works on
Python 3.10 or higher**, so on Python 3.9 or 3.8, pip will continue to exclusively use its
certifi bundle. Additionally,
[any Python implementation that does not implement the `ssl` module APIs needed by truststore][graalpy]
will not be able to use the system trust store either. In these situations, you will need
to continue using `--cert`.

Major thanks goes to [@sethmlarson] for co-authoring truststore and writing patches
improving and fixing pip's truststore integration!

## Legacy editable installs are deprecated

Over the last decade, there has been a major transition towards **standardized
mechanisms** for packaging Python projects:

- **[PEP 517]**: introduced an interface where frontends (e.g. the pip installer) can
  interact with a build backend (e.g. poetry), including asking it to build a wheel for
  later installation
- **[PEP 518]**: introduced `pyproject.toml` and defined a standard format for declaring
  build-time dependencies using this file
- **[PEP 621]**: defined a standard format for declaring project metadata in
  `pyproject.toml`
- **[PEP 660]**: augmented the PEP 517 interface to define a way to ask a build backend to
  prepare an editable install

These PEPs are what enable the use of alternative backends like [poetry], [hatch], and
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

This release, pip has deprecated support for the `setup.py develop` fallback which is used
when a project lacks support for modern editable installs.

```console {hl_lines=[5]}
$ pip install -e temp/pip-test-package
Obtaining file:///home/ichard26/dev/oss/pip/temp/pip-test-package
  Preparing metadata (setup.py) ... done
Installing collected packages: version_pkg
  DEPRECATION: Legacy editable install of version_pkg==0.1 from file:///home/ichard26/dev/oss/pip/temp/pip-test-package (setup.py develop) is deprecated. pip 25.0 will enforce this behaviour change. A possible replacement is to add a pyproject.toml or enable --use-pep517, and use setuptools >= 64. If the resulting installation is not behaving as expected, try using --config-settings editable_mode=compat. Please consult the setuptools documentation for more information. Discussion can be found at https://github.com/pypa/pip/issues/11457
  Running setup.py develop for version_pkg
Successfully installed version_pkg-0.1
```

In practice, this usually means one of two things:

- The `pyproject.toml` doesn't exist at all, thus pip will opt-out of the PEP 517
  interface, running `setup.py` as required
- The declared setuptools version (in the `[build-system].requires` field) is too old and
  doesn't support PEP 660, i.e. anything older than [setuptools 64.0.0]

For affected projects, the best solution is to **use a modern version of setuptools and
declare this via `pyproject.toml`**.

```toml
# pyproject.toml
[build-system]
requires = ["setuptools >= 64"]
build-backend = "setuptools.build_meta"
```

Alternatively, projects can pass the `--enable-pep517` flag to force pip to use the PEP
517 interface, including the editable install extensions from PEP 660. If no backend is
declared, pip will assume that the project uses setuptools and ensure it's available in
the isolated build environment.

One potential snag is that setuptools has introduced a new kind of editable installs while
rolling out PEP 660. They're called [strict editable installs], which behave closer to a
normal installation, but are implemented in an entirely different way from
`setup.py develop`, potentially breaking certain workflows. If the legacy behaviour is
desired, one must pass `--config-settings editable_mode=compat`.[^strict-editables]

If you'd like to learn more, especially about the history of why and how these standards
came to exist, **I strongly recommend
[reading Paul Ganssle's excellent "Why you shouldn't invoke setup.py directly" article][paul ganssle article]**.
It's _long_, but packs way more information than I could ever _hope_ to cover. In
addition, [setuptools' build system docs] and [setuptools' editable install docs] are good
to read too.

[^1]: This [was recognized as a problem in 2014][#1680], and pip gained the ability to
    [use system certificates on Unix in 2015][pull-1866], but this change was later
    reverted because it turned out
    [none of the major distributions had a functional OpenSSL configuration at the time][system-ca-revert].

[^tandem]: pip continues to use truststore in tandem with certifi. This is necessary for
    platforms that do not support truststore, but also provides a fallback if the system
    trust store is broken or otherwise inaccessible for some reason. There are currently
    no plans for pip to switch to exclusively use truststore.

[^its-more-complicated]: In reality, pip calls other hooks as well to query additional build dependencies and
    generate metadata, but this isn't the post where I explain pip's entire PEP 517
    implementation.

[^strict-editables]: As of writing, the TL;DR is that strict editables are implemented as a tree of file
    links to the original source files in an auxiliary directory which is then added to
    `sys.path`, while legacy "compat mode" editables are implemented by placing the entire
    root of your package on `sys.path`. The new method is designed to ensure only files
    present at installation are exposed, mimicking a normal installation.

[#1680]: https://github.com/pypa/pip/issues/1680
[@sethmlarson]: https://sethmlarson.dev/
[certifi]: https://pypi.org/project/certifi/
[changelog]: https://pip.pypa.io/en/stable/news/#v24-2
[graalpy]: https://github.com/pypa/pip/issues/12892
[hatch]: https://hatch.pypa.io/latest/
[issue-10777]: https://github.com/pypa/pip/issues/10777
[issue-12560]: https://github.com/pypa/pip/issues/12560
[paul ganssle article]: https://blog.ganssle.io/articles/2021/10/setup-py-deprecated.html
[pep 517]: https://peps.python.org/pep-0517/
[pep 518]: https://peps.python.org/pep-0518/
[pep 621]: https://peps.python.org/pep-0621/
[pep 660]: https://peps.python.org/pep-0660/
[pip 24.2]: https://discuss.python.org/t/announcement-pip-24-2-release/59402
[poetry]: https://python-poetry.org/
[pull-1866]: https://github.com/pypa/pip/pull/1866
[scikit-build-core]: https://scikit-build-core.readthedocs.io/
[setuptools 64.0.0]: https://setuptools.pypa.io/en/latest/history.html#v64-0-0
[setuptools' build system docs]: https://setuptools.pypa.io/en/latest/build_meta.html
[setuptools' editable install docs]: https://setuptools.pypa.io/en/latest/userguide/development_mode.html
[strict editable installs]: https://setuptools.pypa.io/en/latest/userguide/development_mode.html#strict-editable-installs
[system-ca-revert]: https://github.com/pypa/pip/commit/c77d4ab55ed412a3b72d0b73f504e8ddec918683
[truststore]: https://pypi.org/project/truststore/
