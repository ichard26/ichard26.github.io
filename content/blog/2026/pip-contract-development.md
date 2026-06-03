---
title: I'm working on pip part-time for three months
slug: pip-contract-development
description: &desc >
  The pip project has secured 4000 USD in funding from the Packaging-WG/Python Software
  Foundation to have me work on pip on a contract part-time basis from June to August, 2026.
  Let me explain what you can expect from this exciting chapter!
summary: *desc
date: 2026-06-03
tags: [pip, pip-contract-2026]
---

This is the most exciting news I get to announce so far in my open-source tenure! If you
follow the Python packaging ecosystem closely, you may have seen [hints] to this proposal.
It's finally real, having received the greenlight from the Packaging-WG in the past few
days.

## Hang on, what's happening?

I will be working on pip part-time for 10 weeks from June 15 to August 21, 2026.
The Python Software Foundation, through the [Packaging-WG], has granted us with 4000 USD
for 120 hours of work.

Admittedly, 120 hours is not a lot of time, but it's still miles better than relying purely
on volunteer contributions.[^120-hours] This role is modelled as a mix of a traditional project
contractor and the [CPython Developer in Residence] (DIR) program. I do have a list of key
deliverables:

- Use real virtual environments to isolate build subprocesses
- Removing the crusty old legacy resolver (finally!)
- Improved error reporting and UX
- Creating a proposal for configuring index priority
- General pip development and maintenance

However, I retain significant freedom and flexibility to use the paid time to work on
whatever would be most beneficial to pip. In particular, 32 hours have been allocated to
"general pip development maintenance".

## What work will I be doing?

[A public copy of the formal proposal that we sent to the Packaging-WG and PSF is available][proposal].
Among other things, it lists in detail what I am expected to work on during these three months.

If you're interested in the finer details, you should read the proposal, but for convenience,
I have copied and summarized the key deliverables.

### Use real virtual environments for build isolation

**TL;DR expected outcomes:**

- Add a `--use-feature venv-isolation` flag in pip 26.2
- Use real virtual environments by default in pip 27.0

To isolate build subprocesses from the environment pip is invoked in, pip uses a custom
`sitecustomize.py` and `PYTHONPATH` environment variable tricks. This has worked
surprisingly well, but over the years, the faux virtual environments are proving to be
[quite][gh-13222] [troublesome][gh-11812] at [times][gh-7778].

It's time that pip uses `venv`. Notably, uv has been using real virtual environments for
their build subprocesses which is good evidence that this should be feasible.

This feature will only be supported with in-process build dependencies. Timing-wise, the
subprocess installer will be deprecated and slated for future removal at a similar time.
It'd be ideal to ship these two major QoL improvements sequentially.

### Remove the legacy resolver

**TL;DR expected outcomes:**

- I gain familiarity with the 2020 resolver
- Perform outreach and communicate the impending legacy resolver removal
- File fixes and tweaks to aid in the transition, as per user feedback
- Schedule the legacy resolver removal for pip 27.0

This is the largest unresolved deprecation. A substantial portion of tech debt is related
to the legacy resolver. Pradyun Gedam has filed fixes for the bugs and feature gaps that
prevent us from outright removing it. However, there is considerable communication work
needed to ensure a smooth transition.

It's worth noting that I'm wholly unfamiliar with pip's resolver, thus a secondary goal
here is to gain familiarity in the dependency resolution logic. This is important for
long-term sustainability since there is only one maintainer actively working on dependency
resolution.

> [!important]
> Removing support for `--use-deprecated=legacy-resolver` is out of scope. As [per current
> discussions][legacy-issue], the earliest support can be removed is in pip 26.3 in October.
> The transitional/pre-removal work represents the bulk of the effort though, so this is not a
> major concern.

### Improve error reporting and UX

pip errors are often hard to scrutinise and offer little help. Furthermore, they can be
intimidating to new users. Things don't have to stay this way, we can:

