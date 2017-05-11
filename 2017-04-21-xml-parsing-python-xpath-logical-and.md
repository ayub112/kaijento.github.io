---
title: "XML parsing: Python ~ XPath ~ logical AND"
date:  2017-04-21 05:43:20 +0100
tags:
 - python
 - xml
 - lxml
 - elementtree
 - xpath
---

*Given the following XML data we need to locate all `<record>` tags
whose `<a>` tag is equal to the value `A` **and** `<b>` tag is 
equal to the value `B` and return their `<id>` using Python.*

`records.xml`

```xml
<records>
    <record>
        <id>1</id>
        <a>A</a>
        <b>B</b>
    </record>
    <record>
        <id>2</id>
        <a>A</a>
    </record>
    <record>
        <id>3</id>
        <b>B</b>
    </record>
    <record>
        <id>4</id>
        <a>A</a>
        <b>B</b>
    </record>
</records>
```

From this sample XML we would expect the result to be `[1, 4]`

## xml.etree.ElementTree

Python comes with the [xml.etree.ElementTree](https://docs.python.org/3/library/xml.etree.elementtree.html#tutorial)
parser for parsing XML.

The most basic form of processing is to iterate
from the root element.

```python
>>> import xml.etree.ElementTree as ET
>>> 
>>> doc = ET.parse('records.xml')
>>> 
>>> for child in doc.getroot():
...     child
... 
<Element 'record' at 0x7f9eddd68a50>
<Element 'record' at 0x7f9eddd68b50>
<Element 'record' at 0x7f9eddd68c10>
<Element 'record' at 0x7f9eddd68cd0>
```

So `child` will be each `<record>` tag we could use `find()`
to see if there is an `<a>` tag inside.

```python
>>> for child in doc.getroot():
...     child.find('a')
... 
<Element 'a' at 0x7fbf536186d8>
<Element 'a' at 0x7fbf53618818>
<Element 'a' at 0x7fbf536189f8>
```

To get the content we can use `.text` 

```python
>>> for child in doc.getroot():
...     child.find('a').text
... 
'A'
'A'
Traceback (most recent call last):
  File "<stdin>", line 2, in <module>
AttributeError: 'NoneType' object has no attribute 'text'
```

Not all `<record>` tags contain an `<a>` tag meaning we 
would have to test that `find()` found something or use 
`try` and `catch` the `Exception`

```python
>>> for child in doc.getroot():
...     a = child.find('a')
...     b = child.find('b')
...     if a is not None and b is not None:
...         child.find('id').text
...
'1'
'4'
```

In older versions we could have used `if a and b` but now you
must explicity test for `is not None`. 

*Shirley* there is a better way?


## XPath

`xml.etree.ElementTree` provides us with [limited XPath support](https://docs.python.org/3/library/xml.etree.elementtree.html#xpath-support)
which can simplify the task.

```python
>>> doc.findall('record')
[<Element 'record' at 0x7fbf53683e08>, <Element 'record' at 0x7fbf53618778>, ...
```

Using `record` here is like using the path `./record` it will only search 
the child tags it will not descend further down.

```
>>> doc.getroot().tag
'records'
```

So because we're at `<records>` we can see the `<record>` tags as they
are direct children whereas we cannot see the `<id>` tags as they are
inside the `<record>` tags i.e they are grandchildren.

```python
>>> doc.findall('id')
[]
```

To descend all the way down we can use the double slash `.//` instead 
of `./` 

```python
>>> doc.findall('.//id')
[<Element 'id' at 0x7fbf53618688>, <Element 'id' at 0x7fbf536187c8>,
```

If we look at the [list of supported XPath syntax](https://docs.python.org/3/library/xml.etree.elementtree.html#supported-xpath-syntax)
we can see

> `[tag='text']`
> Selects all elements that have a child named *tag* whose complete text content, including descendants, equals the given *text*.

This sounds like it's what we're trying to do.

```python
>>> len(doc.findall('.//record'))
4
>>> len(doc.findall('.//record[a="A"]'))
3
>>> for record in doc.findall('.//record[a="A"]'):
...     record.find('id').text
...
'1'
'2'
'4'
```

Technically we can omit the `.//` here as the `<record>` tags are direct children
as we previously discussed but that may not be the case in other examples so 
we will use `.//` to avoid any possible confusion.

Also note that you can use `*` if you don't want to name or you don't know the name
of the tag you're trying to target.

```python
>>> len(doc.findall('.//*[a="A"]'))
3
```

## and

Okay so that allows us to match on 1 tag how do we match our 2nd condition? i.e.
how do we say `[tag1='text1' and tag2='text2']`?

Well, we could try just that:

```python
>>> len(doc.findall('.//record[a="A" and b="B"]'))
[...]
SyntaxError: invalid predicate
```

This is a valid XPath expression it's just that `xml.etree.ElementTree`
doesn't parse it as valid with its *"limited"* support.

If you wanted *"better"* XPath support you could use `lxml.etree`

```python
>>> import lxml.etree
>>> lxml.etree.parse('records.xml').xpath('*[a="A" and b="B"]')
[<Element record at 0x7f2b9da01fc8>, <Element record at 0x7f2b9da01f38>]
```

It is possible to do it without `and` though.

If you chain multiple *"tests"* together it's as if there is an implicit
*"logical AND"* applied to them:

```python
>>> len(doc.findall('.//record[a="A"][b="B"]'))
2
```

And finally let's get each `id`

```python
>>> [ record.find('id').text 
>>> for record in doc.findall('.//record[a="A"][b="B"]') ]
...     record.find('id').text
...
'1'
'4'
```

We could create a list and `.append()` each `id` or we could use a *list comprehension*.

```python
>>> [ record.find('id').text for record in doc.findall('.//record[a="A"][b="B"]') ]
['1', '4']
```

## iterfind()

Whereas `findall()` returns a list `iterfind()` returns a *generator*.

```python
>>> doc.iterfind('.//record[a="A"][b="B"]')
<generator object select at 0x7fbf535e68b8>
>>> [ record.find('id').text for record in doc.iterfind('.//record[a="A"][b="B"]') ]
['1', '4']
```

So depending on what you're doing you may wish to replace `findall()` calls with `iterfind()`
ones.
