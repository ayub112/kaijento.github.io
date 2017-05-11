---
title: "JSON parsing: jq modifying arrays"
date:  2017-03-23 23:27:45 +0100
tags:
 - jq
 - json
---

We've been given a JSON configuration file of some sort. It contains
environment definitions for servers. Each server has an array of
shell variables. The task is to search through the array for a particular
variable definition and modify it.

So the question could be worded as <em>"How do I change the value of a variable
inside JSON with jq?"</em>

## Input

`env.json`

```json
{
  "top1": {
    "prop": "value",
    "environment": [
      "VAR2=text2",
      "VAR1=text1"
    ]
  },
  "top2": {
    "prop": "value",
    "environment": [
      "VAR1=text3",
      "VAR2=text4"
    ]
  },
  "top3": {
    "prop": "value",
    "environment": [
      "VAR5=text5"
    ] 
}
```

## Desired output

```json
{
  "top1": {
    "prop": "value",
    "environment": [
      "VAR2=text2",
      "VAR1=ABC"
    ]
  },
  "top2": {
    "prop": "value",
    "environment": [
      "VAR1=ABC",
      "VAR2=text4"
    ]
  },
  "top3": {
    "prop": "value",
    "environment": [
      "VAR5=text5"
    ] 
  }
}
```

So as we can see we cannot rely on the index as its position can change and it 
may not be present at all.

```bash
$ jq -c '.[].environment' env.json
["VAR2=text2","VAR1=text1"]
["VAR1=text3","VAR2=text4"]
["VAR5=text5"]
```

To select certain items based on a condition we can use `select()`. What we want to do is
test if any element starts with `VAR1=` for which jq provides us `startswith()`.

```bash
$ jq '.[].environment[] | select(startswith("VAR1="))' env.json
"VAR1=text1"
"VAR1=text3"
```

So that finds our `VAR1` definitions how do we replace them? Let's just try assigning to them:

```bash
$ jq '.[].environment[] | select(startswith("VAR1=")) = "VAR1=ABC"' env.json
"VAR2=text2"
"VAR1=ABC"
"VAR1=ABC"
"VAR2=text4"
"VAR5=text5"
```

Hmmm... so it does replace them but we're not getting our original structure?
Well the issue is that our command:

`.[].environment[] | select(startswith("VAR1=")) = "VAR1=ABC"` 

gets parsed as

`.[].environment[] | (select(startswith("VAR1=")) = "VAR1=ABC")` 

but we want

`(.[].environment[] | select(startswith("VAR1="))) = "VAR1=ABC"`

So we will have to use `()` manually to give the correct grouping:

```bash
$ jq '(.[].environment[] | select(startswith("VAR1="))) = "VAR1=ABC"' env.json
{
  "top1": {
    "prop": "value",
    "environment": [
      "VAR2=text2",
      "VAR1=ABC"
    ]
  },
  "top2": {
    "prop": "value",
    "environment": [
      "VAR1=ABC",
      "VAR2=text4"
    ]
  },
  "top3": {
    "prop": "value",
    "environment": [
      "VAR5=text5"
    ]
  }
}
```

jq to the rescue.

```bash
$ echo ♥ jq
♥ jq
```
