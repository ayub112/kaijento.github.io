---
title: "bash: add line from file1 to end of lines in file2"
date:  2017-04-18 18:33:04 +0100
tags:
 - bash
 - awk
 - perl
---

<em>I have file1 that contains a single line how do I add this
line to the end of all lines in file2?</em>

Let's create our 2 test files.

```bash
$ cat file1
    line from file1
$ cat file2
1
2
3
4
5
```

## bash

To do it in *"plain bash"* we could use a `while read` loop:

```bash
$ ( IFS=; read -r line1 < file1; while read -r line; do echo "$line$line1"; done < file2 )
1    line from file1
2    line from file1
3    line from file1
4    line from file1
5    line from file1
```

We are creating an explicit subshell here with the surrounding `()` as we're 
modifying the `IFS` variable and don't want to modify it for our current shell.

We're setting `IFS` to an empty value as to not strip any possible leading or
trailing whitespace.

```bash
$ read -r line1 < file1; declare -p line1
declare -- line1="line from file1"
$ IFS= read -r line1 < file1; declare -p line1
declare -- line1="    line from file1"
```

## awk

Another approach could be to use `awk`:

```bash
$ awk 'NR == FNR { line = $0; next } $0 = $0 line' file1 file2
1    line from file1
2    line from file1
3    line from file1
4    line from file1
5    line from file1
```

`NR` in the current record number (records by default in `awk` are lines) and 
`FNR` is the current record number of the current file. This means that 
`NR == FNR` will be true when the first file is being processed.

`$0` is the current record content and `awk` will implicitly join strings
for us. It's as if we did `printf("%s%s\n", $0, line)`

Because we know that `file1` only has a single line we could simplify the command:

```bash
$ awk 'BEGIN { getline line } $0 = $0 line' file1 file2
1    line from file1
2    line from file1
3    line from file1
4    line from file1
5    line from file1
```

## perl

If we wanted to modify `file2` with the updated changes we could
use `perl`


```bash
$ perl -pe 'BEGIN { chomp($line = `cat file1`) } s/$/$line/' file2
1    line from file1
2    line from file1
3    line from file1
4    line from file1
5    line from file1
```

We can also add the `-i` option to overwrite
the original file e.g. `perl -pi -e '...'`

We're shelling out to `cat` here as it's less code. You could `open`
the file without shelling out if you wished. `chomp()` removes
the trailing newline.

Recent versions of GNU `awk` have `-i inplace` which can also overwrite 
the original file however you would have to modify the command as we're
passing it 2 filenames as arguments.
