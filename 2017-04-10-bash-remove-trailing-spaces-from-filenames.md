---
title: "bash: remove trailing spaces from filenames"
date:  2017-04-10 04:51:22 +0100
tags:
 - bash
 - find
 - perl
 - regex
 - glob
 - rename
 - linux
---

<em>I have a directory full of files that contain trailing spaces
which is causing problems when trying to read them on Windows.
I am booted into Linux and have a bash shell how do I remove the
trailing spaces?</em>

If you're just looking for possible solutions:

`extglob` <small>note: non-recursive</small>

```bash
shopt -s dotglob extglob nullglob
for old in ./*' '; do mv -v "$old" "${old%%+( )}"; done
```

`globstar + extglob` <small>note: requires bash 4</small>

```bash
shopt -s dotglob extglob globstar nullglob
for old in ./**/*' '; do mv -v "$old" "${old%%+( )}"; done
```

`find + extglob`

```bash
find . -name '* ' -exec bash -c 'shopt -s extglob; mv -v "$1" "${1%%+( )}"' _ {} \;
```

`find + prename`

```bash
find . -name '* ' -exec rename -v 's/ +$//' {} +
```

It is **highly recommended** to have a backup of your data and work on 
a copy in the event of anything going wrong.

Please be aware that if multiple filenames result in the same name
after trailing space removal they will be overwritten. You may wish to 
use the `-n` option with `mv` and/or `-b / --backup` if your `mv` 
supports it.

The remainder of this article is an explanation of the possible solutions 
that were suggested.

## Test files

We've created some test files to work with. We use `ls` with the `-Q` 
option to quote the filenames which allows us to easily see the trailing 
spaces characters:

```bash
$ ls -RQ
".":
"a"  "b"  "bbq    "  "c"  "lol "  "omg "  "subdir"

"./subdir":
"filename        "
```

Another option could be to use the `-ls` option to `find`:

```bash
$ find -ls
8665642    0 drwx------   3 user group      111 Apr 10 04:49 .
8665643    0 -rw-------   1 user group        0 Apr 10 04:48 ./lol\ 
8665644    0 -rw-------   1 user group        0 Apr 10 04:48 ./omg\ 
8665645    0 -rw-------   1 user group        0 Apr 10 04:48 ./a
8665646    0 -rw-------   1 user group        0 Apr 10 04:48 ./b
8665647    0 -rw-------   1 user group        0 Apr 10 04:48 ./c
8665651    0 -rw-------   1 user group        0 Apr 10 04:48 ./bbq\ \ \ \ 
4472706    0 drwx------   2 user group       54 Apr 10 04:49 ./subdir
4595526    0 -rw-------   1 user group        0 Apr 10 04:49 ./subdir/filename\ \ \ \ \ \ \ \ 
```

<small>note: GNU `find` will default to the current directory `.` 
if you do not give any directory names to process</small>

It was not stated in the problem description whether or not a recursive solution
was required. A recursive approach will work for both cases so we will assume 
that is was.

The non-recursive solution that was suggested above is very similar to the
recursive solution using `globstar` which is explained below so it shouldn't 
need a separate explanation.

## prename

You may or may not have a `rename` command available on your system. There 
is a `rename` command that is written in Perl and one that is not. The 
suggested solution that was given will only work with the Perl version.

```bash
$ rename
Usage: rename [-v] [-n] [-f] perlexpr [filenames]
$ readlink -f /usr/bin/rename
/usr/bin/prename
$ prename
Usage: rename [-v] [-n] [-f] perlexpr [filenames]
```

If you see `perlexpr` in the usage output you should be good to go.

Its `-n` option will print out what will happen without acting:

```bash
$ find -name '* ' -exec rename -n 's/ +$//' {} +
./lol  renamed as ./lol
./omg  renamed as ./omg
./bbq     renamed as ./bbq
./subdir/filename         renamed as ./subdir/filename 
```

It should probably quote the filenames though similar to how `mv -v` does to make
viewing trailing spaces easier. We'll replace `-n` with `-v` to actually rename
the files:

```bash
$ find -name '* ' -exec rename -v 's/ +$//' {} +
./lol  renamed as ./lol
./omg  renamed as ./omg
./bbq     renamed as ./bbq
./subdir/filename         renamed as ./subdir/filename
```

Let's check the result:

```bash
$ ls -RQ
".":
"a"  "b"  "bbq"  "c"  "lol"  "omg"  "subdir"

"./subdir":
"filename"
$ find -ls
8665642    0 drwx------   3 user group      105 Apr 10 04:49 .
8665645    0 -rw-------   1 user group        0 Apr 10 04:48 ./a
8665646    0 -rw-------   1 user group        0 Apr 10 04:48 ./b
8665647    0 -rw-------   1 user group        0 Apr 10 04:48 ./c
4472706    0 drwx------   2 user group       46 Apr 10 04:49 ./subdir
4595526    0 -rw-------   1 user group        0 Apr 10 04:49 ./subdir/filename
8665643    0 -rw-------   1 user group        0 Apr 10 04:48 ./lol
8665644    0 -rw-------   1 user group        0 Apr 10 04:48 ./omg
8665651    0 -rw-------   1 user group        0 Apr 10 04:48 ./bbq
```

The `-name` option of `find` takes a <em>"glob pattern"</em>. If you're familiar
with regular expressions the glob pattern used is equivalent to the regex `.* $` 
i.e. <em>anything that ends with a space character</em>.

The `rename` command allows you to use any `perl` code but it's most common to see 
it used with the `s` command. `s/ +$//` matches <em>1 or
more space characters followed by the end of the string</em> and replaces
them with nothing i.e. removes them.

