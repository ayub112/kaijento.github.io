---
title: "split a string without split()"
date:  2017-04-18 19:41:51 +0100
tags:
 - java
 - perl
 - grep
 - php
 - python
 - regex
---

Given the input string `[a]text here[/a][b]more text here[/b]` the 
goal is to split it up into parts

* `[a]`
* `text here`
* `[/a]`
* `[b]`
* `more text here`
* `[/b]`

It seems to be common enough that people think of `split()` type
functions when faced with this kind of task. Normally with such
functions what you split on (the <em>delimiter</em>) is removed
and not present in the output.

What you can do however is to <em>"match"</em> the parts that you
do want. The sample string we've been given seems to be some *"tag-based"* 
data and they were trying to do this using Java.

`MatcherExample.java`

```java
import java.util.*; 
import java.util.regex.*;

class MatcherExample {

    public static void main (String[] args) {

        String    input   = "[a]text here[/a][b]more text here[/b]";
        Pattern   pattern = Pattern.compile("\\[[^]]+]|.+?(?=\\[[^]]+])");

        ArrayList matches = new ArrayList();
        Matcher   matcher = pattern.matcher(input);

        while (matcher.find()) {
            matches.add(matcher.group());
            System.out.println(matcher.group());
        }
    }

}
```

So the goal was to end up with an `ArrayList` containing all the
individual parts. To do this we're using regex. The actual regex
we're using is `\[[^]]+]|.+?(?=\[[^]]+])` the backslashes need to
be escaped because we're inside double quotes i.e. a *String*.

* `\[` matches a literal [
* `[^]]+` matches a sequence of *1 or more* characters that are not `]`
* `|` means *"or"*
* `.+?` matches a sequence of *1 or more* characters (non-greedy)
* `(?=)` is a *positive lookahead assertion*. It's like saying *"what's contained
in here must follow next"*. It does not consume (or capture) any characters.

What we have contained in the assertion is the same as the first part of our
pattern which matches the tag.

So this means we match a *"tag"* or *"anything" that is followed by a "tag"*.

Another pattern which works for the sample input is `[^][]+|\[[^]]+]` which
would match a sequence of *1 or more* characters that are neither `]` nor `[`
i.e. stuff outside a "tag" or else match a "tag"

This assumes data not inside a tag cannot contain `[` or `]` but the first 
pattern will break if the *"outside"* data contains a `[`. 

This is why parsing *"tag-based"* data with regular expressions is somewhat 
discouraged: *"it can get painful fast"* <small>(depending on your 
requirements)</small>

So to get all the matches we must create a `Matcher` object and loop over
the results of the `find()` method. We're using `group()` to get the 
*"whole match"*. You can use `group(n)` to get the *n'th* capture group.

```bash
$ java MatcherExample
[a]
text here
[/a]
[b]
more text here
[/b]
```

## php

This is a common enough question that we'll show some examples in other languages.

First we have `preg_match_all()` for `PHP`

```bash
$ php -r 'preg_match_all("/\[[^]]+]|.+?(?=\[[^]]+])/", 
    "[a]text here[/a][b]more text here[/b]", $matches); var_export($matches);'
array (
  0 =>
  array (
    0 => '[a]',
    1 => 'text here',
    2 => '[/a]',
    3 => '[b]',
    4 => 'more text here',
    5 => '[/b]',
  ),
)
```

## perl

When needing to *"find all matches of a pattern"* on the command line `perl` 
is a common choice.

```bash
$ echo '[a]text here[/a][b]more text here[/b]' | perl -nle 'print for /\[[^]]+]|.+?(?=\[[^]]+])/g'
[a]
text here
[/a]
[b]
more text here
[/b]
```

It can read all the data in at once <small>(*"slurp*")</small> using `-0777` 
which allows to perform multiline regex operations.

## grep

If your `grep` has `-P` and `-o` like GNU `grep` for does you can replicate
the above `perl` command using:

```bash
$ echo '[a]text here[/a][b]more text here[/b]' | grep -P -o '\[[^]]+]|.+?(?=\[[^]]+])'
[a]
text here
[/a]
[b]
more text here
[/b]
```

## python

Usually though, I end up using an interactive `python` session for debugging regex.

```python
>>> import re
>>> 
>>> data = '[a]text here[/a][b]more text here[/b]'
>>> re.findall(r'\[[^]]+]|.+?(?=\[[^]]+])', data)
['[a]', 'text here', '[/a]', '[b]', 'more text here', '[/b]']
```

## That's it!

So there you have it. When wanting to *split* up a string using `split()` may not always be 
what you're looking for. Similarly when wanting to remove certain parts of a string it
can sometimes be simpler to instead match / extract the parts that you want to keep.
