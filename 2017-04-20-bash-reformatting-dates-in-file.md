---
title: "bash: reformatting dates in a logfile"
date:  2017-04-20 12:38:23 +0100
tags:
 - bash
 - perl
 - date
---

<em>I have a file where each line starts with a date string: 
dayname monthname day year. I need to reformat it into Y-m-d format. 
Pls Halp.</em>

The sample input we're given is:

```
Sat Aug 2 2015, value, value, value
Mon Aug 4 2015, value, value, value
Wed Aug 6 2015, value, value, value
```

We can use the `date` command to reformat date strings. GNU `date`
allows us to specify a date via the `-d` option and we can supply
a format string for the output date using `+FORMAT`

`%F` is equivalent to `%Y-%m-%d`

You can see a full list of the format strings 
[in the manual](https://www.gnu.org/software/coreutils/manual/html_node/date-invocation.html#date-invocation).

```bash
$ declare -p date
declare -- date="Sat Aug 2 2015"
$ date -d "$date" +%F
2015-08-02
```

If you do not have access to GNU `date` e.g. you're on OSX 
(or MacOS or whatever it's called these days) chances are 
you have BSD `date` which has the `-f` option for reading in dates of 
a certain format.

```bash
$ date -j -f "%a %b %d %Y" "$date" +%F
2015-08-02
```

We can combine the use of `date` with a `while read` loop to read the
input file line by line.

```bash
$ cat logfile.log
Sat Aug 2 2015, value, value, value
Mon Aug 4 2015, value, value, value
Wed Aug 6 2015, value, value, value
$ while IFS=, read -r date rest; do date=`date -d "$date" +%F`; echo "$date,$rest"; done < logfile.log
2015-08-02, value, value, value
2015-08-04, value, value, value
2015-08-06, value, value, value
```

We're setting `IFS=,` to allow read to separate *"words"* by a comma. 

```bash
$ IFS=, read -r date rest <<< 'Sat Aug 2 2015, value, value, value'
$ declare -p date rest
declare -- date="Sat Aug 2 2015"
declare -- rest=" value, value, value"
```

As we only supply 2 variable names to `read` it automatically fills the
second one with the *"rest"* of the data on the line. 

If we added another variable we would get the first *"value"* 

```bash
$ IFS=, read -r date first_value rest <<< 'Sat Aug 2 2015, value-a, value-b, value-c'
$ declare -p date first_value rest
declare -- date="Sat Aug 2 2015"
declare -- first_value=" value-a"
declare -- rest=" value-b, value-c"
```

This approach does not modify the input file. To do so we could redirect the output
of the `while` loop to a temporary file and then use `mv` to overwrite the original
file. 

This is somewhat messy and if the file is in anyway large executing `date`
for each line will probably not be the quickest thing in the world.

## perl

Another option is to instead use `Perl` with its `Time::Piece` module for parsing / reformatting 
dates.

```bash
$ perl -MTime::Piece -pe 's/([^,]+)/Time::Piece->strptime($1, "%a %b %d %Y")->strftime("%Y-%m-%d")/e' logfile.log
2015-08-02, value, value, value
2015-08-04, value, value, value
2015-08-06, value, value, value
```

This does not modify the input file.

```bash
$ cat logfile.log
Sat Aug 2 2015, value, value, value
Mon Aug 4 2015, value, value, value
Wed Aug 6 2015, value, value, value
```

To do so we can use the `-i` *"switch"*. The `-i` *"switch"* takes an optional argument though:
a SUFFIX if you want it to create a backup file for you. This means that we cannot
cannot write `-pie` but must use `-pi -e` instead.

```bash
$ perl -MTime::Piece -pi -e 's/([^,]+)/Time::Piece->strptime($1, "%a %b %d %Y")->strftime("%Y-%m-%d")/e' logfile.log
$ cat logfile.log
2015-08-02, value, value, value
2015-08-04, value, value, value
2015-08-06, value, value, value
```

Because we know that each line starts with the date string, matching up to but not including
the first comma on the line will match only the date.

* `([^,]+)` matches and captures a sequence of 1 or more characters that is not `,` i.e. the date
* the `e` modifer on the `s/pattern/replacement/` command evaluates the replacement as Perl code

So on the first line of our logfile `Time::Piece->strptime($1, "%a %b %d %Y")` turns into
`Time::Piece->strptime("Sat Aug 2 2015", "%a %b %d %Y")` as `$1` is the content of the first
capture group i.e. the date.

This creates a `Time::Piece` object from our parsed date which we then reformat using the 
`strftime()` method giving us the desired output format.

The `-M` switch is for loading modules it's like doing a `use` inside your Perl code.

So `perl -pe 's///'` is essentially a replacement for `sed`

```bash
$ perl -MO=Deparse -pe 's/foo/bar/'
LINE: while (defined($_ = <ARGV>)) {
    s/foo/bar/;
}
continue {
    die "-p destination: $!\n" unless print $_;
}
-e syntax OK
```

* `-p` adds a while read line loop with an implicit `print`
* `-n` does the same without the `print`

If you find sed's Basic or Extended Regular Expressions annoying this is a
good replacement. It also gives you access to multiline searching / replacing
with the `-0777` switch.

You can view the documentation for the switches using the `perldoc perlrun` command or
you can try viewing it on [metacpan.org](https://metacpan.org/pod/perlrun).
