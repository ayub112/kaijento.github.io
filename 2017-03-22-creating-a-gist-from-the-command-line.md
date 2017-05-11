---
title: "creating a gist from the command-line"
date: 2017-03-22 06:53:42 +0100
tags:
 - python
 - requests
 - json
 - bash
 - github
---

<strong>GIST 4 CLIFE!</strong>

You can read public gists and create anonymous gists without any authentication
using the Github API.

Let's look at the [docs](https://developer.github.com/v3/gists/#create-a-gist):

![](/img/1490134235-gist-api.png)

So the API call expects JSON encoded data. The filename extension in `"file1.txt"` is
used to detect the type of syntax highlighting. `".txt"` will not use syntax 
highlighting but we could use `".py"` instead to set it to Python.

## requests

Python comes with `urllib2` (Python 2) and `urllib.request` (Python 3) but the recommended
HTTP client library is [requests](http://docs.python-requests.org/). There is even a
GitHub API v3 example in the 
[requests docs](http://docs.python-requests.org/en/latest/user/quickstart/#more-complicated-post-requests).

One could say that this is an example of <em>"sending an API request in Python"<em>.

So let's give it a try:

```python
>>> import requests
>>>
>>> url = 'https://api.github.com/gists'
>>> r   = requests.post(url, json={'files':{'file1.txt':{'content':'hi'}}})
>>> r.json()['html_url']
'https://gist.github.com/bb3b66a670955d401273a2f4ff8007c2'
```

As there is the `json=` <em>"helper"</em> for sending requests which performs a
`json.dumps()` behind the scenes, response objects have the `.json()` method 
which performs a `json.loads()` to turn the JSON string into a Python structure.

## json.dumps(..., indent=4)

If you'd like to view the full response you use `json.dumps()` to
pretty-print it:

```python
>>> import json
>>> print(json.dumps(r.json(), indent=4))
...
    "description": null, 
    "truncated": false, 
    "url": "https://api.github.com/gists/bb3b66a670955d401273a2f4ff8007c2", 
    "created_at": "2017-03-21T21:20:20Z", 
    "html_url": "https://gist.github.com/bb3b66a670955d401273a2f4ff8007c2", 
...
```

We've truncated the output but you can try it out for yourself.

## os.isatty()

So let's try to build something we can call from the command-line.

`os.isatty(0)` is a way to check if your code has been called in a pipeline
or if it has had input redirected into it. It will return `False` in both of
these cases. This can be useful when building command-line tools.

```bash
$ python -c 'import os; print("Yes" if os.isatty(0) else "No")'
Yes
$ python -c 'import os; print("Yes" if os.isatty(0) else "No")' < filename
No
$ echo moo | python -c 'import os; print("Yes" if os.isatty(0) else "No")' 
No
```

If `os.isatty(0)` is `True` we will process `sys.argv` for a filename to read.
If it is `False` we will read from `sys.stdin` to get the data from the 
pipeline or redirection.

## gist.py

`gist.py`

```python
import os, requests, sys

url = 'https://api.github.com/gists'
ext = 'txt'

if os.isatty(0):
    filename = sys.argv[1]
    with open(filename) as fh:
        content = fh.read()
        if '.' in filename:
            ext = filename.split('.')[-1]
else:
    content = sys.stdin.read()

r = requests.post(url, json={
    'files': {
        'file1.' + ext: {
            'content': content
        }
    }
})

print(r.json()['html_url'])
```

We've set the default extension to `txt`. If we look in `sys.argv[1]` for a filename
and it contains `.` we assume it has an extension and we `split()` it out.

```bash
$ python gist.py gist.py
https://gist.github.com/9be8e23c234ee3a55abbd1333435524c
```

Did it work?

![](/img/1490134411-gist.png)

It did. It also extracted the filename extension successfully to give us syntax
highlighting.

## Improvements?

Well we don't check that we were given a filename and if we were we don't check if
it is valid. `len(sys.argv) == 2` would check that we were passed a single filename.

Checking if a file is there before calling `open()` is a race-condition so it's 
usually better to `try` the open and `catch` the exception if it is raised.

One may want to use 
[pygments.lexers.guess_text()](http://pygments.org/docs/api/#pygments.lexers.guess_lexer) 
to try to guess the type of the data if there is no filename (or extension) given. This
could allow you to syntax highlight in such cases.

```python
>>> pygments.lexers.guess_lexer('import sys\nprint(\'o hai\')')
<pygments.lexers.PythonLexer>
```

You could use [argparse](https://docs.python.org/3/howto/argparse.html) or
[click](http://click.pocoo.org/) to build the command-line interface. 

Finally, if you did not want to put this code into its own file you could create
a shell function instead. 

## gist as a bash function 

You have probably encountered aliases before e.g. `alias ll='ls -l'`

When it comes to complex commands that contain their own quoting though it can become painful:

```bash
$ alias gist='python -c '\''print("\"Hello.\"")'\'''
$ gist
"Hello."
```

Using a function removes the need for the first layer of quoting:

```bash
$ unalias gist
$ gist () { python -c 'print("\"Hello.\"")'; }
$ gist
"Hello."
```

Functions are much more powerful though, they are essentially <em>"inlined"</em> scripts:

```bash
$ wrapper () { arg=$1; shift; echo cmd --arg1 "$arg" --arg2 --arg3 "$@"; }
$ wrapper a b c d
cmd --arg1 a --arg2 --arg3 b c d
```

So Python's `-c` option allows to pass code in a string and because we've used single quotes 
to surround our code the simplest thing to do is convert all single quotes within our code
to double quotes.

```bash
gist () {

python -c '
import os, requests, sys

url = "https://api.github.com/gists"
ext = "txt"

if os.isatty(0):
    filename = sys.argv[1]
    with open(filename) as fh:
        content = fh.read()
        if "." in filename:
            ext = filename.split(".")[-1]
else:
    content = sys.stdin.read()

r = requests.post(url, json={
    "files": {
        "file1." + ext: {
            "content": content
        }
    }
})

print(r.json()["html_url"])
' "$@"
}
```

`"$@"` here are all the arguments passed to the function which we send on to Python so they are
available in `sys.argv`

You can use `'\''` to <em>"escape"</em> a single quote within surrounding single quotes but I suppose
technically you could say that's not what you're doing:

```terminal
$ echo '$single'\''$(quote)'
$single'$(quote)
```

So what you have here are 3 separate <em>"words"</em>:

* `'$single'`
* `\'`
* `'$(quote)'`

So `'\''` means you're actually closing the first `'` then using `\'` then opening a new `'`

It is possible to avoid the quoting issues by replacing `-c` with `python -` and a heredoc however
that introduces another issue with regards to the reading of stdin. It can be remedied by using
process substituion a lá `python <(cat <<\.` but that topic probably warrants its own discussion.
