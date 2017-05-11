---
title: "bash: rename files based on modification time"
date:  2017-04-12 07:17:57 +0100
tags:
 - bash
 - find
 - rename
 - perl
 - python
 - linux
---

<em>I have a directory full of files. I would like to prefix
each filename with an ascending count starting from the oldest.</em>

before

```bash
$ ls -l
total 0K
-rw------- 1 user group 0 Apr 12 05:44 a
-rw------- 1 user group 0 Apr 12 04:45 b
-rw------- 1 user group 0 Apr 12 03:45 c
```

after

```bash
$ ls -lQ
total 0K
-rw------- 1 user group 0 Apr 12 03:45 "01 c"
-rw------- 1 user group 0 Apr 12 04:45 "02 b"
-rw------- 1 user group 0 Apr 12 05:44 "03 a"
```

Generally when talking about the age of files on <em>"*nix"</em>
the last modification time or `mtime` is used as the presence of 
"creation time" is filesystem specific and may not exist.

We're using Linux here which means we have access to
GNU `ls` which has `-Q` to quote the filename. This 
can be useful for highlighting the filename portion of 
the ouput when using `-l` as shown above.

So let's create some test files. 

GNU `date` has `-d` which can simplify creating dates 
in the past (or future).

```bash
$ touch -t $(date +%m%d%H%M -d '1 hour ago') c
$ touch -t $(date +%m%d%H%M -d '2 hour ago') b
$ touch a
$ ls -l
total 0K
-rw------- 1 user group 0 Apr 12 07:31 a
-rw------- 1 user group 0 Apr 12 05:31 b
-rw------- 1 user group 0 Apr 12 06:31 c
```

The `$()` here is called <em>Command Substitution</em>.
It is used for capturing the output of a command. You may 
also see it written using backticks (<em>"backquotes"</em>) 
e.g. `` `date +%m%d%H%M -d '1 hour ago'` ``

The backtick form is considered <em>old-style</em> and there
are some slight differences with regards to backslash handling.

You will of course read on <em>"the internet"</em> that you are
<em>"never"</em> to use the old-style but some people find it 
<em>"prettier"</em> and easier to type.

## ls

`ls` has the `-t` option which will sort by modification time.

```bash
$ ls -lt 
total 0K
-rw------- 1 user group 0 Apr 12 07:31 a
-rw------- 1 user group 0 Apr 12 06:31 c
-rw------- 1 user group 0 Apr 12 05:31 b
```

With `-r` you can reverse the sort order. 

```bash
$ ls -ltr 
total 0K
-rw------- 1 user group 0 Apr 12 05:31 b
-rw------- 1 user group 0 Apr 12 06:31 c
-rw------- 1 user group 0 Apr 12 07:31 a
```

According to <em>"the internet"</em> you're <em>"never"</em>
supposed to parse `ls` but <em>"h8rz gonna h8"</em> as they say.

```bash
$ for f in `ls -tr`; do let i++; mv -v "$f" "$i $f"; done
‘b’ -> ‘1 b’
‘c’ -> ‘2 c’
‘a’ -> ‘3 a’
```

Check the results:

```bash
$ ls -lQ
total 0K
-rw------- 1 user group 0 Apr 12 05:31 "1 b"
-rw------- 1 user group 0 Apr 12 06:31 "2 c"
-rw------- 1 user group 0 Apr 12 07:31 "3 a"
```

## double dash

Instead of the filename `b` we will use `-b`

```bash
$ touch -b
touch: invalid option -- 'b'
Try 'touch --help' for more information.
```

The problem is that `-b` is being parsed by `touch` as an argument
due to the leading `-`. 

There are 2 options, we can can give the full path e.g. `./-b` or we can use `-- -b` 

> A  `--` signals the end of options and disables further option processing.  
> Any arguments after the `--` are treated as filenames and arguments.

```bash
$ ls -lt 
total 0K
-rw------- 1 user group 0 Apr 12 07:31 a
-rw------- 1 user group 0 Apr 12 06:31 c
-rw------- 1 user group 0 Apr 12 05:31 -b
```

Does our loop still work?

```bash
$ for f in `ls -tr`; do let i++; mv -v "$f" "$i $f"; done
‘c’ -> ‘1 c’
‘a’ -> ‘2 a’
mv: missing destination file operand after ‘3 -b’
Try 'mv --help' for more information.
```

So as with the `touch` example above `mv` is interpreting `-b` as an option
thus breaking the command. 

We could use add `./` in but we will choose the `--` option instead.

