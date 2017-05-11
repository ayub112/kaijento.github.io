---
title: "JSON parsing: jq group_by() max_by() sort_by()"
tags:
 - jq
 - json
 - bash
---

<em>I have some JSON entries and I would like to filter out those
with the greatest score. If there are multiple entries with the same
score I want to print them all.</em>


## jq

To parse JSON data from within bash chances are you'll want
to be using [jq](https://stedolan.github.io/jq/).

`scores.json`

```javascript
{ 
    "rounds": [
        [
            {"name": "alice", "score": 3},
            {"name": "bob",   "score": 2},
            {"name": "ted",   "score": 3}
        ],
        [
            {"name": "alice", "score": 2},
            {"name": "bob",   "score": 3},
            {"name": "ted",   "score": 1}
        ]
    ]
}
```

## sort_by()

`sort_by(.field)` can be used to sort by a particular field. 

```bash
$ jq -c '.rounds[] | sort_by(.score)' scores.json
[{"name":"bob","score":2},{"name":"alice","score":3},{"name":"ted","score":3}]
[{"name":"ted","score":1},{"name":"alice","score":2},{"name":"bob","score":3}]
```

The `-c` option forces jq to print each object on a single line.

When using `sort_by()` though, there is no way of knowing where the first <em>"max"</em> 
score is so let's try `max_by()`:

```bash
$ jq -c '.rounds[] | max_by(.score)' scores.json
{"name":"ted","score":3}
{"name":"bob","score":3}
```

`max_by()` only gives a single result even if there is a <em>"tie"</em>.

## group_by()

Another of the `by()` functions is `group_by()`. It can be given a field to group on and
will put same value matches into individual arrays (<em>"groups"</em>). The groups
will be sorted by the value of the given field in ascending order.

```jq
$ jq '.rounds[0] | group_by(.score)' scores.json
[
  [
    {
      "name": "bob",
      "score": 2
    }
  ],
  [
    {
      "name": "alice",
      "score": 3
    },
    {
      "name": "ted",
      "score": 3
    }
  ]
]
```

There are <strong>2</strong> arrays inside the result: one for the group of scores with `2` and 
another for the scores with value `3`. They are sorted in ascending order meaning that
the last item will be the <em>"max"</em> group. It can be accessed using negative indexing 
i.e. `[-1]`.

```bash
$ jq -c '.rounds[] | group_by(.score)[-1]' scores.json
[{"name":"alice","score":3},{"name":"ted","score":3}]
[{"name":"bob","score":3}]
```

## strings sort separately

It turns out that in the <strong>actual data</strong> the score values were strings 
and <strong>not</strong> numbers like in the sample data. Strings sort differently 
to numbers and because of this the current approach is <em>"broken"</em>.

We'll need to change the score values in order to generate a failing test case.

`scores-strings.json`

```javascript
{ 
    "rounds": [
        [
            {"name": "alice", "score": "10"},
            {"name": "bob",   "score": "10"},
            {"name": "ted",   "score": "2"}
        ],
        [
            {"name": "alice", "score": "1"},
            {"name": "bob",   "score": "3"},
            {"name": "ted",   "score": "10"}
        ]
    ]
}
```

Do we have a failing test case?

```bash
$ jq -c '.rounds[] | group_by(.score)[-1]' scores-strings.json 
[{"name":"ted","score":"2"}]
[{"name":"bob","score":"3"}]
```

We do. The problem is that the string `"10"` sorts before the string `"2"` 
(and `"3"`).

```bash
$ echo '["1", "2", "10"]' | jq -c sort
["1","10","2"]
$ echo '[1, 2, 10]' | jq -c sort
[1,2,10]
```

## tonumber 

We'll need to turn our strings into numbers. This can be done in
jq with `tonumber`:

```bash
$ echo '["1", "2", "10"]' | jq -c '[ .[] | tonumber ] | sort'
[1,2,10]
```

## Return of the sort_by()

Instead of just passing a field to `sort_by()` we can pass an <em>"expression"</em>
that including our newly found `tonumber` to gives us numerical sorting as opposed
to alphabetical sorting:

```bash
$ jq -c '.rounds[] | group_by(.score) | sort_by(.[].score | tonumber)[-1]' scores-strings.json 
[{"name":"alice","score":"10"},{"name":"bob","score":"10"}]
[{"name":"ted","score":"10"}]
```

You may have noticed that we can replace `sort_by()[-1]` with `max_by()`:

```bash
$ jq -c '.rounds[] | group_by(.score) | max_by(.[].score | tonumber)' scores-strings.json 
[{"name":"alice","score":"10"},{"name":"bob","score":"10"}]
[{"name":"ted","score":"10"}]
```

## That's it!

For more information on [jq](https://stedolan.github.io/jq/) check out the 
[manual](https://stedolan.github.io/jq/manual/) and
[Cookbook](https://github.com/stedolan/jq/wiki/Cookbook).