We could use `\s+$` to match any trailing "whitespace" as opposed to just space
characters specifically.

If you don't have the Perl `rename` command you can use some of bash's string
manipulation capabilities to achieve the same result.

## parameter expansion

We can use `${var%pattern}` to <em>"trim" (remove)</em> the given <em>pattern</em> 
from the end of shell variable <em>var</em>. If you use a double percent i.e. `%%` 
it will remove the longest match of <em>pattern</em> whereas the default is 
the shortest match.

So if it was just a single trailing space we could do the following:

```bash
$ name='omg lol '
$ echo "${name% }" | sed -n l
omg lol$
```

<small>note: we're using `sed -n l` here just to show the trailing spaces. `$`
denotes the end of the string</small>

With multiple trailing spaces this approach wont work:

```bash
$ name='omg lol    ' 
$ echo "${name% }" | sed -n l
omg lol   $
```

What if we try with `%%`?

```bash
$ echo "${name%% }" | sed -n l
omg lol   $
```

Our pattern is just a single space character which will only ever match a single
space character. What we need to do is remove the longest match of <em>1 or more
space characters</em>. 

## extglob

You can do this using the `+(pattern)` syntax which matches <em>1 or more</em>
instances of pattern. It's not part of the standard globbing syntax which means
that you need to enable <em>extended globbing</em> to use it:

```bash
$ shopt -s extglob
$ echo "${name%%+( )}" | sed -n l
omg lol$
```

This allows us to replace `-exec rename` with `-exec bash` and a `mv` command.

We'll use a leading `echo` to first see what the generated commands would look like:

```bash
$ find -name '* ' -exec bash -c 'shopt -s extglob; echo mv -v "$1" "${1%%+( )}"' _ {} \;
mv -v ./subdir/filename         ./subdir/filename
mv -v ./bbq     ./bbq
mv -v ./lol  ./lol
mv -v ./omg  ./omg
```

We remove the `echo` to actually execute the `mv` commands:

```bash
$ find -name '* ' -exec bash -c 'shopt -s extglob; mv -v "$1" "${1%%+( )}"' _ {} \;
‘./subdir/filename        ’ -> ‘./subdir/filename’
‘./bbq    ’ -> ‘./bbq’
‘./lol ’ -> ‘./lol’
‘./omg ’ -> ‘./omg’
```

Check the results:

```bash
$ ls -RQ
".":
"a"  "b"  "bbq"  "c"  "lol"  "omg"  "subdir"

"./subdir":
"filename"
```

The above `-exec` command will execute `bash` for each file found.

A common idiom you may see when execing a shell is: 

```bash
-exec bash -c 'for path; do ...; done' _ {} +
```

So you may see the above `-exec` command written like:

```bash
-exec bash -c '
  shopt -s extglob; for old; do mv -v "$old" "${old%%+( )}"; done' _ {} +
```

This approach could save some forking of bash processes but it's probably
not going to make too much of a difference.

## globstar

Version 4 of `bash` added the `globstar` shell option which allows <em>recursive globbing</em>.
Using this we can remove the need for `find`:

```bash
$ shopt -s globstar
$ for old in ./**/*' '; do echo "‘$old’"; done
‘./bbq    ’
‘./lol ’
‘./omg ’
‘./subdir/filename        ’
```

We still use the `*' '` pattern however before it we use `./**/` which matches 
all directories and subdirectories.

So let's combine the `extglob` and `globstar` operations:

```bash
$ shopt -s extglob globstar
$ for old in ./**/*' '; do echo mv -v "$old" "${old%%+( )}"; done
mv -v ./bbq     ./bbq
mv -v ./lol  ./lol
mv -v ./omg  ./omg
mv -v ./subdir/filename         ./subdir/filename
```

Output looks okay so we remove the `echo` once again to 
execute the `mv` commands:

```bash
$ shopt -s extglob globstar
$ for old in ./**/*' '; do mv -v "$old" "${old%%+( )}"; done
‘./bbq    ’ -> ‘./bbq’
‘./lol ’ -> ‘./lol’
‘./omg ’ -> ‘./omg’
‘./subdir/filename        ’ -> ‘./subdir/filename’
```

Check the result:

```bash
$ ls -RQ
".":
"a"  "b"  "bbq"  "c"  "lol"  "omg"  "subdir"

"./subdir":
"filename"
```

## nullglob + dotglob

The files have been renamed correctly. Let's now create a new <em>"hidden"</em> file
and run our loop again:

```bash
$ touch '.hidden file       '
$ for old in ./**/*' '; do echo mv -v "$old" "${old%%+( )}"; done
mv -v ./**/*  ./**/*
```

U wot m8? 

If a glob (well, technically it's a <em>pathname expansion</em> here) does not match 
the default behaviour is that the glob will be returned.

```bash
$ for f in ./*.nothere; do echo "found: $f"; done
found: ./*.nothere
```

To disable this we can enable `nullglob`:

```bash
$ shopt -s nullglob
$ for old in ./**/*' '; do echo mv -v "$old" "${old%%+( )}"; done
$
```

We created a new <em>"hidden"</em> file with trailing spaces though so why didn't 
it match? 

By default the leading `.` at the start of the name (or after a `/`) must
be explicitly matched. This behaviour can be changed with `dotglob`

```bash
$ shopt -s dotglob
$ for old in ./**/*' '; do echo mv -v "$old" "${old%%+( )}"; done
mv -v ./.hidden file        ./.hidden file
```

So without `dotglob` it would miss <em>hidden</em> files and it would also miss any
files contained in <em>hidden</em> directories.

You may or may not want this behaviour so you can include or exclude `dotglob` as
you see fit.
