---
title: "bash: reading a file into an array"
tags: 
- bash
- hackerrank
---

<em>Given a list of countries, each on a new line, 
your task is to read them into an array and then display the element indexed at 3.
Note that indexing starts from 0.</em>

`Sample input`:

    Namibia
    Nauru
    Nepal
    Netherlands
    NewZealand
    Nicaragua
    Niger
    Nigeria
    NorthKorea
    Norway

`Sample Output`:

    Netherlands

This question was taken from the [http://hackerrank.com](http://hackerrank.com) challenge posted 
[here](https://www.hackerrank.com/challenges/bash-tutorials-display-the-third-element-of-an-array).

## Simplest solution?

bash 4 introduced `readarray` (also known as `mapfile`) which allows you to do:

```bash
readarray -t countries
echo "${countries[3]}"
```

I'm assuming this is not what the author of the challenge had in mind so the rest of this article
discusses how it would have <em>"normally"</em> been implemented e.g. using a `while read` loop.

## What is an array?

So firstly, what is an array? Well you have a "normal" variable which has a single value.
An array is like a list in that it can hold multiple values.

```bash
normal=variable
array=(a b c)
```

## The right way that could be wrong

For the purposes of formatting we will only take a few countries from the sample input.

```bash
$ cat sample-input
Namibia
Nauru
Nepal
Netherlands
$ while read country; do countries+=($country); done < sample-input
$ declare -p countries
declare -a countries='([0]="Namibia" [1]="Nauru" [2]="Nepal" [3]="Netherlands")'
$ echo ${countries[3]}
Netherlands
```

So `read country` reads a line of input from stdin and stores it into the variable
`country`. The `while` means that it will loop over all lines in stdin.

Variables don't need to be predeclared. You can append to a non-existing variable and
it <em>"Just Works"</em>.

```
$ s+=foo
$ s+=bar
$ declare -p s
declare -- s="foobar"
```

So `s` did not exist initially and `s+=foo` did the same as `s=foo` in this instance as
it appended `foo` to nothing. By default, variable are treated as "strings" so 
`s+=bar` then appends the string `bar` to the existing value `foo` giving us `foobar`.

In our code however, we have `countries+=()`. The `()` here forces the variable to be treated
as an array and not a string. When you append to an array it adds a new item to the end 
of the array.

```
$ s+=(baz)
$ declare -p s
declare -a s='([0]="foobar" [1]="baz")'
```

The `< sample-input` is file redirection. It sends the contents of the file `sample-input` to 
stdin. Incidientally, to redirect stdout to a file you can use `> output-file`

And finally we're using `declare -p` to give like a <em>"debugging output"</em> representation 
of a variable.

## Space builds a new place

So let's replace `Nepal` with `New Zealand` in our sample input. <strong>Note</strong> that we
are also adding in the space unlike in the given sample input.

```bash
$ cat sample-input
Namibia
Nauru
New Zealand
Netherlands
$ countries=()
$ while read country; do countries+=($country); done < sample-input
$ declare -p countries
declare -a countries='([0]="Namibia" [1]="Nauru" [2]="New" [3]="Zealand" [4]="Netherlands")'
```

`countries=()` sets `countries` back as an empty array removing the contents from
our previous run.

We now have 5 countries instead of 4. WTF is going on pls?

## Word-splitting

```bash
$ country=New Zealand
bash: Zealand: command not found
```

When parsing bash splits things into <em>"words"</em> - so here we have 2 words `country=New` and `Zealand`.
This is not the behaviour we want so we could use one of the following:

```bash
$ country='New Zealand'
$ country="New Zealand"
$ country=New\ Zealand
```

The difference between single and double quotes is that inside double quotes variables will be replaced
by their values. (You may see this referred to as <em>"expansion"</em>.)

```bash
$ name=Karl
$ echo "My name is: $name"
My name is: Karl
$ echo 'My name is: $name'
My name is: $name
```

But we're using `read` to store our value in `country` so that's not our problem? Well yes, the problem is
with `countries+=($country)`

## set -x

We will use `set -x` which will enable debugging output of how bash is executing our commands. `set +x` 
can be used to turn it back off.

```bash
$ set -x
$ echo $country
+ echo New Zealand
New Zealand
$ echo "$country"
+ echo 'New Zealand'
New Zealand
```

So when we used double quotes around `$country` bash executed `echo 'New Zealand'` i.e. it 
treated the value of `$country` as a single word. Without the double quotes the value of 
`$country` was split up into multiple words. 

## args ()

To give another example:

```bash
$ args () { echo $#; }
$ args $country
2
$ args "$country"
1
```

So here we define a shell function `args` which just echos out `$#` which is the number of arguments passed.
As you can see because of the lack of double quotes word-splitting occurred and we passed 2 arguments
instead of 1.

This is one of the reasons you will see `"$var"` used instead of just `$var`. 

Okay so we want `$country` to be treated as a single word so we must double quote it:

```bash
$ countries=()
$ while read country; do countries+=("$country"); done < sample-input
$ declare -p countries
declare -a countries='([0]="Namibia" [1]="Nauru" [2]="New Zealand" [3]="Netherlands")'
$ echo ${countries[3]}
Netherlands
```

## globbing

There are no quotes around `${countries[3]}` but it did not make a difference in this instance.
However, as well as the word-splitting issue another problem that can arise is if the value of your 
variable contains <em>globbing</em> characters:

```bash
$ ls
sample-input
$ var=*
$ echo "$var"
*
$ echo $var
sample-input
```

So unless you can be sure of the contents of your variable it's usually a good idea to double quote
any expansions.

## read -r

There are other possible issues with regards to `read` depending on the input being processed.

```bash
$ read -a words <<< 'foo\ bar baz'
$ declare -p words
declare -a words='([0]="foo bar" [1]="baz")'
$ read -r -a words <<< 'foo\ bar baz'
$ declare -p words
declare -a words='([0]="foo\\" [1]="bar" [2]="baz")'
```

Without `-r` bash interprets the backslash as a quoting character using it to group `'foo bar'` 
as a single word. Normally this is not something you want which is why some people will just always use `-r`.

The `-a` option of read makes the variable we store the result in an array instead of a <em>"regular"</em>
variable. Like we had `< sample-input` to redirect the contents of a file to stdin `<<<` can be
used to do with same with a <em>"string"<em> instead.

## IFS=

Another possible issue is the removal of leading and trailing whitespace. By default both will
be <em>"trimmed"</em> or <em>"stripped"</em>".

```bash
$ read -r line <<< '    I would like to keep my space.    '
$ declare -p line
declare -- line="I would like to keep my space."
$ IFS= read -r line <<< '    I would like to keep my space.    '
$ declare -p line
declare -- line="    I would like to keep my space.    "
```

The `IFS` variable is a string of characters that define how word-splitting behaves and how
lines are split up into words when using `read`. Its default value is `<space><tab><newline>`

```bash
$ printf %q\\n "$IFS"
$' \t\n'
```

So `IFS=` temporarily sets it to nothing preventing the trimming which is why you will 
see `while read` loops to read something line-by-line written as:

```bash
while IFS= read -r line
```

## var=value command

`IFS= read` doesn't permanently overwrite `IFS` because bash supports the following syntax:

    var=value command

This exports the variable into command's environment (and only that command). We've just 
given an empty value in `IFS=` case.

```bash
$ a=b bash -c 'declare -p a'
declare -x a="b"
$ declare -p a
bash: declare: a: not found
```

It's essentially shorthand syntax for `( export var=value; command )`. The `()` here explicitly 
create a subshell so the parent's environment remains unchanged.

## readarray / mapfile

Bash introduced `readarray` in version 4 which can take the place of the `while read` loop. (For whatever
reason they gave it 2 names `readarray` and `mapfile` are the same thing. I think `readarray` is a more
suitable name but YMMV.)

```bash
$ readarray countries < sample-input
$ declare -p countries
declare -a countries='([0]="Namibia
" [1]="Nauru
" [2]="New Zealand
" [3]="Netherlands
")'
```

By default though, it keeps the trailing newline. You can use `-t` to have it strip 
the trailing newline instead.

```bash
$ readarray -t countries < sample-input
$ declare -p countries
declare -a countries='([0]="Namibia" [1]="Nauru" [2]="New Zealand" [3]="Netherlands")'
$ echo "${countries[3]}"
Netherlands
```

The problem description doesn't mention the use of a file at all so we can assume they will
be providing the data on stdin already so we would remove `< sample-input` from our
actual solution.
