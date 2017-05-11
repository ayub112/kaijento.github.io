---
title: "find: permissions with -perm"
date:  2017-04-08 06:00:58 +0100
tags:
  - find
---

I would like to find <em>"files"</em> that have octal permissions `755` although
I'm not looking for an exact match. I would like files that have `7` set for user
OR `5` set for `group` OR `5` set for other.


## permissions

The first column in the output of `ls -l` shows us the symbolic mode of permissions:

```bash
$ ls -l c 
-rwxr-xr-x 1 user group 0 Apr  3 06:54 c
```

GNU `stat` allows you to see the octal representation of permissions by supplying
`%a` in the format string whereas `%n` gives the name.

```bash
$ stat -c %a\ %n c
755 c
```

So given `-rwxr-xr-x` if we ignore the intial `-` we have 3 "sets" of permissions
`user`, `group` and `other`:

* `rwx` for `user`
* `r-x` for `group`
* `r-x` for `other`

These are commonly referred to as "bits" e.g. `r` stands for the "read bit". Each "bit"
has a particular octal value:

|symbol  |name     |value  |
|--------|---------|-------|
|r     |read   |4    |
|w     |write  |2    |
|x     |execute|1    |

So if the user bits set are `rwx` this means the octal value is `4 + 2 + 1` which gives us
`7`. The `-` means the bit is not set so `r-x` gives us `4 + 0 + 1` i.e. `5`.

## find

For finding things we can use the `find` command. For dealing with
permissions we can use its `-perm` option. In its simplest
form `-perm` is used to match permissions exactly:

```bash
$ find . -perm 755 -exec ls -li {} +
3103056319 -rwxr-xr-x 1 user group 0 Apr  3 06:54 ./c
$ find -perm 755 -ls
3103056319    0 -rwxr-xr-x   1 user group        0 Apr  3 06:54 ./c
$ find -perm 755 -exec stat -c %a\ %n {} +
755 ./c
```

GNU `find` allows you to omit the start directories to search and will default
to `.` i.e. the current directory. 

It also has an `-ls` option which is essentially 
the same as supplying `-exec ls -li {} +`. If your `find` command does not support 
these features you can use the full form as shown in the first command above.

The `-exec` option is used for executing commands. If `{}` is used inside of `-exec`
it is replaced with the pathname of the current "found" file. 

