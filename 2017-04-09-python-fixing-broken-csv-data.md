---
title: "Python: fixing broken \"CSV\" data"
date:  2017-04-09 06:00:43 +0100
tags:
 - python
 - csv
 - regex
 - perl
---

We are given the following "CSV" data which we are asked
to "fix" using Python:

`not-quite.csv`

```
10AT&T$146,801
11General Electric$140,389
12AmerisourceBergen$135,962
13Verizon$131,620
14Chevron$131,118
15Costco$116,199
16Fannie Mae$110,359
17Kroger$109,830
18Amazon.com$107,006
19Walgreens Boots Alliance$103,444
20HP$103,355
```

As it turns out it's not really CSV data at all as the columns are all 
merged into one. The data looks like it came from the 
[Fortune 500 list](http://beta.fortune.com/fortune500/list)
albeit "incorrectly".

So what do we know? We know the rank is going to be the first sequence 
of digits on the line.  The name of the company will then be everything 
up to but not including the dollar sign.  The revenue will be everything 
after the dollar sign from which we want to remove the comma.

## re.search()

This sounds like a job for a regular expression.

`not-quite-csv.py`

```python
import fileinput, re

for line in fileinput.input():
    rank, name, revenue = re.search(r'(\d+)(.+)\$(.+)', line).groups()
        revenue = revenue.replace(',', '')
            print(','.join([rank, name, revenue]))
```

The `fileinput` module is a simple way to write a filter-like program that
processes input line by line. It handles reading data from filenames passed
as arguments and data given on stdin. It can also be used to overwrite
files with the new data like how the `-i` option works on `perl` and some
versions of `sed`.

So for the regex we have `(\d+)` which matches and captures 
<em>1 or more digits</em> i.e. the <em>rank number</em>. `()` is used to
create a capture group. We could have used `^` (which matches the start
of the string) in our pattern but all of the lines we're working with
start with digits so it's not needed.

`(.+)\$` matches and captures everything up to the occurrence of a dollar sign. `$`
matches the end of the string in regex so it needs to be escaped to match literally.

Finally `(.+)` captures everything else on the line i.e. the revenue amount.

We then remove any commas from the revenue amount with `.replace(',', '')` and join
the values back using comma as the delimiter.

Here is a sample run:

```bash
$ python not-quite-csv.py not-quite.csv
10,AT&T,146801
11,General Electric,140389
12,AmerisourceBergen,135962
13,Verizon,131620
14,Chevron,131118
15,Costco,116199
16,Fannie Mae,110359
17,Kroger,109830
18,Amazon.com,107006
19,Walgreens Boots Alliance,103444
20,HP,103355
```

Instead of `','.join()` we could use the `csv` module which would handle things
like automatic quoting of fields that contain the delimiter but the sample data
we were given was "simple" meaning that it was not possible for the company
names to contain a comma.

## perl

If you do not have to use Python specifically you may also consider using
`perl` to solve this problem:

```bash
$ perl -pe 's/(\d+)(.+)\$(.+)/"$1,$2,".$3=~s\/,\/\/gr/e' not-quite.csv 
10,AT&T,146801
11,General Electric,140389
12,AmerisourceBergen,135962
13,Verizon,131620
14,Chevron,131118
15,Costco,116199
16,Fannie Mae,110359
17,Kroger,109830
18,Amazon.com,107006
19,Walgreens Boots Alliance,103444
20,HP,103355
```

The `-p` option adds a loop like the `for line in fileinput.input()` loop
from the Python example. The `s///` command is the same as `re.sub()` it
takes a pattern and a replacement. The pattern here is the same as we used
in the Python code so it needs no explanation.

So we're using the `e` modifier on the `s` command i.e. `s///e`. The `e`
modifier evaluates the replacement part as perl code. 

In the replacement `"$1,$2,"` is just creating a string of the first 2 
capture groups followed by commas.

`.` is the string concatenation operator like `"foo" + "bar"` in Python.
To the string we are adding `$3=~s\/,\/\/gr` which is actually another 
`s///` command embedded inside. This is why the `/` are escaped although
we could have chosen a different delimiter e.g. `s|,||gr`

This removes all commas from `$3` which is the revenue amount. Without
the `g` modifier it would just remove the first similar to 
`.replace(',', '', 1)` in Python.

By default the `s///` command does not return the modified string. In 
order to do that you must use the `r` modifier. 

So this means the final string we have in our replacement is 
<em>"rank,name,revenue with commas removed"</em>.

## a different approach

If we take another look at an input line can we see another way to
achieve the intended output?

```
10AT&T$146,801
```

If we removed the last comma in the string replace the `$`
with a comma and added a comma after the rank number we would
get the current expected output:

```bash
$ perl -pe 's/.+\K,//; s/\$/,/; s/\d+\K/,/' not-quite.csv 
10,AT&T,146801
11,General Electric,140389
12,AmerisourceBergen,135962
13,Verizon,131620
14,Chevron,131118
15,Costco,116199
16,Fannie Mae,110359
17,Kroger,109830
18,Amazon.com,107006
19,Walgreens Boots Alliance,103444
20,HP,103355
```

`s/.+\K,//` removes the last comma in the string. Using capture 
groups you could write it as `s/(.+),/$1/` which may look more 
familiar.

Anything to the left of `\K` is not included in the match which 
can be useful in simplifying patterns.

`s/\$/,/` replaces dollar sign with a comma. We could be more strict
and use the `.+\K\$` approach to stipulate it's the last dollar sign
but there are no other instances of it in our data.

Finally `s/\d+\K/,/` adds a comma after the first sequence of 
digits.

This approach makes assumptions about the data as it only strips
the last comma. If the revenue amount was larger and contained 
multiple commas it would break. It does work for the data we were
given and serves as an example approaching the problem from a 
different perspective.

## redirection

None of these approaches modify the input file. I find it's usually
simpler to just use shell redirection to output the result to a new file
e.g.

```bash
$ perl -pe 's/.+\K,//; s/\$/,/; s/\d+\K/,/' not-quite.csv > fortune500.csv
```

Do note that redirections are set up before command execution so the
output filename must differ from the input filename.

You could however use perl's `-i` option and pass `inplace=True` to 
`fileinput.input()` in the Python example if you wished to do so.

