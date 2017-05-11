---
title: "JSON parsing: parent / child transformation"
date:  2017-04-20 10:58:15 +0100
tags:
 - python
 - json
 - defaultdict
---

Given the following JSON structure 

`before.json`

```javascript
[
    {
        "tag": [
            {
                "taglevel": 1, 
                "name": "Fruit"
            }, 
            {
                "taglevel": 2, 
                "name": "Apple"
            }
        ], 
        "title": "Random"
    }, 
    {
        "tag": [
            {
                "taglevel": 1, 
                "name": "Fruit"
            }, 
            {
                "taglevel": 2, 
                "name": "Apple"
            }
        ], 
        "title": "Other"
    }, 
    {
        "tag": [
            {
                "taglevel": 2, 
                "name": "Food"
            }
        ], 
        "title": "Words"
    }, 
    {
        "tag": [
            {
                "taglevel": 2, 
                "name": "Food"
            }, 
            {
                "taglevel": 2, 
                "name": "Apple"
            }
        ], 
        "title": "That"
    }
]
```

We are to transform / restructure it to create the desired output

```javascript
[
    {
        "name": "Fruit", 
        "tag_child": [
            {
                "name": "Apple", 
                "taglevel": 2
            }
        ], 
        "taglevel": 1
    }, 
    {
        "name": "NoLevel_1", 
        "tag_child": [
            {
                "name": "Food", 
                "taglevel": 2
            }, 
            {
                "name": "Apple", 
                "taglevel": 2
            }
        ], 
        "taglevel": 1
    }
]
```

The critera for the transformation are as follows:

* if any of the `tag` entries contain a `taglevel` of `1` this `name`
will be the *parent* and all other tag entries (which will have 
`taglevel` of `2` or greater) will be *children*

* if there are no `taglevel` entries of `1` then all tag entries
are to be children of the `NoLevel1` *parent*

* there are to be no duplicate children

We will start with the output:

```jq
$ python transform-json.py
[
    {
        "name": "NoLevel_1", 
        "tag_child": [
            {
                "name": "Food", 
                "taglevel": 2
            }, 
            {
                "name": "Apple", 
                "taglevel": 2
            }
        ], 
        "taglevel": 1
    }, 
    {
        "name": "Fruit", 
        "tag_child": [
            {
                "name": "Apple", 
                "taglevel": 2
            }
        ], 
        "taglevel": 1
    }
]
```

Here is the code used to generate it:

```python
import json
from   collections import defaultdict

with open('before.json') as json_file:
    json_data = json.load(json_file)

    result = defaultdict(set)

    for item in json_data:
        parent   = {'name': 'NoLevel_1'}
        children = []
        for tag in item['tag']:
            if tag['taglevel'] == 1:
                parent = tag
            else:
                children.append((tag['taglevel'], tag['name']))
        result[parent['name']].update(children)

    result = [ 
        {
            'name': parent, 
            'tag_child': [
                {'name': name, 'taglevel': taglevel} for taglevel, name in tags
            ],
            'taglevel': 1, 
        } for parent, tags in result.items()
    ]

    print(json.dumps(result, indent=4, sort_keys=True))
```

## collections.defaultdict

We're using a [defaultdict](https://docs.python.org/3/library/collections.html?highlight=ordereddict#collections.defaultdict) 
here. If you've not encountered this before it's like a regular `dict` except
that it can simplify the creation of nested structures.

For example, let's say you were processing some lines of data that contained
student names and their grade. You wanted to create a `dict` that had each
student name as the key and a `list` of their results as the value.

```python
>>> from collections import defaultdict
>>>
>>> test_results = 'Bob: 65\nAlice: 75\nBob: 72\nAlice: 74'
>>>
>>> grades = defaultdict(list)
>>> for result in test_results.splitlines():
...     name, grade = result.split(': ')
...     grades[name].append(grade)
...
>>> grades
defaultdict(<type 'list'>, {'Bob': ['65', '72'], 'Alice': ['75', '74']})
```

If we were to use a regular `dict` we would need to check if the name
was in the dict already if not create it e.g.

```python
if name not in grades:
    grades[name] = [grade]
else:
    grades[name].append(grade)
```

Another use could be for counting with `defaultdict(int)` however `collections.Counter`
takes care of most counting needs.

## set vs. list

We're using a `defaultdict(set)` the reason being that sets cannot contain duplicates.

```bash
>>> set(['bob', 'alice', 'bob', 'bob'])
set(['bob', 'alice'])
>>> names = set()
>>> names.add('bob')
>>> names
set(['bob'])
>>> names.add('bob')
>>> names
set(['bob'])
```

To add a single item to a `set` we can use `.add()` although to add multiple
items we need to use `.update()` 

```bash
>>> names.add(['bob', 'alice'])
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: unhashable type: 'list'
>>> names.update(['bob', 'alice'])
>>> names
set(['bob', 'alice'])
```

So back to our code.

In the `for` loop we loop through each of the `tag` lists inside
each object. We set a default value for `parent` in the event that
it is not present.

If we find the correct `taglevel` set the `parent` accordingly else we

```python
children.append((tag['taglevel'], tag['name']))
```

So with `children` we're creating a list of `tuple` pairs 
that contain the `taglevel` and `name` values. The reason 
for this is that a `dict` is not `hashable` thus meaning
it cannot be an element of a `set`. 

So in the case of

```python
{
    "tag": [
        {
            "taglevel": 2, 
            "name": "Food"
        }
    ], 
    "title": "Words"
}, 
```

We're would have

```python
parent   = {'name': 'NoLevel_1'}
children = [ (2, 'Fruit') ]
```

We then use `result[parent['name']].update(children)` to *"add"*
the list of children to the `set` in `result` that is pointed to 
by the key `parent['name']` i.e. `NoLevel_1` in this instance.

If we print out `result` after the `for` loop it looks like

```python
defaultdict(<type 'set'>, {
    'NoLevel_1': set([(2, 'Food'), (2, 'Apple')]), 
    'Fruit':     set([(2, 'Apple')])
})
```

If we changed it to `defaultdict(list)` and changed the `.update()` call to `.append()` 
we'd get similar output but we'd have our duplicates, hence the use of 
`set` over `list`

```python
defaultdict(<type 'list'>, {
    'NoLevel_1': [[(2, 'Food')],  [(2, 'Food'), (2, 'Apple')]], 
    'Fruit':     [[(2, 'Apple')], [(2, 'Apple')]]
})
```

## comprehending comprehensions

So when `for` loop is done we are left with the `defaultdict` which
we must then transform into the final output.

We've performed the transformation using some *comprehensions*. 

We could have done it without them by creating a new list
and using `.append()`

```python
final_result = []
for parent, tags in result.items():
    item = {
        'name':     parent,
        'taglevel': 1
    }
    children = []
    for taglevel, name in tags:
        child = {'name': name, 'taglevel': taglevel}
        children.append(child)
    item['tag_child'] = children
    final_result.append(item)
```

I find the *comprehension* version prettier.

```python
result = [ 
    {
        'name': parent, 
        'tag_child': [
            {'name': name, 'taglevel': taglevel} for taglevel, name in tags
        ],
        'taglevel': 1, 
    } for parent, tags in result.items()
]
```

Feel free to use whichever style you prefer.

*"H8rz gon h8"*