`-exec` must be terminated by either `;` or `+` and as `;` has special meaning 
to the shell it must be quoted for it to be seen by `find` e.g. with `\` `''` or `""`

`-perm` can also be given symbolic modes:

```bash
$ find -perm u=rwx,g=rx,o=rx -ls
3103056319    0 -rwxr-xr-x   1 user group        0 Apr  3 06:54 ./c
```

It's extremely verbose compared to the octal `755` in this instance but can be of use 
in certain situations.

## /mode

So back to the original problem we don't want an exact match on the permissions
which `-perm` also allows you to do:

> `-perm /mode`
>
> Any of the permission bits mode are set for the file.  Symbolic modes are accepted in this form.

This sounds like `-perm /755` should do what we want, right?

First, some test files:

```bash
$ find -ls 
3103056314    0 drwx------   2 user group       62 Apr  3 06:54 .
3103056317    0 -rw-------   1 user group        0 Apr  3 06:54 ./a
3103056318    0 -rwS---r-x   1 user group        0 Apr  3 06:54 ./b
3103056319    0 -rwxr-xr-x   1 user group        0 Apr  3 06:54 ./c
3103824358    0 ---x--x--x   1 user group        0 Apr  3 06:54 ./d
3103824359    0 ---xr-x-w-   1 user group        0 Apr  3 06:54 ./e
3103056316    0 -------rwx   1 user group        0 Apr  3 06:54 ./f
2505025145    0 -rwx--x--x   1 user group        0 Apr  3 06:54 ./g
```

Their permissions in octal:

```bash
$ find -exec stat -c %04a\ %n {} +
0700 .
0600 ./a
4605 ./b
0755 ./c
0111 ./d
0152 ./e
0007 ./f
0711 ./g
```

So how does `-perm /755` fare?

```bash
$ find -perm /755 -ls
3103056314    0 drwx------   2 user group       62 Apr  3 06:54 .
3103056317    0 -rw-------   1 user group        0 Apr  3 06:54 ./a
3103056318    0 -rwS---r-x   1 user group        0 Apr  3 06:54 ./b
3103056319    0 -rwxr-xr-x   1 user group        0 Apr  3 06:54 ./c
3103824358    0 ---x--x--x   1 user group        0 Apr  3 06:54 ./d
3103824359    0 ---xr-x-w-   1 user group        0 Apr  3 06:54 ./e
3103056316    0 -------rwx   1 user group        0 Apr  3 06:54 ./f
2505025145    0 -rwx--x--x   1 user group        0 Apr  3 06:54 ./g
```

Not quite what we were expecting. What's going on here?  

Perhaps using a symbolic mode example will make it easier to understand.

The octal `700` is equivalent to the symbolic mode `u=rwx` so when it says 
**Any of the permission bits** are set this means that if `u=r`,
`u=w` or `u=x` are set there will be a match i.e. 

<em>"if any of the user mode bits in X are set or any of the group mode bits
in Y are set or any of the other mode bits in Z or set"<em>.

## -mode

As well as `/mode` there is `-mode` 

> `-perm -mode`
>
> All of the permission bits mode are set for the file.

`/mode` had previously been `+mode` in GNU `find` but was deprecated
in favour of `/mode` due to issues regarding `+` as it can be part of the 
symbolic mode string.

So for example `-perm -700` stipulates that `u=r` and `u=w` and `u=x` must 
all be set for there to be a match.

```bash
$ find -perm -700 -ls
3103056314    0 drwx------   2 user group       62 Apr  3 06:54 .
3103056319    0 -rwxr-xr-x   1 user group        0 Apr  3 06:54 ./c
2505025145    0 -rwx--x--x   1 user group        0 Apr  3 06:54 ./g
$ find -perm -u=rwx -ls 
3103056314    0 drwx------   2 user group       62 Apr  3 06:54 .
3103056319    0 -rwxr-xr-x   1 user group        0 Apr  3 06:54 ./c
2505025145    0 -rwx--x--x   1 user group        0 Apr  3 06:54 ./g
```

This means we can combine multiple `-perm` clauses to satisfy the original
goal. One for user, one for group and one for other:

```bash
$ find \( -perm -700 -or -perm -050 -or -perm -005 \) -ls 
3103056314    0 drwx------   2 user group       62 Apr  3 06:54 .
3103056318    0 -rwS---r-x   1 user group        0 Apr  3 06:54 ./b
3103056319    0 -rwxr-xr-x   1 user group        0 Apr  3 06:54 ./c
3103824359    0 ---xr-x-w-   1 user group        0 Apr  3 06:54 ./e
3103056316    0 -------rwx   1 user group        0 Apr  3 06:54 ./f
2505025145    0 -rwx--x--x   1 user group        0 Apr  3 06:54 ./g
$ find \( -perm -u=rwx -or -perm -g=rx -or -perm -o=rx \) -ls
3103056314    0 drwx------   2 user group       62 Apr  3 06:54 .
3103056318    0 -rwS---r-x   1 user group        0 Apr  3 06:54 ./b
3103056319    0 -rwxr-xr-x   1 user group        0 Apr  3 06:54 ./c
3103824359    0 ---xr-x-w-   1 user group        0 Apr  3 06:54 ./e
3103056316    0 -------rwx   1 user group        0 Apr  3 06:54 ./f
2505025145    0 -rwx--x--x   1 user group        0 Apr  3 06:54 ./g
```

## implicit -and

When you build a `find` command items are chained with an implicit "and" e.g.

    find -name foo -type f

is processed as if the command given was:

    find -name foo -and -type f

This is why we need to group the chain of "or" clauses with `()` (which need to 
be quoted so the shell doesn't interpret them) as otherwise the logic of the 
command would be incorrect.

i.e. we would have:

    A -or B -or ( C -and D )

instead of:

    ( A -or B -or C ) -and D


Do note that `-and` `-or` and `-not` are extensions. Their original equivalents
are `-a` `-o` and `!` respectively.

## at least exactly

So `./f` is matched because the `o=rx` bits are set. It also has `o=w` 
set meaning its octal value is `7`. 

```
3103056316    0 -------rwx   1 user group        0 Apr  3 06:54 ./f
```

What if we wanted to stipulate that it must be `5`?

We could add `-not -perm /022` to filter out results that have `g=w` or `o=w` set:

```bash
$ find \( -perm -700 -or -perm -050 -or -perm -005 \) -not -perm /022 -ls 
3103056314    0 drwx------   2 user group       62 Apr  3 06:54 .
3103056318    0 -rwS---r-x   1 user group        0 Apr  3 06:54 ./b
3103056319    0 -rwxr-xr-x   1 user group        0 Apr  3 06:54 ./c
2505025145    0 -rwx--x--x   1 user group        0 Apr  3 06:54 ./g
```

This however could be considered incorrect depending on the exact
intention as it doesn't match `./e` whose group bits equal octal `5`
exactly:

```
3103824359    0 ---xr-x-w-   1 user group        0 Apr  3 06:54 ./e
```

Here the group permission bits equal octal `5` but it's not a match 
because it has `o=w` set.

If we wanted to consider that as a match i.e.
if either group or other is exactly equal to `5` aka `r-x` we 
could do the following:

```bash
$ find \( -perm -700 -or \( -perm -050 -not -perm /020 \) -or \( -perm -005 -not -perm /002 \) \) -ls
3103056314    0 drwx------   2 user group       62 Apr  3 06:54 .
3103056318    0 -rwS---r-x   1 user group        0 Apr  3 06:54 ./b
3103056319    0 -rwxr-xr-x   1 user group        0 Apr  3 06:54 ./c
3103824359    0 ---xr-x-w-   1 user group        0 Apr  3 06:54 ./e
2505025145    0 -rwx--x--x   1 user group        0 Apr  3 06:54 ./g
```
