---
title: "XML parsing: Python ~ regex ~ iterparse()"
date:  2017-04-18 07:15:11 +0100
tags:
 - python
 - xml
 - regex
 - csv
 - lxml
 - dictwriter
 - xmltocsv
---

We're given the task of parsing XML and turning it into CSV
using Python. Due to the size of the data it cannot be
loaded directly into memory which means we cannot use an 
XML parser (or does it?)

In cases where we have multiple subtags they need to be merged 
into a single column / cell in the final CSV output.

```xml
<Record>
    <EA>9780415836661</EA>
    <I3>9780415836661</I3>
    <TI>Applied Multivariate Statistics for the Social Sciences</TI>
    <ST>Analyses with SAS and Ibm's SPSS</ST>
    <AUS>
            <AU>Pituch, Keenan A.</AU>
            <AU>Stevens, James P.</AU>
    </AUS>
    <BCS>
            <BC>JHBC</BC>
    </BCS>
    <BI>Paperback</BI>
    <CO>United Kingdom</CO>
    <MP>No</MP>
    <PD>20160105</PD>
    <PA>814 pages, 352 black &amp; white tables, 65 black &amp; white halftones</PA>
    <NP>814</NP>
    <RP>62.99</RP>
    <RI>62.99</RI>
    <RE>62.99</RE>
    <DI>181 x 255 x 41</DI>
    <EI>6 Rev ed</EI>
    <PU>Taylor &amp; Francis Ltd</PU>
    <YP>2016</YP>
    <RSS>
            <RS RC="P">Professional &amp; Vocational</RS>
    </RSS>
    <IU>352 black &amp; white tables, 65 black &amp; white halftones</IU>
    <RF>R</RF>
    <WE>1464</WE>
    <SG>2</SG>
</Record>
```

So here the multiple `<AU>` values need to be merged as a single "cell" 
separated by a space.

The `<EA>` and `<I3>` numbers need to be reformatted into scientific 
notation e.g. `9.78E+12`

All tags may not be present in each record and they should be given an empty
value in the final CSV row. We've been given a full list of the possible
tag names we need to process.

## parsing XML with regex

The sample data we are given looks well formed and has 1 tag per line
so perhaps we could try to process it line by line and use regular 
expressions to parse out the bits we need.

Yes, regular expressions. Yes, you're not supposed to use them like 
<em>evar!!</em> Yes, you're not supposed to use them on XML, etc. etc.

```python
import csv, fileinput, re, sys

fieldnames = [
    'EA', 'I3', 'TP', 'TI', 'ST', 'ED', 'AU', 
    'TR', 'BC', 'BI', 'CO', 'MP', 'PD', 'PA', 
    'NP', 'RP', 'RI', 'RE', 'DI', 'EI', 'PU', 
    'YP', 'RS', 'SR', 'IU', 'DE', 'RF', 'WE', 
    'SG', 'IB', 'AV', 'PI', 'GC', 'NC', 'IL', 
    'CP', 'LA', 'RC', 'SE', 'PT', 'PN', 'SI'
]

writer = csv.DictWriter(sys.stdout, fieldnames=fieldnames)
writer.writeheader()

row = {}
for line in fileinput.input():
    line = line.replace('&amp;', '&').strip()
    match = re.search(r'^<(\S+)[^>]*>([^<]+)</\1>$', line)
    if match:
        tag, content = match.groups()
        if tag in ('EA', 'I3'):
            content = '%.2E' % int(content)
        if tag in row:
            row[tag] += " " + content
        else:
            row[tag] = content
    elif line == '</Record>':
        writer.writerow(row)
        row = {}
```

The regular `csv.writer` deals with lists. If you do not
have the correct amount of values in your list you would have
to manually add empty values for the missing columns. 

`csv.DictWriter` instead deals with dicts and because it's given the list
of column names / headers / <em>"fieldnames"</em> when being created
it knows if any columns are missing and replaces them with empty values
automatically.

The `fileinput` module allows us to easily process files passed as a command-line 
argument. It can be used for more than that, though.

We're writing the output to `sys.stdout` for demonstration purposes but you can
write it to an actual file if you wish. See the `csv` docs for examples on how
to do so.

So the regex we're using searches for a line that contains an opening
tag (with possible attributes) followed by some content followed
by a closing tag.

```python
>>> re.search(r'^<(\S+)[^>]*>([^<]+)</\1>$', '<AU>blah blah</AU>').groups()
('AU', 'blah blah')
>>> re.search(r'^<(\S+)[^>]*>([^<]+)</\1>$', '</Record>').groups()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
AttributeError: 'NoneType' object has no attribute 'groups'
```

* `^` matches the start of the string 
* `(\S+)` matches and captures a sequence of 1 or more non-whitespace characters i.e. the tag name
* `[^>]*` matches a sequence of 0 or more characters that are not `>` this allows for possible
attributes e.g. `<RS RC="P">`
* `([^<]+)` matches and captures a sequence of 1 or more characters that are not `<` i.e. the tag content
* `</\1>` matches `</` followed by the contents capture group 1 followed by `>` i.e. the closing of the tag we previously matched
* `$` matches the end of the string

So if we find a matching line with that we extract the tag name and content.
We then check if we need to reformat i.e. is it an `EA` or `I3` and add the
content to the row or <em>"merge"</em> the content if there is a value
already contained in the row for that tag.

Because `</Record>` is on a line on its own and the regex wont match it
we need to use `elif` to specifically check for it.

