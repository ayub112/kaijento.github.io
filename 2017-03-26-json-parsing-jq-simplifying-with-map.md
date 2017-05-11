---
title: "JSON parsing: jq simplifying with map()"
date:  2017-03-26 07:28:27 +0100
tags:
 - jq
 - json
---

The goal is update a list of objects by adding a new array to each 
object that contains values from specific keys already defined in
each object.

## Input

`input.json`

```json
[
  {
    "key1": "a",
    "key2": "b",
    "omg": "lol"
  },
  {
    "key1": "c",
    "key2": "d"
  }
]
```

## Output

So by specifying `key1` and `key2` the desired output would be:

```json
[
  {
    "key1": "a",
    "key2": "b",
    "key3": [
       "a",
       "b"
    ],
    "omg": "lol"
  },
  {
    "key1": "c",
    "key2": "d"
    "key3": [
       "c",
       "d"
    ],

  }
]
```

## adding a new key/value pair with +

In other words: <em>"How do I add a new value to an object based on existing values
in the object?"<em>

To add a new key/value pair to an object we can just use `. + {key: value}` e.g.

```bash
$ jq -c '.[] | . + {"key3": [.key1, .key2]}' input.json
{"key1":"a","key2":"b","omg":"lol","key3":["a","b"]}
{"key1":"c","key2":"d","key3":["c","d"]}
```

This however gets rid of our original array. We can just wrap the whole expression
inside `[]` to get back an array:

```jq
$ jq '[.[] | . + {"key3": [.key1, .key2]}]' input.json
[
  {
    "key1": "a",
    "key2": "b",
    "omg": "lol",
    "key3": [
      "a",
      "b"
    ]
  },
  {
    "key1": "c",
    "key2": "d",
    "key3": [
      "c",
      "d"
    ]
  }
]
```

## update

Another way to write it would be to use `|=` which "updates" the left hand side
so we don't lose the original structure:

```jq
$ jq '.[] |= . + {"key3": [.key1, .key2]}' input.json
[
  {
    "key1": "a",
    "key2": "b",
    "omg": "lol",
    "key3": [
      "a",
      "b"
    ]
  },
  {
    "key1": "c",
    "key2": "d",
    "key3": [
      "c",
      "d"
    ]
  }
]
```

## map()

Finally there is `map()` which processes an array and produces an array. This allows us
to simplify the expression down to:

```jq
$ jq 'map(. + {"key3": [.key1, .key2]})' input.json
[
  {
    "key1": "a",
    "key2": "b",
    "omg": "lol",
    "key3": [
      "a",
      "b"
    ]
  },
  {
    "key1": "c",
    "key2": "d",
    "key3": [
      "c",
      "d"
    ]
  }
]
```

So `map(expression)` is just another way of writing `[ .[] | expression ]`
