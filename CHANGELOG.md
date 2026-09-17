# Changelog

All notable changes to robots-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## [0.0.1] — 2026-09-17

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `robotsparse` — the grammar of RFC 9309 section 2.2. `parse` answers
  a value and never a `Result`, because section 2.2.4 says a crawler
  must ignore what it cannot interpret and a library that invented a
  refusal would refuse files every crawler reads. What it does instead
  is keep them: a record with no meaning here lands in `unknown` with
  its line number, so a linter can report that `Disalow: /admin` is
  doing nothing, and a rule written before any `user-agent` line lands
  in `orphan_rules` rather than being dropped.
- `robotsmatch` — the decision. The longest PATTERN decides, not the
  first or the last rule and not the longest match against the path;
  an `allow` wins a tie; groups naming the same token are combined, as
  section 2.2.1 requires; and a path no rule matches is allowed, with
  `RobotsUnmatched` kept apart from `RobotsAllowed` so a tool can say
  "nothing forbids it".
- `robotspolicy` — the section 2.3.1 asymmetry written once: a 4xx
  means there are no rules and a 5xx means complete disallow. It is the
  rule a crawler is most likely to write backwards, and backwards means
  crawling a site hardest while it is already failing. `crawl-delay`
  and `sitemap` are read here, in a module of their own, because
  neither is in the standard.
- Everything is a pure function of a file and a path. Fetching the
  file, waiting out a delay and caching a parse are the caller's.

### Known

- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented:
  robots-nv.<module>.<fn>`.
- No dependencies, and two obvious ones are deliberately absent.
  glob-nv would be wrong because a robots.txt pattern is not a glob:
  section 2.2.2 gives `*` and `$` and nothing else, and its `*` crosses
  a `/`. url-nv would be wrong because section 2.2.2 matches a path as
  octets and does not normalise it, and a matcher that normalised would
  disagree with every crawler in existence on a path holding `%2F`.