```bash
$ i=0; for f in `ls -tr`; do let i++; mv -v -- "$f" "$i $f"; done
‘-b’ -> ‘1 -b’
‘c’ -> ‘2 c’
‘a’ -> ‘3 a’
```

We're explicitly setting `i=0` here because the previous example was executed
in the same bash instance so `i` had a value of `1`. We will set `i=0` for
the remaining examples.

## math in bash

There are various ways to perform arithmetic in bash. 
Instead of `let i++` we could have used `((i++))`. There is also
the `$(())` form which will return the result meaning we could
inline the incrementation into the `dest` filename e.g. 
`mv -v -- "$f" "$((++i)) $f"`

With inlining we need to change the increment from postfix to prefix 
otherwise we would start at `0` e.g. `0 b`, `1 c` and `2 a`

We did not explicitly initialize `i` in the first example as it wasn't needed.

```bash
$ declare -p i
-bash: declare: i: not found
$ ((i++))
$ declare -p i
declare -- i="1"
```

## printf padding

The requirements stated that numbers 1-9 should be padded with a leading 0. We can use `printf`
to achieve that.

```bash
$ i=0; printf '%02d\n' $((++i))
01
```

bash's `printf` was given `-v` which allows you to store the output into a 
variable instead of printing it out.

```bash
$ i=0; printf -v new %02d $((++i))
$ declare -p new
declare -- new="01"
```

You could of course use <em>Command Substitution</em> if your version did not
support `-v`. Let's try again with `printf`:

```bash
$ i=0; for old in `ls -tr`; do 
    printf -v new '%02d %s' $((++i)) "$old"; mv -v -- "$old" "$new"; done
‘-b’ -> ‘01 -b’
‘c’  -> ‘02 c’
‘a’  -> ‘03 a’
```

## to parse or not to parse

<em>But OMG it sez on d internet nevar parse `ls` !!! lmao wut r u doin u t0t4l n00bz0r3!!!!</em>

The problem with parsing `ls` this way is that it cannot handle 
all possible filenames, one example being filenames that contain spaces.

Enter the spaces.

```bash
$ ls -lQ
total 0K
-rw------- 1 user group 0 Apr 12 07:31 "a"
-rw------- 1 user group 0 Apr 12 05:31 "b"
-rw------- 1 user group 0 Apr 12 06:31 "c d"
```

Enter the errors.

```bash
$ i=0; for old in `ls -tr`; do 
    printf -v new '%02d %s' $((++i)) "$old"; mv -v -- "$old" "$new"; done
‘b’ -> ‘01 b’
mv: cannot stat ‘c’: No such file or directory
mv: cannot stat ‘d’: No such file or directory
‘a’ -> ‘04 a’
```

The problem here is that `for` iterates over "words" 

```bash
$ help for
  for: for NAME [in WORDS ... ] ; do COMMANDS; done
```

And because we have a filename that contains a space we end up with `4`
"words" for the `3` filenames.

```bash
$ ls -tr
b  c d  a
```

To fix this we could change the `for` loop to a `while` loop and treat 
each filename as a "line" instead of a "word"

```bash
$ i=0; ls -tr | while read -r old; do 
    printf -v new '%02d %s' $((++i)) "$old"; mv -v -- "$old" "$new"; done
‘b’   -> ‘01 b’
‘c d’ -> ‘02 c d’
‘a’   -> ‘03 a’
```

This approach will break though if any filename contains a newline character.
Yes, filenames can contain newlines. No, I've never had a filename with a
newline in it. 

We can create such a filename using `touch $'e\nf'`

```bash
$ ls -lQ
total 0K
-rw------- 1 user group 0 Apr 12 07:31 "a"
-rw------- 1 user group 0 Apr 12 05:31 "b"
-rw------- 1 user group 0 Apr 12 06:31 "c d"
-rw------- 1 user group 0 Apr 12 21:46 "e\nf"
```

