---
title: Compiling Black with mypyc, Pt. 3 - Deployment
slug: compiling-black-with-mypyc-part-3
description: &desc mypyc can compile Black and provide respectable performance gains,
  let's deploy it in production!
summary: *desc
tags: [black, mypyc]
showToc: true
date: 2022-05-31 16:27:00-04:00
aliases:
  - 31/compiling-black-with-mypyc-part-3/
---

This is part of the "*Compiling Black with mypyc*" series.

- [Pt. 1 - Initial Steps](../compiling-black-with-mypyc-part-1/)
- [Pt. 2 - Optimization](../compiling-black-with-mypyc-part-2/)
- [Pt. 3 - Deployment](.) (you're reading this one)

## Building compiled wheels with GitHub Actions

With the mypyc branch functional and pretty fast, it was time to automate building the
wheels. Not only does this make the release process easier, but **it also means I can
easily build wheels for platforms I don't have access to**, like MacOS.

To make the CI configuration process easier, I finally took a look at [cibuildwheel]. I
won't go into my exact process since it was basically trial and error + my many dumb
mistakes 😅 [^10] but here are some noteworthy takeaways:

- If you're using `setuptools-scm`, you might have to do a full clone on CI so the tag
  history is still available by build time

- **Read cibuildwheel's documentation before writing any configuration!** Seriously, if I
  did I wouldn't have to make [this commit][read-the-bloody-docs] adding `{project}` to
  the test command

- Set `CIBW_BUILD_VERBOSITY` to at least `1` because it will make debugging build errors
  (and trust me you will get some!) so much nicer

- If wheel sizes are an issue then passing `debug_level=0` to `mypyc.build.mypycify`
  should help by stripping all debug information. Not great if you hit a bunch of
  segfaults or similar though, so it's a tradeoff

So yeah, 20-ish commits later I had a basic but functional setup which could compile
wheels for x86-64 Windows, MacOS, and Linux from CPython 3.6 to CPython 3.10. Universial2
and ARM variants are also supported for the shiny M1 platform. The workflow is a bit slow,
but that's expected and not a big deal.

If you're curious, you can find the workflow here: [ichard26/black-mypyc-wheels]

> I don't recommend using my workflow to build your own wheels since it's a bit hacky and
> in need of a cleanup. I'd instead recommend taking a look at the
> [examples in cibuildwheel's docs][cibuildwheel-examples].

I was going to mention the community field testing campaign I carried out that had its own
package index, but honestly it was for the most part a failure as it reached no one. In
hindsight, I did not promote it enough so yeah this one is on me :)

## Stable release prep and shipping the wheels

Black for the longest time ever wasn't stable (see [GH-517]). The team initially aimed to
stabilize Black in late 2018, but that didn't happen for reasons. I won't go into it too
much since it'd be a whole other story, but suffice to say when we made our newest
"commitment" (at that point we explicitly worded our intentions to *not* be promises) **we
were very motivated to get it done and hopefully right.**

We drafted a [stablity policy][stability-policy],
[dropped Python 2 support][cya-python-two], [introduced the `--preview` flag][hi-preview],
and [so much more][changelog] [^11].

In this time I rewrote diff-shades into [what it is today][diff-shades], a reliable enough
tool used to [provide direct][diff-shades-comment-1]
[feedback on PRs][diff-shades-comment-2]. I tried to integrate it into the wheel build
workflow as part of the test step, but that turned out be very painful and I backed out of
it since I had a stable release to manage and publish!

*The TL;DR of it is as follows:*

Building Black with mypyc usually requires disabling pip's build isolation to work
properly as mypyc is not a standard build dependency of Black. This messes up the
installation of diff-shades which is packaged using flit. So I hacked up a script to edit
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

I honestly still have no idea what's wrong.

Anyway since I was like an hour or more into build errors, I just decided to stop and fall
back to the basic testing that was already working.[^12] I triggered the last workflow run
of the day, downloaded the artifacts, tested one of them locally to make sure nothing was
on fire, and pushed 'em to PyPI and that's how release 22.1.0 was born 🎉.

The core team chat was quite lively for the next hour, to say the least; after all, we
just finished a major milestone! Also yeah, we *may* have done a lot of publicizing for
this release which is why it was everywhere for a while :wink:

### Post release calm

To be honest I haven't really seen anyone comment on the fact Black is now compiled with
mypyc yet short of this one [mini blog post][that-one-post-about-black-mypyc][^13] ...
which is both disappointing since I spent so much time on this effort, but also calming
since usually buzz is sparked by bugs and whatnot.

Having mentioned bugs and crashes, so far we've received two reports,
[one bug probably from mypyc and another due to an unintentional restriction of Black's unofficial APIs][not-too-many-fires-so-far].

Version 22.1.0 still unintentionally broke quite a few integrations, though they weren't
mypyc related. Turns out lots of people depend on `black.files.find_project_root`
returning a `pathlib.Path` and nothing else! I've seen at least five issues / PRs on
GitHub fixing crashes related to our change making it return a tuple instead.

For this reason we, the core team, want to [define a stable API (GH-779)][gh-779] soon. I
can't promise anything as Black's development is volunteer-based, but if you were curious
to what's next for psf/black, there's this.

## Results & final thoughts

**Black is now overall 2 times faster**[^1]. And as a bonus, startup time is down too (at
most by 15%). You can [read the whole report here][perf-report]. Please note these numbers
were gathered quite a long time ago and **they are probably a bit outdated**, especially
with the somewhat recent blib2to3 changes made to support 3.10 syntax. If you'd like to
see what all of this work looked in PR form, [here ya go][gh-2431].

From my experience, I believe mypyc will make for an interesting, but viable solution to
speeding up Python code going forward. It's far from perfect and still has a bunch of
bugs, but today it is already making an impact. I don't recommend using mypyc unless you
are willing to feel a bit like a beta tester *and* have an exhaustive test suite, but
**hey I managed to do all of this without knowing much about programming in C**[^14],
which says a lot.

Going forward these open issues remain:

- **pre-commit is still slow**: since pre-commit downloads hooks from source, the compiled
  wheels won't be used. This is a shame since Black's speed is noticeable during git
  commits. I suspect we will have to introduce a shim like mypy's
  [pre-commit/mirrors-mypy], but that will be a painful transition.

- **Wheel building is slow**: this is because how mypyc's setuptools integration works
  with PEP 517 builds. The source code ends up being parsed, type checked, and
  transcompiled twice during a build. [pypa/hatch] might be a good way to fix this
  assuming its mypyc plugin doesn't suffer the same flaw.

- **We're stuck on mypyc 0.920**: a change to newer versions broke our usage of
  dataclasses and it still hasn't been fixed ...

I'm optimistic that it's only a matter of time before all of these issues are resolved.

I hope to see the mypyc project gain more contributors and succeed. Along with Cython,
PyPy, and the faster-cpython project, **perhaps Python will finally have a solid speed
story**.

> And yes, I did indeed learn a ton embarking on this project. Made mistakes along the
> way, but they turned out alright ❀

Congrats on reaching the end of this multi-part blog series!

*By the way, [Glyph has also written an excellent piece on mypyc][glyph] where he tries to
convince you to start using mypyc. You should totally read it.*

*If you would like to chat with me, [my contact details are here][contact].*

## Acknowledgements

I'd like to thank [@msullivan] for his original work on integrating mypyc into Black,
spawning this summer project and eventually this blog series :)

I'd like to also thank [@dawnofmidnight], [@kosayoda], [Jelle Zijlstra][@jellezijlstra],
and [Łukasz Langa][@ambv] for their extensive feedback and help while writing this series.
Any errors are my own, not theirs.

[^10]: If you are in need of some quality entertainment, look at this commit history:
    <https://github.com/ichard26/black-mypyc-wheels/commits/main>

[^11]: I would link to the 22.1.0 heading, but currently the HTML IDs are uhhh ... terrible
    and don't persist. I've tried fixing this with a custom sphinx extension but it
    failed. I have other ideas left to try, but yeah :(

[^12]: I was frustrated and had a headache thanks to the stress 🙃

[^13]: In the time it has taken me to edit this series, Łukasz Langa has done a [talk] about
    mypyc and Black's usage of it.

[^1]: Originally when I first landed the relevant PR it was an overall 2x improvement, but
    once Jelle added a stability hotfix the *effective* speedup for files that are changed
    is 50%. If you're formatting a bunch of already well formatted files, the speedup is
    still 2x

[^14]: Learning C (and Rust) is now one of my future goals thanks to this project

[@ambv]: https://github.com/ambv
[@dawnofmidnight]: https://github.com/dawnofmidnight
[@jellezijlstra]: https://github.com/JelleZijlstra
[@kosayoda]: https://github.com/kosayoda
[@msullivan]: https://github.com/msullivan
[changelog]: https://black.readthedocs.io/en/stable/change_log.html
[cibuildwheel]: https://cibuildwheel.readthedocs.io/en/stable/
[cibuildwheel-examples]: https://cibuildwheel.readthedocs.io/en/stable/working-examples/
[contact]: https://ichard26.github.io/about/#want-to-talk-to-me
[cya-python-two]: https://github.com/psf/black/pull/2740
[diff-shades]: https://github.com/ichard26/diff-shades
[diff-shades-comment-1]: https://github.com/psf/black/pull/2814#issuecomment-1023219426
[diff-shades-comment-2]: https://github.com/psf/black/pull/2726#issuecomment-1019067134
[gh-2431]: https://github.com/psf/black/pull/2431
[gh-517]: https://github.com/psf/black/issues/517
[gh-779]: https://github.com/psf/black/issues/779
[glyph]: https://glyph.twistedmatrix.com/2022/04/you-should-compile-your-python-and-heres-why.html
[hi-preview]: https://github.com/psf/black/issues/2751
[ichard26/black-mypyc-wheels]: https://github.com/ichard26/black-mypyc-wheels
[not-too-many-fires-so-far]: https://github.com/psf/black/issues/2846
[perf-report]: https://gist.github.com/ichard26/b996ccf410422b44fcd80fb158e05b0d
[pre-commit/mirrors-mypy]: https://github.com/pre-commit/mirrors-mypy
[pypa/hatch]: https://github.com/pypa/hatch
[read-the-bloody-docs]: https://github.com/ichard26/black-mypyc-wheels/commit/643de450e252e74040ba14fd066ea8bc23c0b0d7
[stability-policy]: https://black.readthedocs.io/en/latest/the_black_code_style/index.html#stability-policy
[talk]: https://pretalx.com/pycon-lt-2022/talk/BG7LZF/
[that-one-post-about-black-mypyc]: https://simonwillison.net/2022/Jan/30/mypyc/