- Reign in the [warning spam when a HTTP request times out](https://github.com/pypa/pip/issues/5380#issuecomment-2157611325)
- Explain why no matching distributions were found in most common scenarios
- [Explain why a given wheel is incompatible with the platform](https://github.com/pypa/pip/issues/10793)
- Provide advice on [network](https://github.com/pypa/pip/issues/3642)/[TLS errors](https://github.com/pypa/pip/issues/8473)
- Fail fast on invalid option combinations

In particular, I'm looking forward towards improving the following (rather unhelpful) warnings
and errors:

```
WARNING: Retrying (Retry(total=4, connect=None, read=None, redirect=None, status=None)) after connection broken by 'ReadTimeoutError("HTTPSConnectionPool(host='files.pythonhosted.org', port=443): Read timed out. (read timeout=15)")': /path/to/some-package-1.0.0-py3-none-any.whl.metadata
```

```console
$ pip install black
ERROR: No matching distribution found for black
```

```console
$ pip install  black-25.1.0-cp313-cp313-win_amd64.whl
ERROR: black-25.1.0-cp313-cp313-win_amd64.whl is not a supported wheel on this platform.
```

### Create a proposal for index priority

**TL;DR expected outcomes:**

- Summarize the needs that an index priority feature needs to cover
- Research prior art
- Come up with experimental designs and conduct user research
- Draft an agreed upon proposal to bring index priority to pip
- Develop an implementation, if time permits

Index URL priority has been a thorny missing pip feature since 2018. There are
many issues on the pip issue tracker related to index priority. This feature has been
discussed at length. To echo Paul Moore's words, [it's time for someone to sit down,
design, and propose an implementation with care][paul moore wisdom]. The good news is
that there is prior art available.

PEP 708 was the first attempt at solving the larger "dependency confusion" issue. It was
conditionally accepted 2.5 years ago, but it was [recently rejected due to lack of progress
on its implementation][pep 708 rejection].

Since then, [uv has added its own index configuration feature] to seemingly good success.
It would likely be worthwhile to take inspiration from uv's implementation.

The goal under this contract is to design an agreed upon proposal to bring index URL
priority to pip.

In the spirit of "practicality beats purity," proposing a smaller but viable path forward
will be preferred over proposing a large, but unwieldy change. If there is sufficient time
after an agreed upon design is available, I will start implementation.

> [!important]
> Standardizing index priority is explicitly out of scope. Trying to push through a PEP
> is not something I want to take on. Plus, it'd be exceptionally difficult to achieve in
> three months.

### General development and maintenance

While there are no exact deliverables by design, here is a subset of my perennial to-do list:

- **Managing the pip 26.2 (July 2026) release.** In particular, authoring and cutting any
  post-release hotfixes as needed. Also writing a release blog post which will include
  details on work achieved during the June-July period.

- [**Improving installation scheme handling**][gh-6052]. The goal would be
  to ensure there is a consistent working scheme in any pip invocation. Adopting a working
  scheme would lay the groundwork for adding upgrade and uninstallation support to the
  `--target` flag, [which has been a consistent pain point][gh-11366].

- **General optimization work.** Optimizing pip properly would be its own project, but
  there are some low-hanging fruit available. Any work on this would consist of "Richard
  looks a random profile of a pip run and looks out for anything abnormally slow"

  - It would also be beneficial to create a document outlining the main ways pip can be
    optimized, their benefits and costs, and their difficulty. This would serve as a
    foundation for a potential future project on improving pip performance.

- **Catching up on PR review**. In particular, supporting other maintainers' PRs so their
  efforts don't stall, including the very important `pylock.toml` feature work.

What exactly I work on is subject to change.

## What about progress updates?

As documented in the proposal, "weekly updates are to be posted and logged in a publicly
accessible location."

I haven't yet decided what exactly these regular updates will look like[^update-time],
but I will commit to sharing updates via a [public issue on the pip tracker][updates-issue].
It may just consist of links to my personal blog, but I do want to make it easy to find
updates.

## Why seek funding?

It is often difficult for open-source projects to sustain themselves purely on volunteers.
Maintainer burnout is a [real][maintenance-crisis-1] and [growing crisis][maintenance-crisis-2].
In a sense, the pip project is lucky since it has 3-5 active core maintainers, *but*,
we're also a *very* busy group of people. While our collective volunteer efforts are enough to keep
the lights on, pip's pace of feature development has been highly constrained by limited maintainer
availability.

As shown by the rapid growth of [Astral's uv], there is an undeniable benefit to having
dedicated engineering hours for project development.

pip has undergone paid development before. The new pip resolver
was a grant-funded project managed by the Packaging-WG, primarily occurring in 2020.
The project was a success, with pip 20.3 gaining a new dependency resolver that was more
reliable, predictable, and user-friendly. pip still uses it to this very day! None
of this would've been possible without the funding that hired three individuals for
full-time work.

I respect and appreciate the Astral team for building a high-quality, feature-rich and unified
toolchain for Python packaging. They absolutely deserve the success they've had with uv, and yet,
this doesn't mean pip is a dead project. There are still a place for pip. It's shipped in every
CPython build, referenced in thousands of tutorials and guides, and it's an community owned
and maintained alternative, which is important when foundational developer software is
being [increasingly][bun] [bought][gel] and [controlled by private companies][astral].[^astral pressure]

Admittedly, while sponsoring 2.5 months of part-time development is *not* going to fix the
long-term maintainer availability shortfall pip is experiencing, this is an excellent opportunity
to make significant headway on new features, pay down technical debt, and put pip on better
footing to implement further improvements after the contract period.

## Looking beyond pip and this summer

One of my aspirational goals with this work is to trailblaze further paid development
in the Python packaging ecosystem which is already occurring at considerable levels:

> Starting in 2017, we began applying for grants to support PyPI development work. This endeavor
> was largely led by Sumana Harihareswara [...] It's also been wildly successful [...] more than
> $750K has been directed at PyPI development specifically, and nearly $2M has been directed
> at Python packaging projects in general (including PyPI) [as of 2021].

-- Dustin Ingram, in ["What does it take to power the Python Package Index?"][pypi-funding]

However, what's been less common is funding *general* maintenance and development of Python
packaging tools and infrastructure. It takes a lot of work to apply for major project grants.
For one, you have to usually find an external organisation to fund it since the PSF cannot
provide hundreds of thousands of dollars. Of course, they can be worth the effort since if
approved, they provide the massive sums that enable major new features (e.g., adding organisation
accounts to PyPI).

The thing is that not every project can or should apply for these major grants. Some maintainers
won't have the capacity for large-scale grants. Some other projects may be largely feature complete
and don't have any new features that need major funding. Sometimes, just having paid time
to work on bugfixes and documentation is what's most needed.

I'd like to see smaller grants that enable small-scale development and
maintenance, similar to this grant. It should be easier to seek funding for work that's difficult
to complete solely by volunteer contributions, but also doesn't require hiring an expensive
team of developers (e.g., the pip resolver grant).

These smaller grants need not be restricted to the development or maintenance of code. There
are many PEPs being proposed and considered in the packaging space. While we try our best
by seeking community consensus, there have been [PEPs where conducting user research
would've been helpful][user research]. User research is one example of non-code valuable
work that's usually only practical with funding.

Python packaging governance is in the midst of an overhaul. The PyPA, Python Steering Council,
and PSF Board have recently approved [PEP 772] which replaces the standing delegations process
with a formal Packaging Council (PC). This council will have broad delegated authority over
packaging standards, tools, and implementations. As per the PEP, one of the key expectations
for the PC is to "facilitate tactical and fundraising support from the PSF, to increase
capacity and **funding available to packaging tools**."

Once the inaugural Packaging Council has been elected and begins operations later this year,
**I strongly encourage the council to establish a clear process for packaging projects and
contributors to apply for Packaging Council funding**.

It should not be the case that the PyPA has no procedure for approving and disbursing its
funds for packaging work, [as I would find out and try to fix][process hell] while planning
this proposal.[^wg] I should not have to wait months to hear back on administrative details that
could've been easily documented in public.[^forgiveness]

Overall, I hope that this pip grant serves as motivation and an helpful example for future
grants of similar scope and type.

What will happen with pip after my contract ends in mid-August? I don't know yet. I'd love
to see further paid development, but it's impossible for me (or the rest of the pip team)
to plan out that far in the future.

## Acknowledgments

I want to express my greatest gratitude to the Packaging-WG and PSF for providing the
funding to make this work possible.

I'd like to also shout out Alyssa Coghlan, Dustin Ingram, Jannis Leidel, and the fine
folks at PSF Accounting for helping me navigate the administrative realities when it comes to
seeking a formal grant.

Finally, thank you to my fellow co-maintainers for entrusting me with the privilege to
work on pip for money. It's not often that one can turn their open-source work into
a paid gig, even a temporary one!

In the words of Łukasz Langa, the inaugural CPython DIR and my first collaborator in
open-source, [I'll do my best not to disappoint].

[^120-hours]: I have limited personal availability. Also, increasing the proposal amount much more
  than we got would've presented additional challenges for approval. It's easier to start small,
  prove that pip can make good use of paid development, and then plan for bigger things later.

[^forgiveness]: While this was frustrating, I do not wish to place blame on anyone for the delay.
  Anything that involves money is often complicated, and I fully understand that doing things fully
  by the books is of paramount importance. I just hope that "doing things by the books" is made
  as straightforward as possible.

[^wg]: I would eventually decide to apply for Packaging-WG funding on the advice of Alyssa Coghlan
  as seeking an (interim) amendment to PyPA governmance was likely to be challenging and a slow
  enough process such that the window of opportunity for this work would've passed.

[^astral pressure]: I trust the Astral team to do the right thing, but one cannot ignore the
  obvious pressure and challenges that they face being owned by a private company that
  is desperate to grow revenue and reduce operating costs whenever possible.

[^update-time]: While regular updates are important, I also don't want to spend too much time writing
  updates. It's a balancing act in terms of update detail vs time spent writing updates.

[CPython developer in residence]: https://lukasz.langa.pl/a072a74b-19d7-41ff-a294-e6b1319fdb6e/
[maintenance-crisis-1]: https://www.fordfoundation.org/learning/library/research-reports/roads-and-bridges-the-unseen-labor-behind-our-digital-infrastructure/
[maintenance-crisis-2]: https://www.sonarsource.com/open-source-maintainer-survey-2023.pdf
[Astral's uv]: https://github.com/astral-sh/uv
[pypi-funding]: https://dustingram.com/articles/2021/04/14/powering-the-python-package-index-in-2021/
[user research]: https://discuss.python.org/t/pep-722-723-decision/36763/3
[PEP 772]: https://peps.python.org/pep-0772/
[DPO thread #3]: https://discuss.python.org/t/pep-772-packaging-council-governance-process-round-3/100181/126
[process hell]: https://discuss.python.org/t/withdrawn-amending-pypa-governance-to-permit-for-use-of-pypa-funds/106653/
[I'll do my best not to disappoint]: https://lukasz.langa.pl/a072a74b-19d7-41ff-a294-e6b1319fdb6e/#why-me
[hints]: https://discuss.python.org/t/withdrawn-amending-pypa-governance-to-permit-for-use-of-pypa-funds/106653/3
[Packaging-WG]: https://wiki.python.org/psf/PackagingWG
[bun]: https://www.anthropic.com/news/anthropic-acquires-bun-as-claude-code-reaches-usd1b-milestone
[gel]: https://vercel.com/blog/investing-in-the-python-ecosystem
[astral]: https://openai.com/index/openai-to-acquire-astral/
[proposal]: https://docs.google.com/document/d/1-0InfRaXGnT9aTUuNNVqAxgR5fneB75ODScPTA_0kJg/edit?usp=sharing
[updates-issue]: https://github.com/pypa/pip/issues/14025
[gh-11366]: https://github.com/pypa/pip/issues/11366
[gh-6052]: https://github.com/pypa/pip/issues/6052
[gh-13222]: https://github.com/pypa/pip/issues/13222
[gh-11812]: https://github.com/pypa/pip/issues/11812
[gh-7778]: https://github.com/pypa/pip/issues/7778
[legacy-issue]: https://github.com/pypa/pip/issues/10946#issuecomment-4195828617
[uv has added its own index configuration feature]: https://docs.astral.sh/uv/concepts/indexes/
[pep 708 rejection]: https://discuss.python.org/t/proposal-withdraw-reject-pep-708-extending-the-repository-api-to-mitigate-dependency-confusion-attacks-due-to-lack-of-adoption/106800
[paul moore wisdom]: https://github.com/pypa/pip/issues/8606#issuecomment-1370303166
