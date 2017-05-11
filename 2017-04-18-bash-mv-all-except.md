---
title: "bash: mv all except ..."
date:  2017-04-18 13:40:12 +0100
tags:
 - bash
 - find
 - glob
 - linux
---

<em>I have the directory structure `A/B` how do I `mv` everything
inside `A` down 1 level into `B`?</em>

This is essentially asking <em>"How do I move everything except X?"<em>

## find

There are many ways to approach this task. I think the simplest
solution is to use `find`

<small>(note: We're using GNU find here)</small>


```bash
$ find A
A
A/B
A/-foo
A/bar
A/.baz
```

We can use `-mindepth 1` to remove the search directory from the results

```bash
$ find A -mindepth 1
A/B
A/-foo
A/bar
A/.baz
```

In our example structure `B` contains no subdirectories. If it did we would
want to also use `-maxdepth 1` to prevent `find` from descending into 
subdirectories.

```bash
$ find A -mindepth 1 
A/B
A/B/C
A/bar
A/.baz
A/-foo
A/D
A/D/E
A/D/E/F
$ find A -mindepth 1 -maxdepth 1
A/B
A/bar
A/.baz
A/-foo
A/D
```

We will keep `-maxdepth 1` in our command for the sake of <em>"completeness"</em>.

To filter out `B` we can use `-not -name B`

```bash
$ find A -mindepth 1 -not -name B
A/-foo
A/bar
A/.baz
```

If we wanted to filter out multiple names one option is to use the
`\( -not -name B -or -not -name C \)` syntax.

`find` lists <em>"hidden"</em> entries too so when you say
<em>"all"</em> we assume you mean hidden entries too. 

You could add in a `-not -name '.*'`
if you wanted to filter out hidden entries.

Finally we can use `-exec` to execute `mv`

```bash
$ find A -mindepth 1 -maxdepth 1 -not -name B -exec mv -v -t A/B {} +
‘A/-foo’ -> ‘A/B/-foo’
‘A/bar’  -> ‘A/B/bar’
‘A/.baz’ -> ‘A/B/.baz’
```

* inside an `-exec` clause `{}` is replaced with each <em>"item"</em> found. 
You may like to think of it as a <em>"placeholder"</em>.

* GNU `mv` has `-t` which takes the <em>target</em> as an argument. it allows you
to `mv` multiples files somewhere in a single `mv` call.

The <em>"portable"</em> way to write it without `-t` would be:

```bash
$ find A -mindepth 1 -maxdepth 1 -not -name B -exec mv -v {} A/B \;
```

When `-exec` is ended by `;` (we need `\;` to stop the shell from interpreting the `;`)
the `-exec` clause is executed once per item found. When `+` is used the `-exec` clause
is executed as many times as needed. Multiple arguments are passed to the command being
executed similar to how `xargs` works.

```bash
$ find 
.
./a
./b
./c
$ find -exec echo {} \;
.
./a
./b
./c
$ find -exec echo {} +
. ./a ./b ./c
```

As you can see with `+` the `echo` was only executed once. There is a limit on the size
of a command line which you can see with `getconf ARG_MAX` so it will fill as many 
arguments in as possible on each execution.

## globbing

As well as find you can use bash's <em>globbing</em> to achieve the same result.
You can use `*` to say <em>"all"</em> <small>(although it wont match <em>"hidden"</em>
entries unless `dotglob` is enabled)</small> you can use `!(pattern)` to say 
<em>"everything but pattern" </em>

This is not part of the standard globbing syntax and needs `extglob` enabled to work.

<small>(note: You can also supply multiple patterns separated by `|` e.g `!(foo|bar)`)</small>

```bash
$ cd A
$ shopt -s extglob
$ echo !(B)
bar -foo
```

We need to enable `dotglob` to allow globs to match hidden entries.

```bash
$ shopt -s dotglob
$ echo !(B)
bar .baz -foo
```

It's also useful to enable `nullglob` to prevent any possible
errors if there are no matches to the glob. You can enable
all three at the same time.

```bash
$ shopt -s dotglob extglob nullglob
```

Let's try to `mv`

```bash
$ mv -v -t B !(B)
mv: invalid option -- 'o'
Try 'mv --help' for more information.
```

The problem here is that the leading dash in `-foo` is being
interpreted as an option by `mv` we can use the <em>"double dash"</em> 
`--` to indiciate the end of command line options.

```bash
$ mv -v -t B -- !(B)
‘bar’  -> ‘B/bar’
‘.baz’ -> ‘B/.baz’
‘-foo’ -> ‘B/-foo’
```

Another option is to use the full path in our glob by prepending
`./`

```bash
$ echo ./!(B)
./bar ./.baz ./-foo
$ mv -v -t B ./!(B)
‘./bar’  -> ‘B/bar’
‘./.baz’ -> ‘B/.baz’
‘./-foo’ -> ‘B/-foo’
```

As mentioned there are other approaches for example you
could `mv B` somewhere temporarily out of `A` then use
`mv A/* path/to/B` and finally `mv B` back into `A`.
Again note the use of `dotglob` if you want "hidden" 
entries to be matched.

Another option could be to use `rsync`. I personally 
prefer the `find` the most straightforward approach but 
YMMV.
