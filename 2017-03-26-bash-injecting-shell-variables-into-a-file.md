---
title: "bash: injecting shell variables into a file"
date:  2017-03-26 08:52:24 +0100
tags:
  - bash
  - perl
  - sed
  - awk
---

Given an alpine linux docker container image we need to replace "tokens" 
or "placeholders" in a javascript file with values from shell variables 
e.g.  `%TESTAPP_FOO%` gets replaced with the value of the shell variable 
`TESTAPP_FOO`

I suppose you could consider them as a "template file":

`template`

```
Bob%TESTAPP_FOO%%TESTAPP_BAR%%TESTAPP_BAZ%qux
```

So given `TESTAPP_FOO=foo TESTAPP_BAR=bar TESTAPP_BAZ=baz` the desired output is:

    Bobfoobarbazqux

## perl

So according to the internet `perl` is "dead" and "unreadable" which I guess means the rest
of this article doesn't exist:

```bash
$ TESTAPP_FOO=foo TESTAPP_BAR=bar TESTAPP_BAZ=baz perl -pe 's/%(TESTAPP_[^%]+)%/$ENV{$1}/g' template
Bobfoobarbazqux
```

With perl's `-i` option it will overwrite the original file.

`-i` takes an optional argument so we must use `-pi -e` and not `-pie`

What happens if we put the variable definitions onto their own line:

```bash
$ TESTAPP_FOO=foo TESTAPP_BAR=bar TESTAPP_BAZ=baz 
$ perl -pe 's/%(TESTAPP_[^%]+)%/$ENV{$1}/g' template
Bobqux
```

`var=value command` is like using `( export var=value; command )` in that it exports those
variable definitions only for that command. They are not defined in the current shell. Without
the `export` they are not visible from the `perl` command.

`%(TESTAPP_` matches `%TESTAPP_` with the `(` starting a capture group. Capture groups allow
you to refer to what was captured. `$1` to refers to the first capture group, `$2` the second, etc.

If you want to match up to a character but you want to "wildcard" what can come before it
you can use `[^x]+x` with `x` being the character. The `+` (meaning <em>"1 or more times"</em>) 
states there must be at least 1 character before `x`. If we wanted to remove this 
stipulation we can replace `+` with `*` as `*` means <em>"0 or more"</em>.

So as we have already matched `%TESTAPP_` the `[^%]+%` part of the regex (ignoring the `)`) 
matches up to the following `%` character.  Our use of `()` means the leading and 
trailing `%` will not be included in `$1`.

```bash
$ perl -nle 'print for /%(TESTAPP_[^%]+)%/g' testfile
TESTAPP_FOO
TESTAPP_BAR
TESTAPP_BAZ
```

Finally `%ENV` is a perl hash that contains all of the environment's variable definitions.
We can use `$ENV{key}` to get a specific value from it:

```bash
$ TESTAPP_FOO=foo perl -le 'print $ENV{TESTAPP_FOO}'
foo
```

So with `$ENV{$1}` we are using the value of the capture group as the key to get the desired result.

Sadly, however, `perl` was not available by default on this installation setup.

## grep + sed

GNU `grep` has `-o` which allows us to replicate the `perl` code above to an extent:

```bash
$ grep -E -o '%TESTAPP_[^%]+%' testfile 
%TESTAPP_FOO%
%TESTAPP_BAR% 
%TESTAPP_BAZ%
```

We can then use `bash` to help with the rest:

```bash
$ TESTAPP_FOO=foo TESTAPP_BAR=bar TESTAPP_BAZ=baz
$ grep -E -o '%TESTAPP_[^%]+%' testfile | while read -r search; do replace=${search//%}; echo "$search ${!replace}"; done
%TESTAPP_FOO% foo
%TESTAPP_BAR% bar
%TESTAPP_BAZ% baz
```

* `${var/pattern/replace}`  is like `s/pattern/replace/`
* `${var//pattern/replace}` is like `s/pattern/replace/g`

With an empty "replace" you can omit the `/` after "pattern" meaning:

`${search//%}` is like `s/%//g` thus deleting all `%` characters.

## ${!var} 

If the value of your variable is the name of a variable you can use `${!var}` to get its
value. This is called variable <em>"indirection"</em>.

```bash
$ var=SHELL
$ echo ${!var}
/bin/bash
```

It's as if we just did `echo $SHELL` directly.

If your `grep` has `-P` (chances are it does if you have `-o`) we can simplify things:

```bash
$ grep -P -o '%\KTESTAPP_[^%]+(?=%)' testfile | 
    while read -r search; do echo "%$search% ${!search}"; done
%TESTAPP_FOO% foo
%TESTAPP_BAR% bar
%TESTAPP_BAZ% baz
```

We have a list of search and replacement strings so we could just inject them into `sed`

```bash
$ cat testfile
Bob%TESTAPP_FOO%%TESTAPP_BAR%%TESTAPP_BAZ%qux
$ grep -P -o '%\KTESTAPP_[^%]+(?=%)' testfile | 
    while read -r search; do sed -i "s/%${search}%/${!search}/g" testfile; done
$ cat testfile
Bobfoobarbazqux
```

Certain versions of `sed` got `-i` from `perl` which outputs to a temporary file and then overwrites the
original file.

