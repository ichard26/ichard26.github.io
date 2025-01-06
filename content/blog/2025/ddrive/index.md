---
slug: faster-pip-ci-on-windows-d-drive
title: >-
  Speeding up pip's Windows CI by setting TEMP to the D: drive
description: &desc >-
  By simply moving all temporary file I/O onto the D: drive, pip's
  Windows CI times decreased by up to 30%.
summary: *desc
date: 2025-01-05
tags: [pip]
---

> [!conclusion]
>
> TL;DR \<words>

A few weeks ago, I was waiting for pip's CI to turn 🍏 so I could merge a PR.
I'd just spent an hour on review, having pushed some minor changes, so
waiting an additional 20 minutes was the last thing I wanted to do.[^spoiled]

I was sufficiently annoyed with checking and rechecking the CI status,
so I enabled [auto-merging on the repository] so I could at least focus
my attention on something entirely different while waiting for CI to pass. That smoothed
over the pain, but it got me wondering, can pip's tests (and CI in particular)
be faster?

It turns out for the Windows CI jobs, the answer was: **yes, they could be _much_ faster.**

## How slow was pip's Windows CI?

[trimmed redundant work]: https://github.com/psf/black/pull/2205
[added `pytest-xdist` for parallelized testing ]: https://github.com/psf/black/pull/2196

At the time of writing, pip's Github Actions CI workflow consists of 28 jobs. 23
run the test suite across Ubuntu, macOS, Windows, and CPython 3.8 through 3.13.

- 7 on the `ubuntu-latest` runners
- 6 on the `macos-latest` (ARM) macOS runners
- 6 on the `macos-13` (Intel) runners
- 4 on the `windows-latest` runners

The extra Ubuntu job runs the functional test suite against a [zipapp] version
of pip.

On the other hand, the Windows jobs have been so slow historically[citation-needed]
that we don't even test pip across the entire supported Python version matrix.
We only test the oldest and newest version, 3.8 and 3.13, on Windows. Plus, the
test suite is [sharded over two jobs per version] because—as you can probably guess—the Windows jobs are
that much slower. 😔

I wrote a script that queries the GitHub API and calculates the average job duration
over the last X successful runs. Before the changes discussed later, this is what pip's
CI looked like:

```text { hl_lines=[15, 17, 25, 27] }
                    Last 50 CI runs on main

  Job                            Mean    Minimum   Stdev
 ─────────────────────────────────────────────────────────────
  tests / 3.10 / macos-latest    6:25    5:44      0:34 (8%)
  tests / 3.11 / macos-latest    6:40    6:03      0:39 (9%)
  tests / 3.9 / macos-latest     6:57    6:03      0:55 (13%)
  tests / 3.13 / macos-latest    6:59    6:15      0:37 (8%)
  tests / 3.12 / macos-latest    7:09    6:14      1:01 (14%)
  tests / 3.8 / macos-latest     7:11    6:32      0:31 (7%)
  tests / 3.11 / ubuntu-latest   9:33    9:08      0:10 (1%)
  tests / 3.10 / ubuntu-latest   10:04   9:43      0:14 (2%)
  tests / 3.12 / ubuntu-latest   10:34   10:16     0:14 (2%)
  tests / 3.13 / ubuntu-latest   10:35   10:10     0:12 (2%)
  tests / 3.13 / Windows / 2     10:39   9:55      0:20 (3%)
  tests / 3.8 / ubuntu-latest    11:02   10:41     0:14 (2%)
  tests / 3.13 / Windows / 1     11:13   10:29     0:24 (3%)
  tests / 3.9 / ubuntu-latest    11:44   11:20     0:19 (2%)
  tests / 3.10 / macos-13        12:10   8:53      2:16 (18%)
  tests / 3.11 / macos-13        12:59   9:09      2:34 (19%)
  tests / 3.13 / macos-13        12:59   9:54      2:41 (20%)
  tests / 3.9 / macos-13         13:10   10:09     2:42 (20%)
  tests / 3.8 / macos-13         13:34   10:09     2:30 (18%)
  tests / 3.12 / macos-13        13:55   9:54      3:23 (24%)
  tests / 3.8 / Windows / 2      14:58   14:04     0:26 (2%)
  tests / zipapp                 16:53   16:27     0:15 (1%)
  tests / 3.8 / Windows / 1      17:37   16:40     0:30 (2%)
```

Among the slowest three jobs, the Windows 3.8 jobs are 3rd and
1st slowest. Yikes!

Let's see if I can make that faster...

### Windows file I/O is poor on GitHub Actions

Now, I am no Windows expert. I stopped using the OS on a daily basis
many years ago. Either way, [it's a **known issue that file I/O on
the Windows runners is noticeably slower**][slow-windows-io] than the Ubuntu and macOS runners.

As a package installer, **pip's test suite is especially I/O heavy**. Essentially
every functional test requires at least setting a scratch environment and
installing something into it. Countless files and directories are created and
deleted during a run.

## Moving TEMP to the D: drive

[According to Steve Dower], the GHA Windows workers have a **slow C: system drive and a
faster D: drive for working space**.

For many projects, this is _probably_ fine. The repository is still checked out on the D: drive.
Any files you create in the checkout will be on the D: drive. For pip, however, all of the
temporary files are created under the system temporary file path, i.e. the `TEMP` environment
variable. And where is this path set to by default? On the C: drive.[^we-set-it-explicitly]

Thus, by setting `TEMP` to `D:\\Temp`, the average Windows CI times decreased by up to 30%. 🎉

| Job | Before | After | % decrease  |
|---|---|---|---|
| Python 3.8 (1) | 17m 37s | 12m 41s | 28% |
| Python 3.8 (2) | 14m 58s | 10m 44s | 28% |
| Python 3.13 (1) | 11m 13s | 9m 53s | 11% |
| Python 3.13 (2) | 10m 39s | 9m 32s | 10% |

Or in graph form (**note**: that the data used for this plot was collected essentially yesterday).

![image](windows-ci-times.png)


[according to Steve Dower]: https://github.com/pypa/pip/pull/13123#issuecomment-2561079373

## What about ReFS or Dev Drives?

When I was investigating

## Conclusion

Making pip's test suite faster is a whole another project. There are definitely
ways to make the functional (end-to-end) tests faster, by reducing unnecessary
installs and external network requests, but this would likely require a major
clean-up (which the test suite in dire need of, honestly).

[^spoiled]: Before you kill me for complaining, let me provide some context...
  When I used to maintain Black, I never really noticed how CI took.
  Most of the jobs would finish in 3-6 minutes. Short enough that I could
  look over my changes one last time, add notes as comments, and then
  come back to merge. \
  \
  I did notice that Black's test suite was slow, taking a good few minutes
  on the slow laptop I had at that time. I took a look at the test logic,
  [trimmed redundant work], and [added `pytest-xdist` for parallelized testing].
  Afterwards, local test suite runs took less a minute!\
  \
  I was spoiled by Black's fast tests and CI, admittedly.
