---
title: Privacy Statement
description: TL;DR, I'm doing my best to not track you and respect your privacy as
  much as practically possible. All analytics are anonymized and follow GDPR, PECR
  and CCPA. Data processors are GitHub Pages and Microanalytics.io.
layout: single
showReadingTime: false
modified: 2022-05-14
---

## Scope

All content and resources served under <https://ichard26.github.io/> are subject to this
privacy statement unless noted otherwise.

**Exception**: this statement does not apply to the following URLs prefixes as they are
served under a different GitHub repository without any data subprocessers (except for
GitHub, Inc. the host):

- <https://ichard26.github.io/ghstats/>

For the time being, I will not embed content from social media platform like Twitter or
YouTube directly. Instead, I will copy the relevant text and make sure to link back to the
source to help preserve your privacy.

## What is collected and not?

This website does not use any cookies and all data used for analytics is anonymized.
Please note as this website is hosted and served by [GitHub Pages][pages] (GitHub, Inc.)
which does
[**log IP addresses regardless whether you are signed into a GitHub account or not**][pages-data].

Analytics are collected, processed, and provided by [Microanalytics.io] which is a privacy
friendly web analytics service based in the EU. It is GDPR, PECR and CCPA compliant and
does not track with IP addresses, cookies or fingerprinting. It currently collects the
following information:

- Location as associated with your IP address

- What and when pages were accessed, and the referring websites, search engines and social
  networks

- The operating system, browser, and the screen resolution of your device

**If you would like to opt out of all analytics, please turn on Do Not Track in your
browser.** To enable DNT for the major browsers please see the following resources:

- **Google Chrome**: <https://support.google.com/chrome/answer/2790761>
- **Firefox**: <https://support.mozilla.org/en-US/kb/how-do-i-turn-do-not-track-feature>
- **Safari**: DNT cannot be enabled
- **Google Chrome for Android**:
  <https://support.google.com/chrome/answer/2790761?co=GENIE.Platform%3DAndroid&oco=1>
- **Google Chrome for iPhone & iPad**: DNT is not available at this time

## Cookies & localStorage

While no first-party or third-party cookies are used, your browser may report the presence
of one "cookie". This "cookie" is really an entry in your browser's localStorage. The data
stored is purely used for functionality including:

1. **Your light/dark theme preference** -> so it can persist across reloads

1. **Your last vertical position** -> so if you reload the same page it will autojump to
   your former location

### Exceptions

The GitHub REST API sets third-party cookies upon a request. **This only affects Next PR
Number.** (<https://ichard26.github.io/next-pr-number/>)

## History

- **May 14, 2022**: actually, microanalytics.io, or specifically
  <https://microanalytics.io/js/script.js>, doesn't set cookies if loaded into an external
  webpage. Cookies are only set if any resources under <https://microanalytics.io> are
  accessed directly. Therefore, this website does not use third-party cookies (unless you
  access <https://microanalytics.io> directly beforehand). Although, Next PR Number still
  has third-party cookies though as the GitHub API sets its own. *Hopefully I finally got
  this right. Cookies are hard.*

- **May 13, 2022**: turns out that GitHub (or at least their API) and microanalytics set
  their own cookies, so sadly third-party cookies do exist :(

- **May 13, 2022**: Next PR Number (<https://ichard26.github.io/next-pr-number/>) now uses
  [microanalytics.io] in accordance in this privacy statement

[microanalytics.io]: https://microanalytics.io/
[pages]: https://pages.github.com/
[pages-data]: https://docs.github.com/en/pages/getting-started-with-github-pages/about-github-pages#data-collection
