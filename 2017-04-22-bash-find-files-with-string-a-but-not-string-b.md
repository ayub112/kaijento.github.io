---
title: "bash: find files that contain string A but not string B"
date:  2017-04-22 08:06:06 +0100
tags:
 - bash
 - find
 - grep 
 - regex
 - awk
 - glob
 - linux
---

<em>How do I find all files that contain string `A` but do
NOT contain string `B` on linux using bash?</em>

The original question asked about *"strings"* but the tools
we'll be using support regular expressions so we will essentially
be answering:

<em>How do I find all files that match pattern `A` but don't
match pattern `B`?</em>

If you'd just like a possible solution:

```bash
find . -type f -exec grep -q foo {} \; -not -exec grep -q bar {} \; -print
```

* `.` here refers to the current directory you can pass multiple directories
to start the search from if you wish e.g. `find /tmp /sys`

* GNU `find` allows you to omit any start directories and will default to `.`

There are many ways to solve this problem, what follows is a discussion
on the above example along with some others.

Create some test files.

```bash
$ ls -l
total 16K
-rw------- 1 user group  4 Apr 22 08:01 a
-rw------- 1 user group  7 Apr 22 08:01 b
-rw------- 1 user group  4 Apr 22 08:01 c
-rw------- 1 user group 24 Apr 22 08:01 d
-rw------- 1 user group 24 Apr 22 08:01 e
```

Their contents.

```bash
$ cat a
foo
$ cat b
foobar
$ cat c 
foo
$ cat d
blahblahfoo
barblahblah
$ cat e 
lol
```

So we want to find all files that contain `foo` but at the same time
do not contain `bar` which would be files `a` and `c` in our sample data.

We can use `grep` to find lines that match a pattern. We can use the
`-q` option which will *Exit immediately with zero status if any match is found*.

Combined with the shell's `&&` and `||` constructs we could say something
like  `grep -q foo && grep -q bar || print filename`

`command1 && command2` will only execute `command2` if `command1` exits
*successfully* <small>(i.e. an exit code of `0`)</small> whereas 
`command1 || command2` will only execute it if `command1` exits with a
*non-zero* status.

```bash
$ for file in ./*; do grep -q foo "$file" && grep -q bar "$file" || echo "$file"; done
./a
./c
./e
```

Is that an `./e` I see before me?! WAT?!

The problem is if the first `grep` fails i.e. when the file does not contain `foo`
it's executing the `||`. We need the `grep -q bar || echo` treated as a single *"group"*.

One option is to use `()` to group them together in their own explicit subshell.

```bash
$ for file in ./*; do grep -q foo "$file" && ( grep -q bar "$file" || echo "$file" ) done
./a
./c
```

We could have also used `{}` which is for grouping commands. Do note though that `}` needs
the `;` before it (or a newline).

```bash
$ for file in ./*; do grep -q foo "$file" && { grep -q bar "$file" || echo "$file"; } done
./a
./c
```

## * vs ./*

So you will have probably seen `for file in *` before but why are we using `./*` here instead?

```bash
$ echo *
a b c d e
$ echo ./*
./a ./b ./c ./d ./e
```

It's because of filenames with a leading dash `-`

```bash
$ touch ./-HELLO
$ for file in *; do ls "$file"; done
ls: invalid option -- 'E'
Try 'ls --help' for more information.
```

Let's use `set -x` to enable some *"debugging output"* <small>(use `set +x`
to disable it)</small>

```bash
$ set -x
$ for file in *; do ls "$file"; done
+ for file in '*'
+ ls -HELLO
ls: invalid option -- 'E'
Try 'ls --help' for more information.
```

So because of the `-` each letter of the filename is being interpreted
as a command-line option. If we give a full path to the file e.g. `./-HELLO`
this will not be an issue. Another option is to use `--` to indicate the
end of command-line options 

```bash
$ for file in *; do ls -- "$file"; done
-HELLO
```

