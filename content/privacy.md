---
title: Privacy Statement
description: TL;DR, I'm doing my best to not track you and respect your privacy as
  much as practically possible. The only data processor is GitHub Pages.
layout: single
showReadingTime: false
modified: 2023-02-26
---

## Scope

All content and resources served under <https://ichard26.github.io/> are subject to this
privacy statement unless noted otherwise.

For the time being, I will not embed content from social media platform like Twitter or
YouTube directly. Instead, I will copy the relevant text and make sure to link back to the
source to help preserve your privacy.

## What is collected and not?

This website does not use any cookies and all data used for analytics is anonymized.
Please note as this website is hosted and served by [GitHub Pages][pages] (GitHub, Inc.)
which does
[**log IP addresses regardless whether you are signed into a GitHub account or not**][pages-data].

## Cookies & localStorage

While no first-party or third-party cookies are used, your browser may report the presence
of one "cookie". This "cookie" is really an entry in your browser's localStorage. The data
stored is purely used for functionality including:

1. **Your light/dark theme preference** -> so it can persist across reloads

1. **Your last vertical position** -> so if you reload the same page it will autojump to
   your former location

## Exceptions

### Next PR Number

> https://ichard26.github.io/next-pr-number/

- **Uses third-party cookies**: The GitHub REST API sets third-party cookies upon a
  request sadly

## History

- **February 26, 2023**: microanalytics.io discontinued their free plan so all web
  analytics have been removed. That also means that <https://ichard26.github.io/ghstats/>
  now falls under this privacy statement without any exceptions (it never had
  microanalytics.io enabled).

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
