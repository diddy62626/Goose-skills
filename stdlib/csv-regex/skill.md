---
name: goose-csv-regex
title: csv and regex modules — RFC 4180 records, Rust-regex-style matching
category: stdlib
tags: [goose, csv, regex, each_record, format_field, compile, captures, each_match]
triggers: [goose csv, goose regex, parse csv, regex find, regex captures, format_replaced]
description: >
  csv::each_record streams RFC 4180 records as zero-copy field slices with
  full quoting support; csv::format_field quotes where needed. regex has
  Rust-regex syntax and leftmost-first matching, complete DFAs compiled up
  front, find/matches/count/each_match/each_captures/format_replaced, with
  the supported syntax and known limits.
source: https://github.com/aardappel/goose/blob/main/docs/stdlib.md#csv
---

# `csv` and `regex`

## csv — RFC 4180

```goose
fn each_record<F>(text: const u8[:], sep: u8 = ',') -> bool   // F(fields: const u8[:][:]) per record
fn format_field(out: u8[>..]&, s: u8[:], sep: u8 = ',')        // s, quoted where needed
```

```goose
var total = 0.0;
let ok = csv::each_record(text) { fields =>
    let price, valid = parse_flt(fields[2]);
    if valid { total += price; }
};
if !ok { abort("malformed CSV"); }
```

- Fields are **slices of the text** — nothing copied. The slice-of-fields
  belongs to the reader and is reused for the next record.
- Quoted fields may contain separators, line breaks, doubled quotes (each
  standing for one quote); passed without the surrounding quotes, doubled
  quotes left as-is. Un-quoted fields cannot contain a quote, so
  `format_replaced(out, field, "\"\"", "\"")` appends any field's text.
- Records end at LF or CRLF (last record need not); an empty line is a
  record of one empty field; a header is a record like any other; spaces
  around a field are part of it.
- `sep` may be any byte except quote or line break (`';'`, `'\t'` for TSV).
- Returns `false` at the first malformed record (after earlier records were
  passed to the block). A block's `return` leaves the calling function.
- `format_field` appends `s` as-is or quoted with doubled quotes when it
  contains sep/quote/newline; the caller writes separators and newlines.

## regex — the syntax and rules of Rust's `regex` crate, on bytes

```goose
fn compile(pattern: u8[:]) -> Regex                   // aborts on malformed
fn compile(pattern: u8[:], err: u8[>..]&) -> Regex    // appends the error instead
fn find(re: Regex&, text: u8[:]) -> i64, i64          // start, end; -1, -1
fn matches(re: Regex&, text: u8[:]) -> bool
fn count(re: Regex&, text: u8[:]) -> i64
fn each_match<F>(re: Regex&, text: u8[:])             // F(start, end)
fn each_captures<F>(re: Regex&, text: u8[:])          // F(caps: Captures)
fn format_replaced<F>(out: u8[>..]&, re: Regex&, text: u8[:])   // F(caps) replaces each
fn group(c: Captures, k: i64) -> const u8[:]
```

```goose
let re = regex::compile("(\\w+)=(\\d+)");
regex::each_captures(re, line) { c =>
    let n, ok = parse_int(regex::group(c, 2));
    if ok { settings.insert(line[c.slots[2]..c.slots[3]], n); }   // a key that outlives c
};

var err: u8[>..] = [];
let user = regex::compile(pattern_from_user, err);
if err.len > 0 { print(err); }
```

- **Leftmost-first** matching (crate/Perl rules): earliest start wins;
  among those, the pattern's preference; alternatives in order; greedy
  unless lazy. Unanchored. Iterators find non-overlapping, never an empty
  match where the previous ended.
- Group 0 is the whole match; group k spans `slots[2k]..slots[2k+1]` (-1/-1
  if it took no part). A repeated group captures its last iteration. A
  `group` view is rooted at the `Captures` — slice the text by slots
  yourself for a view that outlives the block (e.g. a dictionary key).
- **Syntax**: literals; `.`; classes `[a-z]`/`[^...]`; `\d \w \s` and
  complements; escapes `\n \t \r \f \v \xHH` and punctuation; groups
  `(...)`, `(?:...)`, named `(?P<name>...)`/`(?<name>...)` (names accepted,
  unused); flags `i` and `s` via `(?i)`, `(?i:...)`, `(?-i)`; alternation
  `|`; `* + ? {n} {n,} {n,m}` with lazy `?`. Classes and case folding are
  ASCII.
- **Not supported** (compile reports them): anchors `^ $ \A \z \b \B`,
  class set operations, nested classes, Unicode classes, other flags.
- Matching is DFA-based (one table lookup per byte, with skip prefilters on
  rare bytes); a pattern whose DFAs would grow too large is an error. **Make
  the `Regex` once, outside the loop.**

## Related skills

- [parsing-text](../strings/parsing-text/skill.md)
- [dictionary](dictionary/skill.md)
- [json](json/skill.md)