From the [manual](https://www.gnu.org/software/bash/manual/bashref.html#ANSI_002dC-Quoting):

> Words of the form `$'string'` are treated specially.  
> The word expands to string, with backslash-escaped characters 
replaced as specified¬ by the ANSI C standard.
> The expanded result is single-quoted, as if the dollar sign had not been present.

Similar to the extra "word" problem with `for` we now have an 
extra "line" with `while` as `e` and `f` are treated as separate
filenames.

```bash
$ i=0; ls -tr | while read -r old; do 
    printf -v new '%02d %s' $((++i)) "$old"; mv -v -- "$old" "$new"; done
‘b’   -> ‘01 b’
‘c d’ -> ‘02 c d’
‘a’   -> ‘03 a’
mv: cannot stat ‘e’: No such file or directory
mv: cannot stat ‘f’: No such file or directory
```

One possible solution to this problem is to use the 
<em>null-byte</em> to delimit entries as opposed to 
spaces or newlines and that means ditching the use of `ls`.

## find

GNU `find` has `-printf` which takes a format string to print out
all kinds of stats (similar to the `stat(1)` command) such as
`%t` for the last modification time. GNU `sort` has `-z` which 
allows it to process null-delimited records.

This allows us to build a list of null-delimited filenames and sort 
them according to their last modification time which we can then 
process with a `while read` loop.

```bash
$ i=0; find -mindepth 1 -printf '%T@ %p\0' | sort -z -k1,1n | 
while read -r -d '' _ old
do 
    dirname=${old%/*}/
    printf -v new '%s%02d %s' "$dirname" $((++i)) "${old##*/}"
    mv -v -- "$old" "$new"
done
‘./b’    -> ‘./01 b’
‘./c d’  -> ‘./02 c d’
‘./a’    -> ‘./03 a’
‘./e\nf’ -> ‘./04 e\nf’
```

You may notice that we have `./` in the output here. That is because we get the 
full path back from `find` so technically we could omit the `--` and it would
work as expected with filenames with leading dashes.

The result:

```bash
$ ls -lQ
total 0K
-rw------- 1 user group 0 Apr 12 05:31 "01 b"
-rw------- 1 user group 0 Apr 12 06:31 "02 c d"
-rw------- 1 user group 0 Apr 12 07:31 "03 a"
-rw------- 1 user group 0 Apr 12 21:46 "04 e\nf"
```

The `-mindepth 1` prevents the directory from appearing
in the results. 

```bash
$ find
.
./b
./c d
./a
./e?f
$ find -mindepth 1
./b
./c d
./a
./e?f
```

As mentioned `%t` prints the last modification time. Using
`%T` also prints it but allows you to customize the format. 
`@` is used to set the format to <em>epoch time</em> i.e.
to print it out as seconds since the epoch.

```bash
$ find -mindepth 1 -printf '%t\n'
Wed Apr 12 05:31:00.0000000000 2017
Wed Apr 12 06:31:00.0000000000 2017
Wed Apr 12 07:31:17.0508221217 2017
Wed Apr 12 21:46:53.0661150219 2017
$ find -mindepth 1 -printf '%T@\n'
1491971460.0000000000
1491975060.0000000000
1491978677.5082212170
1492030013.6611502190
```

`%p` is the path of the item found and we used `\0` to delimit
the record with the null-byte.

```bash
$ find -mindepth 1 -printf '%T@\0' | cat -vet
1491971460.0000000000^@1491975060.0000000000^@1491978677.5082212170^@1492030013.6611502190^@$
```

The `^@` here is a visual representation of the null-byte.

A full list of the `-printf` format directives can be found 
[in the manual](https://www.gnu.org/software/findutils/manual/html_node/find_html/Format-Directives.html#Format-Directives).

The requirements we were given stated that we had a flat directory that
only contained files and they were all to be renamed. A solution using
`find` would allow for recurision and one could also utilize the filtering
capabilities of `find` if needed e.g. `-name`, `-type`, etc.

The results of the `find` command are piped to `sort -z -k1,1n`

`-z` tells `sort` to expect null-delimited records as opposed to newline. 
`-k1,1n` is to sort on the first "field" where `n` means to sort
numerically.

The sorted results are then fed to `while read -r -d '' _ old`

`-r` has to do with backslash handing, you pretty much always 
want to use `-r` with `read`

`-d ''` sets the delimiter to null rather than a newline.

`_` is just a placeholder variable name, we could have instead 
used `read -r -d '' time old`. Using `_` is a naming convention
if you have to set a variable but don't need to use it.

Because `%p` from the `-printf` gives the full path to the file 
we need to break it up into the `directory` component and `filename`
component as to only add the prefix to the filename.

That is what we're doing with `dirname=${old%/*}/` and `${old##*/}`
which is called <em>Parameter Expansion</em> and they're essentially
doing the same as the `dirname` and `basename` commands.

```bash
$ old=/foo/bar/baz/file.ext
$ echo "${old%/*}"
/foo/bar/baz
$ echo "${old##*/}"
file.ext
```

If you didn't have access to GNU `find` and `sort` you would probably
have to reach for `perl`, `python`, `ruby` or whichever language
you like to use.

## perl

With `perl` you could use something like:

```bash
$ perl -e 'system "echo", "mv", "-v", "--", $_, sprintf "%02d %s", ++$i, $_ 
    for sort { (stat $a)[9] <=> (stat $b)[9] } <*>'
mv -v -- b 01 b
mv -v -- c d 02 c d
mv -v -- a 03 a
mv -v -- e
f 04 e
f
```

We use `sprintf` to do the zero-padding and gives us back a string.
`<*>` here is doing the same as `glob "*"` which as the name 
suggests mimics the behaviour of the shell glob `*`

These filenames are passed to the `sort` block which sorts 
based on the 10th element of the `stat` function which is 
the last modification time.

Written out in "full-form" it could look like:

```perl
my @files = sort { ... } glob "*";
for my $file (@files) {
    system "echo", "mv", "-v", "--", $file, ...
}
```

The `$_` in our one-liner is called the <em>Default Variable</em>.
If you don't supply a variable name to the `for` loop `$_` will
be used. So `$_` will contain the filename on each iteration of
the loop.

We're only executing `echo` to see how the generated command will
look. It looks okay so we can remove it to run `mv` instead.

```bash
$ perl -e 'system "mv", "-v", "--", $_, sprintf "%02d %s", ++$i, $_ 
    for sort { (stat $a)[9] <=> (stat $b)[9] } <*>'
‘b’    -> ‘01 b’
‘c d’  -> ‘02 c d’
‘a’    -> ‘03 a’
‘e\nf’ -> ‘04 e\nf’
$ ls -lQ
total 0K
-rw------- 1 user group 0 Apr 12 05:31 "01 b"
-rw------- 1 user group 0 Apr 12 06:31 "02 c d"
-rw------- 1 user group 0 Apr 12 07:31 "03 a"
-rw------- 1 user group 0 Apr 12 21:46 "04 e\nf"
```

You could use Perl's `grep` if you needed to filter the filenames
generated from the `glob` e.g. `grep { -f } <*>` to match only 
<em>"plain files"</em>. 

You could also use the `File::Find` module to implement a 
recursive solution if needed.

## python

It's also possible to produce a Python <em>"one-liner"</em> to
do rougly the same thing.

```bash
$ python -c 'import os, subprocess; [
    subprocess.call(("echo", "mv", "-v", "--", f, "%02d %s" % (i, f))) 
      for i, f in enumerate(sorted(os.listdir("."), key=os.path.getmtime), start=1)]'
mv -v -- b 01 b
mv -v -- c d 02 c d
mv -v -- a 03 a
mv -v -- e
f 04 e
f
```

You can of course just write it "normally" which could look something like:

```python
import os, subprocess
    
filenames = sorted(os.listdir("."), key=os.path.getmtime)
for count, filename in enumerate(filenames, start=1):
    subprocess.call(...)
```

`sorted()` can take a <em>"callback"</em> (the name of a function to call or
an <em>"inlined"</em> function created with `lambda`) via the `key` argument.

`enumerate()` takes an iterable and gives us the `index` and `item` per iteration.
It defaults to `start` at `0` so we change that to `1`.

```python
>>> list(enumerate(['one', 'two', 'three']))
[(0, 'one'), (1, 'two'), (2, 'three')]
>>> list(enumerate(['one', 'two', 'three'], start=1))
[(1, 'one'), (2, 'two'), (3, 'three')]
```

As we're importing the `os` module we can replace our `subprocess.call()` to `mv`
with `os.rename()`. We lose the verbose output from `mv -v` though.

```bash
$ python -c 'import os; [ os.rename(f, "%02d %s" % (i, f))
    for i, f in enumerate(sorted(os.listdir("."), key=os.path.getmtime), start=1)]'
$ ls -lQ
total 0K
-rw------- 1 user group 0 Apr 12 05:31 "01 b"
-rw------- 1 user group 0 Apr 12 06:31 "02 c d"
-rw------- 1 user group 0 Apr 12 07:31 "03 a"
-rw------- 1 user group 0 Apr 12 21:46 "04 e\nf"
```

You could utilize `os.walk()` if you needed a recursive solution.

## That's it!

Yes, you can to parse `ls`. Yes, there are edge cases where it can fail.  
If these edge cases do not apply to you however, it can be the "simplest"
approach. 

Be aware of the edge cases and choose the appropriate approach for your 
specific needs.
