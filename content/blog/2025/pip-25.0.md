---
title: What's new in pip 25.0
slug: whats-new-in-pip-25.0
description: &desc >-
  pip 25.0 adds support for SPDX License Expressions (PEP 639), build environment bugfixes, and
  further optimizations among other changes.
summary: *desc
date: 2025-01-31
tags: [pip, release]
showToc: true
---

On January 26, 2025, [the pip team released pip 25.0][announcement].

Before delving into the key changes, I'd like to mention that the pip project has two new
maintainers: [Damian Shaw] and Richard Si (that's me!). I want to thank the pip core team
for entrusting me with the commit bit on such a foundational project like pip.

Also, this is the first release cut from a GitHub Actions CI/CD workflow that implements
[Trusted Publishing] and bundles [PEP 740 digital attestations].

As always, please read the [changelog] for the full list of changes.

## Key features

### PEP 639: SPDX License Expressions

About 5 months ago,
[PEP 639 – Improving License Clarity with Better Package Metadata][pep-639] was approved.

It added the `License-Expression` core metadata field, which is mandated to contain a
valid [SPDX expression]. In addition, there is now a multi-use `License-File` field so
packages can declare the path to their licensing-related files.

> [!warning]
> The old free-form `License` field and license classifiers are **deprecated**, although
> there are no current plans to remove them from the metadata specification.

It took a few months, but it is now generally supported by build backends, publishing
tools, and PyPI. With this release, `pip show` will display the `License-Expression` over
the `License` field when available.[^metadata-version]

```console { hl_lines=[9] }
$ pip install prettytable==3.13.0
$ pip show prettytable
Name: prettytable
Version: 3.13.0
Summary: A simple Python library for easily displaying tabular data in a visually appealing ASCII table format
Home-page: https://github.com/prettytable/prettytable
Author:
Author-email: Luke Maurits <luke@maurits.id.au>
License-Expression: BSD-3-Clause
Location: /home/ichard26/.local/lib/python3.12/site-packages
Requires: wcwidth
Required-by:
```

Furthermore, the `License-Expression` and `License-File` fields are included in the JSON
report emitted by `pip install --report` and `pip inspect`.

Thank you Philippe Ombredanne, C.A.M. Gerlach, and Karolina Surma for authoring the PEP!
Doubly so to Karolina who drove the PEP to completion and contributed part of pip's
implementation.

### Network cache inherits permissions from root cache directory

It's easiest to explain this change by [quoting the original issue][#11012]:

> On a shared Linux system, we want to share pip's cache between multiple users, so
> packages are not downloaded 50 times when 50 users install the same package.

Problem is that pip would set the permissions for network cache entries so that the user
running pip is the sole user able to access them. This wasn't intentional, but a byproduct
of `tempfile.NamedTemporaryFile`.

Now, pip will update the entries' permissions to inherit the read/write permissions set on
the root cache directory. Thus, if the root cache directory is accessible by other users,
those users will be able to use any entries added later.[^stale-cache] Thank you to Justin
van Heek for contributing this improvement!

## Key bugfixes

First off, here's a rapid-fire list of smaller, but still noteworthy bugfixes:

- **Fix a security bug allowing a specially crafted wheel to execute code during
  installation.**

  [You should read this issue][#13079] for all of the details, but essentially, a deferred
  import of pip's self check could be abused to run arbitrary code at the end of the
  _same_ pip install. Reported and fixed by Caleb Brown.

- **The inclusion of [packaging] 24.2 changes how pre-release specifiers with `<` and `>`
  behave.**

  Including a pre-release version with these specifiers now implies accepting pre-releases
  (e.g., `<2.0dev` can include `1.0rc1`). To avoid implying pre-releases, avoid specifying
  them (e.g., use `<2.0`). The exception is `!=`, which never implies pre-releases.

### Certificate and proxy options are used for build dependencies

When pip needs to build a project for installation or metadata[^build-for-metadata], pip
by default sets up an isolated environment to install the build backend and its
dependencies in. These build dependencies are installed by calling pip recursively in a
subprocess.

Unfortunately, the `--cert`, `--client-cert`, and `--proxy` options were not passed down
to the pip subprocesses. However, `--index` always[^always] has been, resulting in the
common problem where pip tries to install build dependencies from a private index that
uses a custom certificate... that _it can't verify_.

Before pip 25.0, the workaround was to use environment variables (e.g., `PIP_CERT`) as
those are passed down to the subprocess by the OS. **This is no longer necessary.**

The `--proxy` patch was authored in part by Luis Carlos. Thanks!

> [!warning] Truststore regression
> While fixing this, [truststore was inadvertently disabled] while installing build
> dependencies. Truststore enables system CAs to work without any configuration. This will be fixed in an upcoming
> pip 25.0.1 bugfix release.

> [!memo] "that seems silly!"
> Admittedly, the whole "pip calling itself" logic is not
> ideal. There is a performance penalty to starting a new pip process, numerous bugs,
> unexpected behaviour caused by forgetting to pass enough state (i.e. pip flags), and the
> error reporting is abysmal.
>
> [We would like to pay down the debt that has accumulated here][#9081], but installing
> build dependencies in-process is likely to require substantial refactoring as the
> codebase was not designed w/ repeated resolve-download-install cycles in mind.

## Faster package candidate collection

When pip initially fetches the list of distributions available for each package, it needs
to parse the [HTML or JSON returned by the index][simple-api] and filter out the
distributions that obviously aren't supported by the current platform.

This release eliminated redundant URL parsing and similar processing, especially for
indices which serve absolute URLs to their distribution files (e.g., PyPI). Furthermore,
`requires-python` checks are cached.

The URL processing and `requires-python` checks are fast individually, but they add up
once you start to use packages with hundreds of releases and thousands of distributions. A
good example is setuptools. It has ~1500 different distributions that could be eligible
for installation, leading to a fair bit of overhead:

| `pip install setuptools --dry-run` | pip 24.3.1 | pip 25.0 |
| ---------------------------------- | ---------- | -------- |
| Index page parsing                 | 90ms       | 45ms     |
| Candidate evaluation               | 130ms      | 30ms     |

Obviously the network overhead usually dominates in practice, but this is still a good
improvement! And since setuptools is a common backend, this makes the build environment
setup phase of installing from source just a bit faster for many projects. 🏎️

## Upcoming removals

Finally, here is a list of deprecations currently scheduled for removal in pip 25.1 (Q2
2025). As always, this schedule is subject to change. **Any given removal may be pushed to
a future release as needed.**

Legacy (setup.py) editable installs _Deprecated since pip 24.2_
: I've already said enough on this deprecation, so I'm going to simply ask that you read
  [the deprecation issue for information and advice][#11457]. This was originally
  scheduled for removal in 25.0, but it got pushed back as the removal wasn't finished in
  time.

Legacy installed .egg distributions _Deprecated since pip 23.2_
: ([#12330]) On Python 3.11 or later, support for detecting and uninstalling installed
  `.egg` distributions is deprecated. Eggs are the legacy format used to record installed
  distributions. It has been long since replaced by [`.dist-info`]. Today, they're most
  often the product of using `setup.py install`. The recommended solution is to reinstall
  the distribution using a recent pip. FWIW, this removal has been pushed back several
  times.

Non-bare project name in egg fragment _Deprecated since pip 23.0_
: ([#13157]) If you're ever seen `#egg=mypkg` at the end of an URL requirement, that's
  called an egg fragment. It's the old syntax used to tell pip what project the URL is
  for. They've been largely replaced by the standardized [Direct URLs references]. When
  egg fragments were initially added, they were meant to only contain the project name.
  You can technically put a version specifier in the fragment, but pip won't respect it.
  To avoid confusion, anything but a bare project name is deprecated. However, the removal
  is blocked on adding support for requesting extras via the Direct URL syntax for VCS
  references.

Non-standard wheel filenames _Deprecated since pip 24.3_
: ([#12938]) For historical reasons, pip does its own parsing of wheel filenames. We'd
  like to replace this with the reference parser in [packaging]. The side-effect is that
  wheels that violate the [wheel filename specification] are now rejected. Unless you use
  old packaging tooling or play shenanigans with your project name or wheel filenames, you
  shouldn't be affected.

-​-no-python-version-warning _Deprecated since pip 25.0_
: ([#13154]) The flag has long done nothing since Python 2 support was removed in pip
  21.0. pip has also never warned about deprecations of any other Python version as the
  flag's help suggests.

[^metadata-version]: As long as the package's metadata version is 2.4 or higher. There was a bit of a
    kerfuffle as PEP 639 was implemented by certain backends before being accepted, thus
    they generated distributions without metadata 2.4 declared (because it didn't exist
    before approval). This is technically invalid, and pip wishes to enforce specification
    compliance.

[^stale-cache]: Of course, this doesn't help with old cache entries with the overly restrictive
    permissions. Plus, everyone must be using pip 25.0 or else there will be a mix of
    entries with the right and overly restrictive permissions, rendering the shared cache
    useless.

[^build-for-metadata]: For example, during dependency resolution, pip may need to build metadata so it knows
    what requirements `mypkg` needs. Building a wheel and inspecting its metadata is a
    valid approach, but PEP 517 allows pip to ask a backend for only metadata.

[^always]: Maybe not always, but definitely for a long time.

[#11012]: https://github.com/pypa/pip/issues/11012
[#11457]: https://github.com/pypa/pip/issues/11457
[#12330]: https://github.com/pypa/pip/issues/12330
[#12938]: https://github.com/pypa/pip/issues/12938
[#13079]: https://github.com/pypa/pip/issues/13079
[#13154]: https://github.com/pypa/pip/issues/13154
[#13157]: https://github.com/pypa/pip/issues/13157
[#9081]: https://github.com/pypa/pip/issues/9081
[announcement]: https://discuss.python.org/t/announcement-pip-25-0-release/78392
[changelog]: https://pip.pypa.io/en/latest/news/#v25-0
[damian shaw]: https://github.com/notatallshaw
[direct urls references]: https://packaging.python.org/en/latest/specifications/version-specifiers/#direct-references
[packaging]: https://packaging.pypa.io/en/stable/
[pep 740 digital attestations]: https://peps.python.org/pep-0740/
[pep-639]: https://peps.python.org/pep-0639/
[simple-api]: https://packaging.python.org/en/latest/specifications/simple-repository-api/
[spdx expression]: https://spdx.github.io/spdx-spec/v2.2.2/SPDX-license-expressions/
[trusted publishing]: https://docs.pypi.org/trusted-publishers/
[truststore was inadvertently disabled]: https://github.com/pypa/pip/issues/13186
[wheel filename specification]: https://packaging.python.org/en/latest/specifications/binary-distribution-format/#file-name-convention
[`.dist-info`]: https://packaging.python.org/en/latest/specifications/recording-installed-packages/
