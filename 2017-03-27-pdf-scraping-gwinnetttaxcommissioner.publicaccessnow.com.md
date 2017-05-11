---
title: "PDF scraping: Gwinnett County Tax"
date:  2017-03-27 09:52:27 +0100
tags:
 - pdf
 - python
 - perl
 - pdftotext
 - csv
 - regex
---

Given a PDF file from 
[http://gwinnetttaxcommissioner.publicaccessnow.com](http://gwinnetttaxcommissioner.publicaccessnow.com)
we need to extract certain text from it and convert it to CSV using Python. You could say we want to 
<em>"Convert a PDF to TXT"</em> or even <em>"Convert a PDF to CSV"</em> with Python. 

## pdftotext

There are various PDF modules for Python such as `PyPDF2` and `pdfminer` however I've 
never had much luck with their `extractText()` functions (it has been a while since I tried them, 
perhaps things have changed). The best results have come from using `pdftotext` tool from the 
`Poppler` PDF rendering library.

It should be available as `poppler-utils` from your package manager on linux or `poppler` from homebrew
if you're on a mac. For windows there is a pdftotext from `xpdf` (Poppler is based on the xpdf codebase) 
[here](http://www.foolabs.com/xpdf/download.html). I've not tested but I'm assuming it should work the
same.

`pdttotext` has a `-layout` option which attempts to keep the structure of the text which greatly simplifies
working with its output.

The sample URL we're given is 
[this](http://gwinnetttaxcommissioner.publicaccessnow.com/Portals/0/PDF/Excess%20funds%20all%20years%20-%20rev02232017.pdf)
and here is what the PDF looks like:

![](/img/1490605297-gwinnetttaxcommissioner-pdf.png)

Let's run the PDF file through `pdftotext`. By default `pdftotext` will create a new file
with a `txt` extension e.g. if you run `pdftotext blah.pdf` it will create `blah.txt`.
We use `-` here as the output filename to make it print to stdout instead.

```bash
$ pdftotext Excess\ funds\ all\ years\ -\ rev02232017.pdf - | less
```

We should see output structured like this:

```
OWNER #1 
COMPANY NAME #1

R3007 766
R6066 125

COMPANY NAME #2
OWNER #2

COMPANY #1 ADDRESS
COMPANY #2 ADDRESS 

$
$

289.31
3,551.35
```

## pdftotext -layout

So records are intertwined together which is going to make it rather difficult to work with.
Let's try again with `-layout`:

![](/img/1490606509-pdftotext-layout.png)

Not exactly perfect but it's something we can work with. Each record is on its own line (it looks
like 2 but that's only due to line-wrapping in my terminal). There are some issues with whitespace
between the dollar sign and amount. The same issue appears fon page 2 with the Parcel Number. 
They all seem to follow the same format though so we should be able to clean that up.

So each record we want starts with a number and it ends with MONTH YEAR e.g. `3 .... AUG 2011`. The
simplest way to isolate these is probably to use regex. The records we want are numbered from `3` 
to `77` meaning that there are `74` in total.

## subprocess

```python
>>> import re, subprocess
>>> 
>>> pdf_file = 'Excess funds all years - rev02232017.pdf'
>>> command  = ['pdftotext', '-layout', pdf_file, '-']
>>> output   =  subprocess.check_output(command).decode()
>>> len(re.findall(r'(?m)^\d.* [A-Z]{3} \d{4}$', output))
74
```

To execute external commands we use the `subprocess` module. `check_output()` will return
the output as a single byte string so we `decode()` it. Note that we don't pass the command
as a single string but we pass it as a list (or tuple) of "arguments". 

## regex

In regex `^` matches the start of the string and `$` matches the end of the string. The `m`
modifer changes their behaviour so they match the start and end of the "line".

```python
>>> re.findall(r'^\d.*?$', '1 line\n2 line\n3 line')
[]
>>> re.findall(r'(?m)^\d.*?$', '1 line\n2 line\n3 line')
['1 line', '2 line', '3 line']
```

So with multiline mode `^` matches the start of each "line" as opposed to just the start
of the string. It's also possible to pass `flags=re.MULTILINE` instead of embedding 
`(?m)` into your pattern.

```python
>>> re.findall(r'^\d.*?$', '1 line\n2 line\n3 line', flags=re.MULTILINE)
['1 line', '2 line', '3 line']
```

So back to our pattern for matching our lines:

* `\d` matches a digit
* `.` matches "any" character (it wont match `\n` unless `(?s)` or `re.DOTALL` have been used)
* `*` means the previous "item" 0 or more times so `.*` allows to match "anything"
* `[A-Z]` matches an upppercase letter `{3}` means 3 times i.e the month `AUG`
* `\d{4}` matches 4 digits i.e. the year `2011`

So this is how we can match any lines that start with a digit and ends with space followed by
3 uppercase letters followed by space followed by 4 digits.

## re.split()

So now that we can match each line how do we split it up correctly?

```python
>>> line = '3 NAME NAME NAME     R7042   003B     COMPANY NAME         ADDRESS LINE 1         $        100.00      AUG 2011'
>>> line.split()[:5]
['3', 'NAME', 'NAME', 'NAME', 'R7042']
>>> line.split('  ')[:5]
['3 NAME NAME NAME', '', ' R7042', ' 003B', '']
```

If we split on a single space we wont be able to identify the individual pieces of information. If
we split on 2 spaces we will get lots of empty elements and some elements will require space trimming.

How do we split on "2 or more" spaces? That sounds like something regex can help us with?

```python
>>> re.split(r' {2,}', line)
['3 NAME NAME NAME', 'R7042', '003B', 'COMPANY NAME', 'ADDRESS LINE 1', '$', '100.00', 'AUG 2011']
```

Yes, there is `re.split()` which is like regular `split()` except it takes a regex as opposed 
to a string. `*` means "0 or more times" which is really just shorthand for `{0,}`. So "2 or more
times" is simply `{2,}`. 

There is a problem though with the spacing between the dollar sign and amount and with the parcel
number. We will need to fix these issues before we `re.split()`.

For the dollar sign we just need to replace `$` followed by 1 or more spaces with `$`. There are
no other instances of `$` in our data so we don't need to be more specific with the pattern. 

## re.sub()

```python
>>> re.split(r' {2,}', re.sub(r'\$ +', '$', line))
['3 NAME NAME NAME', 'R7042', '003B', 'COMPANY NAME', 'ADDRESS LINE 1', '$100.00', 'AUG 2011']
```

`re.sub()` allows you to substitute a regex pattern for replacement in a "string". The
`r''` here is a raw string which you can read about in the [re docs](https://docs.python.org/3/library/re.html#module-re).

So because `$` has special meaning in regex we need to escape it to match a literal dollar sign.
Space followed by `+` matches "1 or more" spaces which we simply replace with `$`. 

Onto the parcel number. Each parcel number follows the same pattern:

The letter `R` followed by 4 digits followed by "0 or more" spaces followed by 3 digits. Some 
of them end with an uppercase letter meaning it may not be there i.e. it is optional 
(0 or 1 times).

```python
>>> line = re.sub(r'\$\s+', '$', line)
>>> re.split(r' {2,}', re.sub(r' (R\d{4}) *(\d{3}[A-Z]?) ', r' \1\2 ', line))
['3 NAME NAME NAME', 'R7042003B', 'COMPANY NAME', 'ADDRESS LINE 1', '$100.00', 'AUG 2011']
```

`re.sub()` doesn't modify `line` it just returns the substituted text so we must assign it
back into `line` to update its contents.

`()` in regex creates a capture group which allows us to refer to what was captured. `\1` being
the first capture group `\2` the second, etc. These are commonly referred to as "backreferences".

`?` is the same as `{0,1}` which means "0 or 1 times"
i.e. `[A-Z]?` makes the presence of the `[A-Z]` "optional". So we capture both parts of
the parcel number and join them together in the replacement with `r' \1\2 '`. We're using a
raw string here because we don't want the backreferences to be interpreted as string escapes.

```python
>>> '\110\151'
'Hi'
```

We've matched the leading and trailing space in our pattern and put them back in the replacement.
The reason for matching the spaces was just to lessen the chances of our pattern accidentally 
matching something that isn't a parcel number. 

We get the same results without the spaces with our current data but we'll just leave them in
our pattern.

## csv

Now that we can match the lines and split them up correctly let's use the `csv` module to
turn it into CSV.

`pdftotext.py`

```python
import csv, re, subprocess, sys

pdf_file = 'Excess funds all years - rev02232017.pdf'
command  = ['pdftotext', '-layout', pdf_file, '-']
output   =  subprocess.check_output(command).decode()

writer = csv.writer(sys.stdout)
for line in re.findall(r'(?m)^\d.+[A-Z]{3} \d{4}$', output):
    line = re.sub(r'\$ +', '$', line)
    line = re.sub(r' (R\d{4}) *(\d{3}[A-Z]?) ', r' \1\2 ', line)
    row  = re.split(r' {2,}', line)
    writer.writerow(row)
```

Let's run it and see what the output looks like:

![](/img/1490637328-pdftotext.py.png)

The `head` command prints the first 10 lines (you can change the number with `-n` e.g. `-n 20`). Note
that the `csv` module will automatically quote any fields that contain the delimiter e.g. `"$3,551.35"`
in the output.

We've used `sys.stdout` as the target of the `csv.writer()` you could instead save the output to a file
from within the code.

## requests

The code assumes the PDF file is already saved to disk. Can we fetch the file from Python?
We will use `requests` to do so:

`pdftotext-url.py`

```python
import csv, re, requests, sys
from   subprocess import Popen, PIPE

url = (
    'http://gwinnetttaxcommissioner.publicaccessnow.com/'
    'Portals/0/PDF/Excess%20funds%20all%20years%20-%20rev02232017.pdf'
)

command = ['pdftotext', '-layout', '-', '-']

p = Popen(command, stdout=PIPE, stdin=PIPE)
r = requests.get(url, headers={'user-agent': 'Mozilla/5.0'})

stdin  = r.content
output = p.communicate(input=stdin)[0].decode()

writer = csv.writer(sys.stdout)
for line in re.findall('(?m)^\d.+\d$', output):
    line = re.sub(r'\$ +', '$', line)
    line = re.sub(r' (R\d{4}) *(\d{3}[A-Z]?) ', r' \1\2 ', line)
    row  = re.split(r' {2,}', line)
    writer.writerow(row)
```

The declaration of `url` may look like a tuple to you but it's not. Python will
implicity join strings together:

```python
>>> s = 'foo'   'bar'
>>> s
'foobar'
```

If you want to do that with strings on different lines we need to group them with
`()` 

```python
>>> s = ( 
...     'foo'
...     'bar'
... )
>>> s
'foobar'
```

We've changed the `command` to use `-` as the input filename which will make pdftotext
read from stdin. We can't use `check_output()` anymore either we must use `Popen()`.
We will use `communicate()` to pipe to the commands stdin. This means that we must  
set `stdin` and `stdout` to `subprocess.PIPE` in our `Popen()` call.

To fetch the URL we use `requests.get()` whilst setting the `user-agent`. The result
is available in `r.content` which we then pipe in as stdin of the pdftotext command.

Does it work as expected?

![](/img/1490639685-pdftotext-url.py.png)

It sure does. So what we have done here is essentially emulated a shell pipeline and as
we're using an external command to process the PDF we could do it without Python at all.

## perl is alive

![](/img/1490640260-perl.png)

We could use also use `curl` to fetch the file for us to replicate the last version of our
code

```bash
$ curl 'http://...' | pdftotext -layout - - | perl -nle 'print ....'
```

So let's break down that `perl` command:

```bash
$ perl -nle '
      print join ",", 
        map { /,/ ? "\"$_\"" : $_ } 
        split / {2,}/ 
        if s/\$ +/\$/, 
           s/ (R\d{4}) *(\d{3}[A-Z]?) / $1$2 /, 
           /^\d.+ [A-Z]{3} \d{4}$/'
```

Perl's `-n` option makes it read input line-by-line.
There is also `-p` which does the same but has an implicit `print` at the end.

Let's start from the end:

`if s/\$ +/\$/, s/ (R\d{4}) *(\d{3}[A-Z]?) / $1$2 /, /^\d.+ [A-Z]{3} \d{4}$/`

This should look somewhat familiar to you `s///` is just like the `re.sub()` we did in 
Python. For backreferences we use `$` instead of `\` (although it does support `\`).

The `$` in the first replacement needs to be escaped because Perl has a variable named
`$/` so the `\` stops Perl from interpreting it as that variable.

`/pattern/` does a match so with the above we've essentially done the equivalent of the 
following Python:

```python
    for line in output:
        line = re.sub()
        line = re.sub()
        if re.search(..., line):
            ...
```

`split / {2,}/` splits on 2 or more spaces and `",", join` joins them all together using
the comma as the delimiter. So what is this `map { ... }` business? Well remember how 
we noted that the `csv` module quoted any fields that contained the delimiter?

`map { /,/ ? "\"$_\"" : $_ } split / {2,}/`

So the `map` takes all items from the `split` as input. Inside the `map` block the current item
is available in the `$_` (default) variable. `/,/` checks if it contains a comma, if so
we return the item quoted else we return the item untouched.

It would be similar to the following Python:

```python
','.join('"{}"'.format(item) if ',' in item else item for item in re.split(...))
```

## That's it!

Each PDF file will be different so the exact details of "scraping" the data you
will change each time. Using `pdftotext` with its `-layout` option will usually
be a good first step to take though.