Also note that *"hidden"* entries are not matched by default by `*`. We 
would need to use `shopt -s dotglob` to allow `*` to match names with a 
leading `.` if desired.

## search recursively

What happens if we need a *"recursive"* solution? Well `bash` 4 brought us
`globstar` which allows for *recursive globbing*. It needs to be enabled
using `shopt -s`

```bash
$ shopt -s globstar
$ mkdir -p h/i/j/k/l/
$ touch h/i/j/k/l/m
$ echo ./**/
./ ./f/ ./h/ ./h/i/ ./h/i/j/ ./h/i/j/k/ ./h/i/j/k/l/
$ echo ./**/*
./a ./b ./c ./d ./e ./f ./f/g ./h ./h/i ./h/i/j ./h/i/j/k ./h/i/j/k/l ./h/i/j/k/l/m
```

With globstar enabled `**/` will match all directories and subdirectories <small>(the `./`
is here due to the *leading dash* issue mentioned earlier)</small>. We then have the
extra `*` on the end to match the contents of each directory and subdirectory. <small>
(possibly not all contents though. see `dotglob`)</small>

Let's try it with our greps.

```bash
$ mkdir f
$ echo foo > f/g
$ shopt -s globstar
$ for file in ./**/*; do grep -q foo "$file" && ( grep -q bar "$file" || echo "$file" ) done
./a
./c
grep: ./f: Is a directory
./f/g
```

`grep` spits out some warnings though because we ran it on `./f` which is not a file.
This is just a warning and doesn't prevent the command from working so we could just
discard it by using `2>/dev/null` on the first `grep` command.