If we find a `</Record>` line we write the contents of `row` set it back
to an empty dict to hold the next row's values.

The output is rather large so we'll just show a few of the columns that
display the <em>"special cases"</em> namely `EA`, `I3` and `AU`

EA|I3|TP|TI|ST|ED|AU
-|-|-|-|-|-|-
9.78E+12|9.78E+12||Applied Multivariate Statistics for the Social Sciences|Analyses with SAS and Ibm's SPSS||"Pituch, Keenan A. Stevens, James P."

The regex version works because the opening and closing tags are on the same
line and there are no nested tags - what if that was not the case? 

Well attempting to use regex on XML can only get you so far. It can work for 
certain situations but if you want to properly parse XML you can use an actual
XML parser.

## iterparse()

So we mentioned earlier the data was too large to load up into memory at once
and use an XML parser however both `xml.etree.ElementTree` and `lxml.etree` 
support `iterparse()` which allows us to parse XML without the need to load
the whole file directly into memory at once. 

You may see this referred to as a <em>Streaming XML Parser</em>.

`iterparse()` returns an iterator of `event, element` pairs and the default behaviour
is to only return `end` events i.e. the closing of an element `</element>`

We're going to use `lxml.etree` here as <em>"efficiency"</em> was mentioned
in the requirements due to the size of the data being processed. `lxml` is
an interface to the `libxml2` and `libxslt` C libraries.

```python
import csv, lxml.etree, sys

xmlfile = 'input.xml'

fieldnames = [
    'EA', 'I3', 'TP', 'TI', 'ST', 'ED', 'AU', 
    'TR', 'BC', 'BI', 'CO', 'MP', 'PD', 'PA', 
    'NP', 'RP', 'RI', 'RE', 'DI', 'EI', 'PU', 
    'YP', 'RS', 'SR', 'IU', 'DE', 'RF', 'WE', 
    'SG', 'IB', 'AV', 'PI', 'GC', 'NC', 'IL', 
    'CP', 'LA', 'RC', 'SE', 'PT', 'PN', 'SI'
]

writer = csv.DictWriter(sys.stdout, fieldnames=fieldnames)
writer.writeheader()

row = {}
for event, elem in lxml.etree.iterparse(xmlfile):
    if elem.tag == 'Record':
        if row:
            writer.writerow(row)
            row = {}
    else: 
        if elem.tag in ('EA', 'I3'):
            content = '%.2E' % int(elem.text)
        if elem.tag in row:
            row[elem.tag] += " " + elem.text
        else:
            row[elem.tag] = elem.text
```

Just a couple of changes from our previous version. 

We no longer using `fileinput` and passing the name of the 
input filename directly to `iterparse()`

How does it do?

```python
ValueError: dict contains fields not in fieldnames: 'AUS', 'BCS', 'EDS', 'RSS'
```

`DictWriter()` handles the cases of missing columns but if a row has extra
columns i.e. ones not contained in `fieldnames` it will raise an error
when you try to write it.

The difference is that our regex version ignored lines that only contained
an opening tag e.g. `<AUS>` so we did not have this problem.

```xml
<AUS>
        <AU>Pituch, Keenan A.</AU>
        <AU>Stevens, James P.</AU>
</AUS>
<BCS>
        <BC>JHBC</BC>
</BCS>
```

Using `iterparse()` we would need to 
add a check to skip any tag that is not a `Record` or 
is not contained in `fieldnames`

```python
import csv, lxml.etree, sys

xmlfile = 'input.xml'

fieldnames = [
    'EA', 'I3', 'TP', 'TI', 'ST', 'ED', 'AU', 
    'TR', 'BC', 'BI', 'CO', 'MP', 'PD', 'PA', 
    'NP', 'RP', 'RI', 'RE', 'DI', 'EI', 'PU', 
    'YP', 'RS', 'SR', 'IU', 'DE', 'RF', 'WE', 
    'SG', 'IB', 'AV', 'PI', 'GC', 'NC', 'IL', 
    'CP', 'LA', 'RC', 'SE', 'PT', 'PN', 'SI'
]

writer = csv.DictWriter(sys.stdout, fieldnames=fieldnames)
writer.writeheader()

tags = set(fieldnames)

row  = {}
for event, elem in lxml.etree.iterparse(xmlfile):
    if elem.tag == 'Record':
        if row:
            writer.writerow(row)
            row = {}
    elif elem.tag not in tags:
        continue
    else: 
        content = elem.text
        if elem.tag in ('EA', 'I3'):
            content = '%.2E' % int(content)
        if elem.tag in row:
            row[elem.tag] += " " + content
        else:
            row[elem.tag] = content
```

We've created `tags = set(fieldnames)` the reason being that testing
if something is in a `set` is cheaper / <em>"faster"</em> than testing 
if something is in a `list`.

A `set` is similar to a list except that it cannot contain duplicated
items and it does not retain order. Having no specified order is why
we cannot just use a `set`. Our columns have a specified order.

So if it's a `Record` write the row else check if it's a "valid"
tag, it not we `continue` which jumps to the next iteration of the 
loop.

Otherwise we have a valid tag and we add or merge as needed just as
we did in the first version.

## That's it!

If you need to parse <em>"huge"</em> XML files that cannot fit into
memory one option is to use `iterparse()` from `xml.etree.ElementTree` 
or `lxml.etree`.

The final version of the code and the full `input.xml` file are 
available [on github](https://github.com/kaijento/code/tree/master/xmlparsing/2017/04/18).
