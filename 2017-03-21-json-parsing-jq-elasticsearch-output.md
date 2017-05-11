---
title: "JSON parsing: jq elasticsearch output"
tags:
 - jq
 - json
 - bash
 - elasticsearch
---

<em>I'm trying to use jq on elasticsearch output to produce something like this: ...</em>

## Input

Here is the sample input we are given:

`input.json`

```json
{
    "_shards": {
        "failed": 0,
        "successful": 1,
        "total": 1
    },
    "aggregations": {
        "top_fields": {
            "buckets": [
                {
                    "cardinality": {
                        "value": 259,
                        "value_as_string": "0.0.1.3"
                    },
                    "doc_count": 1747,
                    "key": "Ecuador"
                },
                {
                    "cardinality": {
                        "value": 1,
                        "value_as_string": "0.0.0.1"
                    },
                    "doc_count": 6,
                    "key": "Finland"
                }
            ],
            "doc_count_error_upper_bound": 0,
            "sum_other_doc_count": 26
        }
    },
    "hits": {
        "hits": [],
        "max_score": 0.0,
        "total": 3084
    },
    "timed_out": false,
    "took": 8
}
```

## Desired output 

```json
{
  "date": "YYYYMMDD",
  "countries": [
    {
      "key": "Ecuador",
      "doc_count": 1747
    },
    {
      "key": "Finland",
      "doc_count": 6
    },
    ...
  ]
}
```

## --arg

The `--arg` option can be used to define jq variables with given values so we define the jq
variable `date` whose value is the output of the date command.

```jq
$ date +%Y%m%d
20170321
$ jq --arg date `date +%Y%m%d` '{ "date": $date, "countries": [ .aggregations.top_fields.buckets[] | { key, doc_count } ] }' input.json
{
  "date": "20170321",
  "countries": [
    {
      "key": "Ecuador",
      "doc_count": 1747
    },
    {
      "key": "Finland",
      "doc_count": 6
    }
  ]
}
```

Variables in jq are accessed like in bash so `$date` gets replaced with the value we gave it using `--arg`.

## { key }

Normally you would see `{"key": .value}` used to extract a particular field:

```bash
$ jq -c '.aggregations.top_fields.buckets[] | { "key": .key, "doc_count": .doc_count }' input.json
{"key":"Ecuador","doc_count":1747}
{"key":"Finland","doc_count":6}
```

If however, you want to keep the original key name you can just use `{ key }`

```bash
$ jq -c '.aggregations.top_fields.buckets[] | { key, doc_count }' input.json
{"key":"Ecuador","doc_count":1747}
{"key":"Finland","doc_count":6}
```

In other words: `{ doc_count }` is equivalent to `{"doc_count": .doc_count}`

The surrounding `[]` just groups each object together into a single list:

```bash
$ jq -c '[ .aggregations.top_fields.buckets[] | { key, doc_count } ]' input.json
[{"key":"Ecuador","doc_count":1747},{"key":"Finland","doc_count":6}]
```