Another option is to use bash's
[Conditional Expressions](https://www.gnu.org/software/bash/manual/html_node/Bash-Conditional-Expressions.html#Bash-Conditional-Expressions)
to filter out only *"regular"* files. e.g. 

```bash
$ for file in ./**/*; do [[ -f $file ]] && grep ... && ( grep ... || echo ) done
```

At this stage though we're basically emulating `find` in `bash` when
we could just use it directly instead.

## find

This is the command given at the beginning of the article.

```bash
$ find -type f -exec grep -q foo {} \; -not -exec grep -q bar {} \; -print
./a
./c
./f/g
```

There is an implicit `-and` between `find` *"expressions"* so if we took
the last two expressions we would have

`-exec grep -q bar {} \; -and -print`

This would print the filename if the `grep -q bar` was successful. We want the 
opposite behaviour though which is why `-not` is there.

> `-not expression`
> This is the unary NOT operator.  It evaluates to true if the expression is false.

So with `-not -exec grep ... -print` we only print the filename if the second
`grep` failed i.e. it did not contain `bar`. Due to the implicit ands the previous 
expressions must have all succeeded to reach `-not` i.e. it must be `-type f` and
the first `-exec grep` matched.

So this works and to me is the *"simplest"* solution. It does however call `grep`
at least once (and possibly twice) for every file. Can we be more *"efficient"*
or as they say on the internet: 

*"CAN WE GO FASTAAR LULZ?!!1"*

## grep -l

`grep` has `-l` which will print out a list of filenames that match. GNU `grep`
has `-r` <small>(and `-R`)</small> to search recursively.

```bash
$ grep -r -l foo
a
b
c
d
f/g
```

It also has `-Z` which changes the output of `-l`. By default `-l` will print 1
filename per line but if you have a filename with a newline in it this will
prevent you from treated each line as a filename.

So if for example you were trying to process the output with a `while read` loop
it would break. I've never had a filename with a newline in it but this seems
to be the standard example used for how such approaches can be broken and you
should be aware of it as it is a possibility.

Anyways, `-Z` outputs a null-delimited list instead.

```bash
$ grep -r -l -Z foo | cat -vet
a^@b^@c^@d^@
```

We're using `cat -vet` here to show the *non-printing* characters and `^@` is a 
visual representation of the null-byte.

We could then pipe this to `while read -d ''` which sets read's `DELIM` to null to
process the results.

```
$ grep -r -l -Z foo | while read -d '' -r file; do grep -q bar "$file" || echo "$file"; done
a
c
f/g
```

Is this more efficient than the previous `find` solution? Maybe? I don't know. 

## awk

Back to `find` again we can use `awk` to process a file once instead of using 2 `grep` calls.

```bash
$ find -type f -exec awk '/foo/{ x = 1 } /bar/{ y = 1; exit } END { if (x && !y) print FILENAME }' {} \;
./a
./c
./f/g
```

* `/foo/` matches lines that contain `foo`
* `/bar/` matches lines that contain `bar` 
* `/regex/` matches any lines that *"matches"* `regex` it is shorthand for `$0 ~ /regex/`

In `awk` `0` is considered *"False"* and non-zero is considered *"True"*.

```bash
$ awk 'BEGIN { if (0) print "True" }'
$ awk 'BEGIN { if (1) print "True" }'
True
```

Or we could use the *ternary operator* for perhaps more explicit output.

```bash
$ awk 'BEGIN { print 0 ? "True" : "False" }'
False
$ awk 'BEGIN { print 1 ? "True" : "False" }'
True
```

So we're using `x` and `y` as *"flag variables"*. Then inside `END` we test if
`foo` is found and if `bar` is not found print `FILENAME`. The `END` block is
executed after all the input has been processed whereas `BEGIN` block executes
before any input has been processed.

You may look at this and say why don't we just `exit` when `/bar/` matches?
Well the reason for this is that after `exit` is called `END` blocks are
still executed so we need to set our flag variable or we'd get false results.

## -exec \; vs +

Is this faster than using 2 `grep` commands? I'm not sure but I'd hazard a guess
at "probably". 

`-exec command {} \;` will execute `command` once per *"file*" found e.g.

* `command a`
* `command b`
* `command c`

`-exec command {} +` will pass as many *"filenames"* as possible per execution e.g.

* `command a b c`

The reason we don't use the `+` form with our `grep` version is because *"all"* the 
*"filenames*" are passed to a single `grep` command and `foo` is found in one of them 
meaning the whole chain becomes *"True"* giving us all files as a match, which is not 
the behaviour we want.

```bash
$ find -type f -exec grep -q foo {} + -print
./a
./b
./c
./d
./e
./f/g
```

When we say *"all"* *"filenames"* it's true for this small example. There is a length limit
on the size of the command-line `ARG_MAX` which `find` works out and calls `-exec`
with as many *"filenames"* as possible each time, similar to how `xargs` works.

The problem still stands however but one possibility is to use the `+` form of `-exec` 
to execute our `grep -l` from earlier to build the list of matching files.

```bash
$ find -type f -exec grep -l -Z foo {} + | while read -d '' -r file; do grep -q bar "$file" || echo "$file"; done
./a
./c
./f/g
```

Is this more efficient? I'm not sure, it would be interesting to compare though.

## BEGINFILE

So finally back to `awk` again.

`awk` can accept multiple filenames but as with the `grep` example having all files 
treated as *"one"* will give us false results. 

GNU `awk` however has both `BEGINFILE` and `ENDFILE` blocks. This means we can have 
multiple files in a single `awk` execution but still treat each one *"separately"*.

It also has `nextfile` to move to the next file which can replace our `exit` call.

```bash
$ find -type f -exec awk 'BEGINFILE { x = y = 0 } /foo/{ x = 1 } /bar/{ y = 1; nextfile } ENDFILE { if (x && !y) print FILENAME }' {} +
./a
./c
./f/g
```

So we replace `exit` with `nextfile` and `END` with `ENDFILE`. 

We need to add a `BEGINFILE` block though to reset our flag variables back to their 
default state for each new file we process. 

I'd imagine this would be the *"most efficient"* of the approaches discussed 
but you would need to benchmark them to get a definite answer.
