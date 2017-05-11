---
title: "bash: delete first N lines matching pattern"
date:  2017-04-20 18:34:38 +0100
tags:
 - bash
 - awk
 - perl
 - regex
---

<em>How do I delete the first N lines from a file that match
a pattern?</em>

Some sample data to work on.

```bash
$ cat lines
line1
line2 foo
line3
line4 foo
line5 foo
line6
line7 moo
```

For example let's say `N = 2` that means we should remove
lines `2` and `4` from our sample data.

## perl

According to *the "internet"* it's *"so unreadable 
and cryptic that people look at code they've previously 
written and have no idea what it does anymore!!! It's also
ded!!! LOL!!!"*

But anyways, this seems like a simple task for `perl`

```bash
$ perl -ne 'print unless /foo/ and $count++ < 2' lines
line1
line3
line5 foo
line6
line7 moo
```

We can use `-ni -e` to overwrite the original file instead
of printing to stdout.

```bash
$ perl -ni -e 'print unless /foo/ and $count++ < 2' lines
$ cat lines
line1
line3
line5 foo
line6
line7 moo
```

## awk

Another possible solution is using `awk`. We can use the `next`
command to skip to the next *"line"* <small>(note: actually *record*
but the default *record* separator is `\n`)</small>

```bash
$ awk '/foo/ && count++ < 2 { next } { print }' lines
line1
line3
line5 foo
line6
line7 moo
```

GNU `awk` 4.1.0 added `-i inplace` which emulates Perl's
`-i` *"switch"* although one could just use shell 
redirection and `mv` to get the same end result e.g.

```bash
awk '...' lines > tmpfile && mv tmpfile lines
```

## bash

It is also possible to do it in *plain bash* using a `while read` loop combined with 
`[[` and `=~` for regex testing. We can use `continue` to skip to the next loop iteration.

```bash
$ while IFS= read -r line; do [[ $line =~ 'foo' && count++ -lt 2 ]] && continue; echo "$line"; done < lines
line1
line3
line5 foo
line6
line7 moo
```

Yes, `foo` is just a plain string and we could use `$line = *foo*` instead
but we're assuming that *pattern* means a regular expression.

You would also need to add redirection to a temporary file
and `mv` to overwrite the file as shown in the `awk` examples.
