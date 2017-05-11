---
title: "JSON parsing: does a number contain a certain digit?"
date:  2017-04-20 09:12:27 +0100
tags:
 - bash
 - jq
 - python
---

<em>I have a list of JSON objects that contain `name` and `id` 
entries. I would like to filter out all entries that contain
a certain digit in their `id` e.g. `1`</em>

`id.json`

```javascript
[ 
  {"name": "alice", "id": 1},
  {"name": "bob",   "id": 201},
  {"name": "frank", "id": 30},
  {"name": "ted",   "id": 555}
]
```

This person wants to do this from the *"command-line"* using *"bash"*.

There are some JSON parsers written in pure `bash` there is even one
for `Perl` that is a [single regex](http://www.perlmonks.org/?node_id=995856).

My preferred options are `python` or `jq` but there are many to choose from.

## Python

```python
>>> import json
>>> 
>>> with open('id.json') as f:
...     people = json.load(f)
...     for person in people:
...         if '1' in person['id']:
...             print(person)
... 
Traceback (most recent call last):
  File "<stdin>", line 4, in <module>
TypeError: argument of type 'int' is not iterable
```

The problem is that `id` is a number in the JSON data
and not a string so it gets converted to an `int` when
loaded by the `json` module.

To fix this we can use `str()` to convert it into a string.

```python
>>> with open('id.json') as f:
...     people = json.load(f)
...     for person in people:
...         if '1' in str(person['id']):
...             print(person)
... 
{'name': 'alice', 'id': 1}
{'name': 'bob', 'id': 201}
```

Well that's great but how do we get it back in JSON format?

```python
>>> with open('id.json') as f:
...     people = json.load(f)
...     found  = [ person for person in people if '1' in str(person['id']) ]
...     print(json.dumps(found, indent=4))
... 
[
    {
        "name": "alice", 
        "id": 1
    }, 
    {
        "name": "bob", 
        "id": 201
    }
]
```

The `found = [ ... for ... in ... ]` syntax here is called a *list comprehension*. 
It's the equivalent to

```python
found = []
for person in people:
    if '1' in str(person['id']):
        found.append(person)
```

It's just *"shorthand syntax"* for creating a list.

You could write the whole command as a *"one-liner"* 

```jq
$ python -c 'import json, sys; print(json.dumps(
    [person for person in json.load(sys.stdin) if "1" in str(person["id"])], 
    indent=4))' < id.json
[
    {
        "name": "alice", 
        "id": 1
    }, 
    {
        "name": "bob", 
        "id": 201
    }
]
```

But if you're looking for a *"one-liner"* you may wish to check out `jq`

## jq

With `jq` we can use `select()` to filter so here want to to filter
out those whose `.id` value `contains` the string `1`

```bash
$ jq '.[] | select(.id | contains("1"))' id.json
jq: error (at id.json:6): number (1) and string ("1") cannot have their containment checked
```

`.[]` here *"unpacks"* the *"list"* of objects so it's like the
equivalent of our `for person in people` loop. You could use 
`.[x]` to access index `x` of a *"list"*.

We have the same issue as we did with the Python version though.
We need to convert `.id` to a string which we can do with `tostring`

```jq
$ jq '.[] | select(.id | tostring | contains("1"))' id.json
{
  "name": "alice",
  "id": 1
}
{
  "name": "bob",
  "id": 201
}
```

You may noticed though that we have lost the original structure
but we can just add it back in by surrounding our filter with `[]`

```jq
$ jq '[ .[] | select(.id | tostring | contains("1")) ]' id.json
[
  {
    "name": "alice",
    "id": 1
  },
  {
    "name": "bob",
    "id": 201
  }
]
```

`map(x)` is another way of writing `[ .[] | x ]` so you may also
see the command written as

```jq
$ jq 'map(select(.id | tostring | contains("1")))' id.json
[
  {
    "name": "alice",
    "id": 1
  },
  {
    "name": "bob",
    "id": 201
  }
]
```