So this works for our example however there are some issues. Firstly it processes the input file multiple
times. Second, injecting variables into sed commands can break under certain conditions.
One is if the contents of your variables contains the delimiter i.e. `/` and another is if it contains newlines.
You can change the delimiter e.g. `s@foo@bar@` however you would have to change it to something you know isn't
in your variable which may not be possible.

## awk

`awk` has the `ENVIRON` array which can access variables as `%ENV` can in `perl`. Arrays in awk are
"associative arrays" which are the same as a "hash" in `perl` or "dict" in `python`.

```bash
$ TESTAPP_FOO=foo awk 'BEGIN { print ENVIRON["TESTAPP_FOO"] }'
foo
```

We'll just `export` the variables to save having a gigantic 1-liner:

```bash
$ cat testfile
Bob%TESTAPP_FOO%%TESTAPP_BAR%%TESTAPP_BAZ%qux
$ export TESTAPP_FOO=foo TESTAPP_BAR=bar TESTAPP_BAZ=baz
$ awk '{ 
    while (match($0, /%TESTAPP_[^%]+%/)) { 
        search = substr($0, RSTART + 1, RLENGTH - 2)
        $0 = substr($0, 1, RSTART - 1)   \
             ENVIRON[search]             \
             substr($0, RSTART + RLENGTH) 
    } 
    print 
}' testfile
Bobfoobarbazqux
```

This doesn't modify the original file however GNU `awk` allows you to use `-i inplace` which 
overwrites the original file. Without this you could use shell redirection to a temporary file
and `mv` to overwrite the original e.g.

```bash
$ awk '...' testfile > tempfile && mv tempfile testfile
```

You could use `mktemp` to generate a temporary file for you instead of hardcoding in `tempfile`
as we did in this example.

When you use `match()` in `awk` it sets the `RSTART` and `RLENGTH` variables. `RSTART` being the
index of where the match starts in the string and `RLENGTH` being the length of the match. `$0` 
refers to the whole <em>"line"</em>. 

Using these variables combined with `substr()` allows us to extract what comes before the match 
(to the "left") and what comes after the match (to the "right"). We then just rebuild the line 
by inserting our replacement value in the middle e.g.

```awk
left  = substr($0, 1, RSTART - 1)
right = substr($0, RSTART + RLENGTH)
$0    = left ENVIRON[search] right
``` 

`awk` will implicitly join the strings for us. This is why we have a trailing backslash on lines
3 and 4 -  to have it treated as a "single line" e.g.

```awk
$0 = substr($0, 1, RSTART - 1) ENVIRON[search] substr($0, RSTART + RLENGTH)
```

Otherwise it would parse as:

```awk
$0 = substr($0, 1, RSTART - 1)
ENVIRON[search]
substr($0, RSTART + RLENGTH)
```

Which would just be a single assignment `$0 = substr($0, 1, RSTART - 1)` and that would be <em>"Less Than Awesome"</em>.

## That's it!

Well, not entirely. It was never mentioned what to do if a token string corresponded to a shell variable
that was not defined or if that was possible, it was just assumed they would all exist.

If you wanted to only replace if the variable was defined you could make some adjustments:

```bash
$ export TESTAPP_FOO=foo TESTAPP_BAR=bar TESTAPP_BAZ=baz
$ echo %TESTAPP_DONTREPLACEME% | perl -pe 's/%(TESTAPP_[^%]+)%/$ENV{$1}/ge'

$ echo %TESTAPP_DONTREPLACEME% | perl -pe 's/%(TESTAPP_[^%]+)%/exists $ENV{$1} ? $ENV{$1} : "%$1%"/ge'
%TESTAPP_DONTREPLACEME%
```

So the `exists` call checks if the key is in the hash. If it is give us the value, else give us
the key back with the surrounding `%`. The `e` modifier of the `s` command evalutes the
right hand side as perl code which allows this to work.

In `awk` you can use `in` to check if a key is in an array.

```bash
$ echo %TESTAPP_FOO% %TESTAPP_DONTREPLACEME% | awk '{ 
    while (match($0, /%TESTAPP_[^%]+%/) && substr($0, RSTART + 1, RLENGTH - 2) in ENVIRON) { 
        search = substr($0, RSTART + 1, RLENGTH - 2)
        $0 = substr($0, 1, RSTART - 1)    \
             ENVIRON[search]              \
             substr($0, RSTART + RLENGTH) 
    } 
    print 
}'
foo %TESTAPP_DONTREPLACEME%
```

As we are using `while(match())` if we don't replace a token we will have an infinite loop 
because `%TOKEN%` will still be in our string. One way to avoid this is by adding another
check to the loop condition to test if the token corresponds to a defined variable in
the `ENVIRON` array.

As for the `grep` and `sed` approach it is still possible. We can use `[[ -v varname ]]` to 
test if a variable is set. As the name of the variable we want is contained in `search` we 
can use `[[ -v $search ]]` to test correctly:

```bash
$ cat testfile
%TESTAPP_DONTREPLACEME% %TESTAPP_FOO%
$ grep -P -o '%\KTESTAPP_[^%]+(?=%)' testfile | 
    while read -r search; do [[ -v $search ]] && sed -i "s/%${search}%/${!search}/g" testfile; done
$ cat testfile
%TESTAPP_DONTREPLACEME% foo
```

However in an ideal situation `perl` would be available and you could use that solution.
