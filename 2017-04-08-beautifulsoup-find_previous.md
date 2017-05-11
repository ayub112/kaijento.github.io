---
title: "BeautifulSoup: find_previous()"
date:  2017-04-08 07:59:00 +0100
tags:
 - python
 - beautifulsoup
---

Given the following sample html we want to create a list of stories
where each story is a dict with the keys `title`, `date` and `content` 
using `BeautifulSoup`:

```html
<html>
  <head>
    <title>Newssite</title>
  </head>
  <body>
    <h1>This is news story 1</h1>
    <h3>28.01 2017</h3>
    <p class="content">
      This is a news story
    </p>
    <h1>This is news story 2</h1>
    <h3>29.01 2017</h3>
    <p class="content">
    This is another news story
    </p>
  </body>
</html>
```

We could find each `<h1>` tag move forwards however as this is sample HTML
we've been given it is possible there may be other `<h1>` tags that are not
story titles.

In that case it seems like the `<p class="content">` tags would be a better
choice. We can use `soup.find_all('p', {'class': 'content'})` to isolate those
or we can use the shorthand syntax `soup('p', 'class')`

The default method if none is given is `find_all()` and if a 2nd argument is
given to either find method it is matched against the class attribute.

If you prefer to work with CSS selectors you could also use `soup.select('p.content')`

So let's take a look at some code:

`list-of-stories.py`

```python
import json
from   bs4 import BeautifulSoup

html = '''
<html>
  <head>
    <title>Newssite</title>
  </head>
  <body>
    <h1>This is news story 1</h1>
    <h3>28.01 2017</h3>
    <p class="content">
      This is a news story
    </p>
    <h1>This is news story 2</h1>
    <h3>29.01 2017</h3>
    <p class="content">
    This is another news story
    </p>
  </body>
</html>
'''

soup = BeautifulSoup(html, 'html5lib')

stories = [
    dict(
        title   = p.find_previous('h1').text,
        date    = p.find_previous('h3').text,
        content = p.text.strip()
    ) 
    for p in soup('p', 'content')
]

print(json.dumps(stories, indent=4, sort_keys=True))
```

The declaration of `stories` make look a little odd it's a <em>dict comprehension</em>
inside a <em>list comprehension</em>. It may be more common to see it written as:

```python
stories = []
for p in soup('p', 'content'):
    story = {
        'title'  : p.find_previous('h1').text,
        'date'   : p.find_previous('h3').text,
        'content': p.text.strip()
    }
    stories.append(story)
```

So we find each `<p class="content">` tag and from there we search backwards
using `find_previous()` for the `<h1>` title tag and the `<h3>` date tag.

This is what the output looks like:

```bash
$ python list-of-stories.py 
[
    {
        "content": "This is a news story", 
        "date": "28.01 2017", 
        "title": "This is news story 1"
    }, 
    {
        "content": "This is another news story", 
        "date": "29.01 2017", 
        "title": "This is news story 2"
    }
]
```

The `json` module was only used to <em>"pretty-print"</em> the resulting list
of dicts.
