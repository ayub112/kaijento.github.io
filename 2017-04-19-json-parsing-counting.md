---
title: "JSON parsing: counting"
date:  2017-04-19 08:28:53 +0100
tags: 
 - bash
 - jq
 - python
 - json
 - counter
---

The goal is to loop through each object in a list
of JSON objects and count produce a single object
as the result containing the counts of all `has_`
keys that have a `true` value. In the final object 
the leading `has_` string should be removed.

Meaning for the following `input.json`

```javascript
[   

    {   
        "detector": "<censored>",
        "has_email": true,
        "has_hipchat": false,
        "has_pagerduty": false,
        "has_slack": false,
        "has_victorops": false,
        "has_webhook": false,
        "id": "<censored>",
        "url": "<censored>"
    },
    {   
        "detector": "<censored>",
        "has_email": true,
        "has_hipchat": false,
        "has_pagerduty": false,
        "has_slack": true,
        "has_victorops": false,
        "has_webhook": false,
        "id": "<censored>",
        "url": "<censored>"
    }

]
```

The desired output would be:

```javascript
{
  "email": 2,
  "slack": 1
}
```

## Python 

We're working on a Linux command line so we will start with a solution
using Python.

```jq
$ python count-has.py
{
    "email": 2, 
    "slack": 1
}
```

Here is the code we ran.

`count-has.py`

```python
import json
from   collections import Counter

with open('input.json') as f:
    j = json.load(f)
    c = Counter(k[4:] for d in j for k, v in d.items() if k.startswith('has_') and v)
    print(json.dumps(c, indent=4, sort_keys=True))
```

When you need to count things in Python you could use a `dict` or a `collections.defaultdict`
however there is a `collections.Counter` object which simplifies things further.

```python
>>> from collections import Counter
>>> Counter('foobar')
Counter({'o': 2, 'a': 1, 'r': 1, 'b': 1, 'f': 1})
>>> Counter(['foo', 'bar', 'bar', 'baz'])
Counter({'bar': 2, 'baz': 1, 'foo': 1})
```

In our code we're passing `Counter()` a <em>generator statement</em> which may
look odd if you've never seen that or a <em>list comprehension</em> before. 

```python
>>> (k[4:] for d in j for k, v in d.items() if k.startswith('has_') and v)
>>> <generator object <genexpr> at 0x7f6506938500>
>>> list(k[4:] for d in j for k, v in d.items() if k.startswith('has_') and v)
>>> ['email', 'slack', 'email']
>>> [ k[4:] for d in j for k, v in d.items() if k.startswith('has_') and v ]
>>> ['email', 'slack', 'email']
```

Let's rewrite it using regular `for` loops.

```python
c = Counter()
for d in j:
    for k, v in d.items():
        if k.startswith('has_') and v:
            c.update([k[4:]])
```

Let's explain the variable names:

* `j` for `json`
* `c` for `count`
* `d` for `dict`
* `k` for `key`
* `v` for `value`

As we've split it out into regular `for` loops we may as well choose some descriptive
variable names.

```python
count = Counter()
for item in json.loads(data):
    for key, value in item.items():
        if key.startswith('has_') and value:
            count.update([key[4:]])
```

So `update()` adds a new item to a `Counter` object. The reason we need to wrap `key[4:]`
in a list is because if you pass a string to a `Counter` it will split it up into
characters and count them as with our `Counter('foobar')` example.

`key[4:]` here is called *slicing*. We know `key` starts with `has_` so if we get the
*5th* character to the end we will have removed `has_`. We can omit the end index
and Python will go to the end for us.

```python
>>> 'has_foo'[4:]
'foo'
```

As well as `if key.startswith('has_')` we have  `and value` to test that `value` is *"truthy"*.
`if value` is like saying `if value is True` however you don't need the `is True` as it is
implied.

## jq

Another option for parsing JSON from *"within bash"* is to use the excellent `jq` tool.

```bash
$ jq 'map(to_entries[] | select(.key[:4] == "has_" and .value)) | group_by(.key) | map({(.[0].key[4:]): length}) | add' input.json
{
  "email": 2,
  "slack": 1
}
```

A good way to help understand `jq` commands is to break each component down
and run them separately to visualize what's happening.

```jq
$ jq 'map(to_entries[])' input.json
[
  {
    "key": "detector",
    "value": "<censored>"
  },
  {
    "key": "has_email",
    "value": true
  },
  {
    "key": "has_hipchat",
    "value": false
  },
[...]
```

`map(to_entries[])` is equivalent to `[ .[] | to_entries[] ]` so it's a way to loop
over each *key, value* pair of each object and return a *"list"*.

From there we use `select(.key[:4] == "has_" and .value)` to *select* ones that
start with `has_` and have a *"truthy"* value. There is a `startswith` function
but we can also use *"slicing"*.

```jq
$ jq 'map(to_entries[] | select(.key[:4] == "has_" and .value))' input.json
[
  {
    "key": "has_email",
    "value": true
  },
  {
    "key": "has_email",
    "value": true
  },
  {
    "key": "has_slack",
    "value": true
  }
]
```

So we have a *list* of 3 objects. What we need to do is to *group* the objects
together based on their *key*. For that we can use `group_by(.key)`

```jq
$ jq 'map(to_entries[] | select(.key[:4] == "has_" and .value)) | group_by(.key)' input.json
[
  [
    {
      "key": "has_email",
      "value": true
    },
    {
      "key": "has_email",
      "value": true
    }
  ],
  [
    {
      "key": "has_slack",
      "value": true
    }
  ]
]
```

Now we have a *list of lists* so what we need to do is:

* extract their `key` whilst removing `has_`
* get their length

```jq
$ jq 'map(to_entries[] | select(.key[:4] == "has_" and .value)) | group_by(.key) | map({(.[0].key[4:]): length})' input.json
[
  {
    "email": 2
  },
  {
    "slack": 1
  }
]
```

Because we have a *list of lists* we use `map()` to iterate over each *sublist*. 
`.[0].key` extracts `key` from the first item in each *sublist*. We know they 
must have at least 1 element so we can just index it directly with `.[0]`

`[4:]` gives us the key without the leading `has_`

In order to use the result of an expression such as `.[0].key[4:]` as the *key* 
for an object we need to wrap it with `()` to prevent `jq` from getting confused.

So we extract the name for the *key* in our new object and the value is the 
result of the `length` function which will give us the *count*.

```bash
$ echo '[1,2,3]' | jq length
3
```

We still have a list of multiple objects so we need the final call to `add` to flatten
them down into a single object.

```jq
$ echo '[{"name": "me"},{"age": "539"}]' | jq 
[
  {
    "name": "me"
  },
  {
    "age": "539"
  }
]
$ echo '[{"name": "me"},{"age": "539"}]' | jq add
{
  "name": "me",
  "age": "539"
}
```

## That's it!

For parsing JSON both Python with its `json` module and `jq` are excellent options. 
Most languages will come with a JSON parser though, so feel free to use *&lt;insert your
favourite language here&gt;*

*"H8rz gon h8"*
