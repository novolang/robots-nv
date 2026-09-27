# robots-nv

`robots.txt` is a file at the root of a website saying which parts of
it an automated client may fetch. The format and the rules for applying
it are specified in
[RFC 9309, the Robots Exclusion Protocol](https://www.rfc-editor.org/rfc/rfc9309).
This package parses such a file and answers whether a given crawler may
fetch a given path. It makes no request of its own.

## What the format is

A `robots.txt` file is a list of records, one per line, each a name, a
colon and a value. A `#` begins a comment that runs to the end of the
line. Three record names are in the standard.

| Record | Meaning |
| --- | --- |
| `user-agent` | The **product token** the rules after it apply to. `*` is the catch-all |
| `disallow` | A path pattern the crawler must not fetch |
| `allow` | A path pattern the crawler may fetch |

A **group** is one or more `user-agent` lines followed by the rules
that apply to them. A group ends at the next `user-agent` line that
follows a rule. A rule written before any `user-agent` line is in no
group and applies to nobody.

A crawler finds its group by comparing its own product token with each
group's tokens, without regard to case. A product token is letters,
`_` and `-`, so `user-agent: ExampleBot/1.0` names `examplebot`. The
comparison is exact: a file naming `googlebot` does not constrain
`googlebot-image`. `*` is used only when no group names the crawler. If
more than one group names the same token, they are combined into one.

A **path pattern** is matched against the path and query of a URL,
octet by octet from the start. Two characters are special and no others
are.

| Written | Meaning |
| --- | --- |
| `*` | Any run of octets, including `/` |
| `$` | At the end of a pattern, anchors the match to the end of the path |

There is no `?` and no character class. A pattern with no `$` matches
a prefix: `disallow: /tmp` forbids `/tmpfile` as well as `/tmp/x`. A
literal `*` or `$` is written percent-encoded, as `%2A` or `%24`.

Before comparing, section 2.2.2 brings both sides to one form. An octet
outside ASCII is written as `%` and two hex digits, so a pattern holding
`ツ` matches a path holding `%E3%83%84`. In the path, an escape of an
unreserved character (a letter, a digit, `-`, `.`, `_` or `~`) is
decoded, so `%62%61%7A` is `baz`. Other escapes stay as they are, with
their hex digits compared without regard to case, so `%2F` never
matches `/`.

When several rules match a path, **the one whose pattern is longest in
octets decides** — not the first, not the last, and not the one that
matched the most of the path. If an `allow` and a `disallow` are the
same length, the `allow` wins. A path that no rule matches is
**allowed**: the file is a list of exclusions, and silence is
permission. A rule with an empty pattern, such as `disallow:` with no
value, decides nothing, and `/robots.txt` itself is always allowed.

When `robots.txt` itself cannot be fetched, the status decides.

| Status | What a crawler must assume |
| --- | --- |
| 2xx | The body is the file |
| 3xx the caller stopped following, 4xx | There are no rules; anything may be fetched |
| 5xx, or no response | Complete disallow, while it lasts |

`crawl-delay` and `sitemap` are **not** in RFC 9309. Both are written
by a great many sites and read by several large crawlers, and section
2.2.4 permits a crawler to interpret records outside the protocol. They
are read here, in a module of their own.

## Install

```
novo pkg add robots-nv
```

## Example

```novo
use robotsmatch
use robotsparse
use robotspolicy

fn main() [io]
    // The caller fetched this; nothing here makes a request.
    let text = "user-agent: *\ndisallow: /\nuser-agent: novobot\ndisallow: /tmp/\nallow: /tmp/public/\ncrawl-delay: 1.5\nsitemap: https://example.org/sitemap.xml\n"
    let rules = robotsparse.parse(text)

    // This crawler has its own group, so the catch-all does not apply.
    println("${robotsmatch.is_allowed(rules, "novobot", "/about")}")         // true
    println("${robotsmatch.is_allowed(rules, "novobot", "/tmp/x")}")         // false

    // The longer pattern decides, so the allow re-opens part of it.
    println("${robotsmatch.is_allowed(rules, "novobot", "/tmp/public/a")}")  // true

    // Which rule decided, for a log that explains a skipped URL.
    let verdict = robotsmatch.decide(rules, "novobot", "/tmp/x")
    println("${robotsmatch.verdict_text(verdict)} by line ${robotsmatch.verdict_line(verdict)}")

    // The two extensions, read as the extensions they are.
    match robotspolicy.crawl_delay_ms(rules, "novobot")
        Some(ms) => println("wait ${ms} ms between requests")
        None     => println("no delay asked for")
```

The program prints `true`, `false` and `true`, then `disallowed by
line 4` and `wait 1500 ms between requests`.

## What the package contains

| Module | Contents |
| --- | --- |
| `robotsparse` | The grammar of section 2.2: groups, rules, the records that were not understood, and the parse limit. |
| `robotsmatch` | The decision: finding the group, the longest-match rule, the tie-break, and the two special characters. |
| `robotspolicy` | What a status code means when no file arrived, and the `crawl-delay` and `sitemap` extensions. |

## How to choose an entry point

**`robotsmatch.is_allowed` answers the question a crawler has.** One
call, a boolean.

**`robotsmatch.decide` answers the same question with the rule.** With
`robotsmatch.verdict_line` it is what a log that explains a skipped URL
needs.

**`robotsmatch.matching_rules` answers every rule that matched.** A
tool explaining a site's own file to its author wants the list, because
the interesting rule is usually the one the author expected to win.

**`robotspolicy.rules_for_status` is what to use when no file
arrived.** It answers the rules the specification requires for that
status, so the asymmetry is written once.

**`robotsparse.parse` alone is enough for a linter.** The parsed value
carries the records that were not understood, with their line numbers.

## The rules a user needs

1. **Nothing here fetches anything.** The caller retrieves
   `robots.txt`, and the caller obeys a crawl delay. Making a request
   and waiting are effects; deciding is not.
2. **A path is the path and query of a URL as it appears in the URI,
   beginning with `/`.** The only changes made before comparing are the
   ones section 2.2.2 prescribes and "What the format is" lists.
   Nothing is case-folded, and `..` is not resolved.
3. **An agent is a product token.** `"novobot"`; a longer name such as
   `"novobot/1.0 (+https://example.org)"` is read up to its first
   character that cannot be in a token.
4. **The longest pattern decides.** Measured in octets of the pattern,
   after a non-ASCII octet is written as its three-octet escape, with
   `*` and `$` counted as the characters they are. Not the first rule,
   not the last. Section 2.2.2.
5. **An `allow` wins a tie.** Two matching rules of equal length are
   decided in favour of the `allow`. Section 2.2.2.
6. **A path no rule matches is allowed.** `RobotsUnmatched` is a
   separate verdict from `RobotsAllowed` so that a tool can say
   "nothing forbids it" rather than name a rule that does not exist,
   and `robotsmatch.permits` is true for both.
7. **`*` crosses a `/`.** That is the opposite of what `*` means in a
   shell or a glob library, and it is why this package does not borrow
   one.
8. **`$` only anchors at the end of a pattern.** A pattern with no `$`
   matches a prefix.
9. **Groups naming the same token are combined.** A file that names
   `googlebot` twice constrains it with both sets of rules. Section
   2.2.1.
10. **`*` is used only when no group names the crawler.** A token names
    a crawler when the two are equal without regard to case; one is not
    a prefix of the other.
11. **A rule before any `user-agent` line applies to nobody.** It is
    kept in `RobotsRules.orphan_rules` rather than dropped, because a
    file with one in it is almost always a file whose author meant
    something else.
12. **A `robots.txt` file has no syntax errors.** `robotsparse.parse`
    answers a value and never a `Result`. A record it does not
    understand is kept in `RobotsRules.unknown` with its line number,
    so a linter can report that `Disalow: /admin` is doing nothing.
13. **A 4xx opens a site and a 5xx closes it.** That asymmetry is the
    one rule a crawler is most likely to write backwards, and the
    failure mode of backwards is crawling a site hardest while it is
    already failing. `robotspolicy.rules_for_status` writes it once.
14. **`crawl-delay` is milliseconds.** Sites write it as a number of
    seconds and several write fractions. `None` means none was asked
    for, which is not the same as zero.
15. **A `sitemap` line belongs to the file, not to a group.** It
    applies wherever it is written, which is why it is not a field on
    `RobotsGroup`.
16. **A file over the parse limit is cut, and says so.** Section 2.5
    requires a crawler to parse at least 500 kibibytes and permits it
    to stop there. `RobotsRules.truncated` is how a caller finds out,
    and a line cut in half is dropped rather than read as a shorter
    one.
17. **A line ends at CR, LF or CR LF.** Section 2.2's grammar allows
    all three.

## What is not included

- **Fetching `robots.txt`, or anything else.** See rule 1.
- **Waiting out a crawl delay.** `robotspolicy.crawl_delay_ms` answers
  the number; sleeping is the caller's.
- **Caching a parsed file.** `robotspolicy.cache_seconds` answers the
  twenty-four hours section 2.4 sets, so that a caller's cache and this
  package agree on the number. Holding the cache is the caller's.
- **Following redirects.** `robotspolicy.max_redirects` answers the
  five section 2.3.1.2 allows.
- **The `host`, `clean-param` and `request-rate` records.** They are
  one search engine's extensions each. They are kept in
  `RobotsRules.unknown` rather than interpreted, and
  `robotspolicy.is_extension` says which records this package does
  read.
- **The `robots` meta tag and the `X-Robots-Tag` header.** They are a
  different mechanism, applied per page rather than per site, and they
  arrive in HTML and in HTTP headers rather than in this file.
- **Parsing or normalising a URL.** See rule 2.
  [url-nv](https://novo-lang.org/packages/url-nv) is for that, and a
  caller that normalises before asking this package will get answers no
  other crawler gives.
- **Google's lenient readings.** Google's parser accepts a record with
  no colon, such as `disallow /x`, and some misspelled record names.
  RFC 9309 requires the colon, and a misspelled name is kept in
  `RobotsRules.unknown`.

## Related packages

- [sitemap-nv](https://novo-lang.org/packages/sitemap-nv) reads the
  files the `sitemap` lines point at.
- [url-nv](https://novo-lang.org/packages/url-nv) parses a URL into the
  path and query this package takes. See rule 2 for what not to do with
  it first.
- [glob-nv](https://novo-lang.org/packages/glob-nv) matches glob
  patterns. A `robots.txt` pattern is not one: see rule 7.

## Tests

```bash
novo test tests/robotsparse_tests.nv    # the grammar, and what is kept
novo test tests/robotsmatch_tests.nv    # longest match, the tie-break, `*` and `$`
novo test tests/robotspolicy_tests.nv   # the status asymmetry and the extensions
bash tests/coverage.sh                  # line coverage over src/
```

The normative source is RFC 9309, and its examples are in the suite:
the merged group of Figure 2, the fallback to `*` of Figure 3, the
percent-encoding of Figure 4, and the special characters of Figures 5
and 6. So are the documented cases of Google's parser,
`google/robotstxt`'s `robots_test.cc`: the user-agent value read up to
its first space and without regard to case, the global group as a
fallback only, case-sensitive paths, the longest match, the special
characters, and the empty `disallow`.

Google's parser differs from RFC 9309 in two places, and the suite
follows the RFC. It does not encode a non-ASCII octet in the path
before comparing, and it does not decode an escaped unreserved
character in the path.

The suite also asserts that an unrecognised record is kept with its
line number, that CR, LF and CR LF all end a line, that a cut file
drops its partial last line, that a 404 opens a site while a 503 closes
it, and the arithmetic of `crawl-delay`.

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
