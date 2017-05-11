---
title: "Web scraping: reddit.com"
date:  2017-04-11 13:25:28 +0100
tags:
 - python
 - requests
 - webscraping
 - beautifulsoup
 - json
 - praw
---

<em>How do I scrape the front page of a subreddit using Python?</em>

<em>Scraping a subreddit</em> seems to be a commonly requested task
 and people usually attempt to do it using `urllib` or `requests` 
combined with `lxml` or `BeautifulSoup`.

Reddit does have an [official API](https://www.reddit.com/dev/api) and 
there is a Python interface to it in [PRAW](https://praw.readthedocs.io/). 
You do however, need to register an account and use OAuth with this 
approach.

There is also another option. Reddit allows you to add a `.json` 
extension to the end of your request and will give you back 
the data in JSON format which can then be processed using a
JSON parser such as the `json` module.

It is sort of like they provide unauthorized access to their JSON API
with this method.

```python
>>> import requests
>>> url = 'http://reddit.com/r/learnpython/new/.json'
>>> r   = requests.get(url, headers={'user-agent': 'Mozilla/5.0'})
>>>
>>> len(r.json()['data']['children'])
25
```

Reddit blocks certain user-agent strings so we must spoof it.

## r.json()

`r.json()` just does a `json.loads()` in the background which turns
a JSON string into a Python structure. We get back a nested structure 
of dicts and lists.

```python
>>> type(r.json())
<type 'dict'>
>>> type(r.json()['data'])
<type 'dict'>
>>> r.json()['data'].keys()
['modhash', 'children', 'after', 'before']
```

To view a pretty-printed version of the structure we can use `json.dumps()`
e.g. `print(json.dumps(r.json(), indent=4, sort_keys=True))`

You will get a lot of output running it directly on `r.json()` so it may 
make sense to write it out to a file (`json.dump()`) instead.

After looking at the structure we can see that the posts are available inside
the `['data']['children']` list. `['data']` also holds some other information.

```python
>>> r.json()['data']['after']
't3_64o6gh'
```

These `before` and `after` values are used for result page navigation
when you click on the `next` and `prev` buttons. To get to the next
page we can pass `after=t3_64o6gh` as a GET param.

```python
>>> next_page_url = url + '?&after=' + r.json()['data']['after']
>>> next_page = requests.get(page2_url, headers={'user-agent': 'Mozilla/5.0'})
>>> next_page.json()['data']['children'][0]['data']['url']
'https://www.reddit.com/r/learnpython/comments/64o5yx/help_breakdown_list_comprehension_example/'
```

When making multiple requests however, you will usually want to use a 
[session object](http://docs.python-requests.org/en/latest/user/advanced/#session-objects).

So each post is a dict and the important information is available inside the `data` key:

```python
>>> posts = r.json()['data']['children']
>>> 
>>> import json
>>> print(json.dumps(posts[0], indent=4, sort_keys=True))
{
    "data": {
        "approved_by": null, 
        "archived": false, 
        "author": "HolyCoder", 
        "author_flair_css_class": null, 
        "author_flair_text": null, 
        "banned_by": null, 
        "brand_safe": true, 
        "clicked": false, 
        "contest_mode": false, 
        "created": 1491943248.0, 
        "created_utc": 1491914448.0, 
        "distinguished": null, 
        "domain": "self.learnpython", 
[...]
```

I've truncated the output here but the ones people are usually after are `author`,
`selftext`, `title` and `url`

```python
>>> posts[0]['data']['url']
'https://www.reddit.com/r/learnpython/comments/64qkav/efficiency_of_an_algorithm/'
>>> posts[0]['data']['title']
'Efficiency of an algorithm'
```

It's pretty annoying having to use `['data']` all the time so we could have 
instead done:

```python
>>> posts = [ post['data'] for post in r.json()['data']['children'] ]
>>> posts[0]['url']
'https://www.reddit.com/r/learnpython/comments/64qkav/efficiency_of_an_algorithm/'
```

## r/aww

So let's say you wanted to "scrape" all the newest "images" from 
[r/aww](http://reddit.com/r/aww/new/) for teh cuddlez you could:

```python
>>> r = requests.get('http://www.reddit.com/r/aww/new/.json', headers={'user-agent': 'Mozilla/5.0'})
>>> for post in r.json()['data']['children']:
...     post['data']['url']
... 
'https://youtu.be/nJRf-fJNdJ4'
'https://i.redd.it/jmctfmktixqy.jpg'
'http://imgur.com/gallery/k5UvK'
'https://i.redd.it/q0ybf3nlixqy.jpg'
'http://i.imgur.com/JoF5FNd.jpg'
'http://i.imgur.com/NI5GuJf.gifv'
'http://i.imgur.com/UHg5RbU.jpg'
'https://i.redd.it/5jwksp64hxqy.jpg'
'http://i.imgur.com/Bninome.gifv'
'http://i.imgur.com/gs8rRg4.jpg'
'https://i.redd.it/m6xpbtuogxqy.jpg'
'https://i.redd.it/gwenjc8dgxqy.jpg'
'http://i.imgur.com/zLmZJTc.gifv'
'http://i.imgur.com/7ihgzyx.gifv'
'http://imgur.com/gallery/jjr6C'
'http://imgur.com/AkdxXuT.gifv'
'https://gfycat.com/UnpleasantBothJunco'
'https://i.redd.it/hk9y3kb8fxqy.jpg'
'http://imgur.com/ADLfEwY'
'https://i.redd.it/wfn3t1b5fxqy.jpg'
'https://i.redd.it/wa44h0zaexqy.jpg'
'https://i.redd.it/viy7gp1cexqy.jpg'
'https://i.redd.it/a1nw9bcydxqy.png'
'https://i.redd.it/wmbn7lf2owqy.jpg'
'https://i.redd.it/yestv4i5dxqy.jpg'
```

These URLs would require further processing though as not all of them are 
direct links and not all of them are images however it seems to be a
common request with the image dedicated subreddits.

## BeautifulSoup

You can of course just request the regular URL and process the HTML
 with a parser such as `BeautifulSoup`:

```python
>>> r = requests.get('http://reddit.com/r/aww/new/', headers={'user-agent': 'Mozilla/5.0'})
>>> soup = BeautifulSoup(r.content, 'html5lib')
>>> for div in soup.select('div.thing'):
...     div['data-url']
... 
'https://youtu.be/nJRf-fJNdJ4'
'https://i.redd.it/jmctfmktixqy.jpg'
'http://imgur.com/gallery/k5UvK'
'https://i.redd.it/q0ybf3nlixqy.jpg'
'http://i.imgur.com/JoF5FNd.jpg'
'http://i.imgur.com/NI5GuJf.gifv'
'http://i.imgur.com/UHg5RbU.jpg'
'https://i.redd.it/5jwksp64hxqy.jpg'
'http://i.imgur.com/Bninome.gifv'
'http://i.imgur.com/gs8rRg4.jpg'
'https://i.redd.it/m6xpbtuogxqy.jpg'
'https://i.redd.it/gwenjc8dgxqy.jpg'
'http://i.imgur.com/zLmZJTc.gifv'
'http://i.imgur.com/7ihgzyx.gifv'
'http://imgur.com/gallery/jjr6C'
'http://imgur.com/AkdxXuT.gifv'
'https://gfycat.com/UnpleasantBothJunco'
'https://i.redd.it/hk9y3kb8fxqy.jpg'
'http://imgur.com/ADLfEwY'
'https://i.redd.it/wfn3t1b5fxqy.jpg'
'https://i.redd.it/wa44h0zaexqy.jpg'
'https://i.redd.it/viy7gp1cexqy.jpg'
'https://i.redd.it/a1nw9bcydxqy.png'
'https://i.redd.it/wmbn7lf2owqy.jpg'
'https://i.redd.it/yestv4i5dxqy.jpg'
```

Which to me at least looks like a simpler way of getting the data compared to
the JSON response parsing approach.

As already mentioned Reddit does have an API with
[rules / guidelines](https://github.com/reddit/reddit/wiki/API#rules) and if 
you're wanting to do some large-scale interaction with Reddit you should
probably go that route and use one of the extensive API interfaces for
your language such as [PRAW](https://praw.readthedocs.io/) in the case of
Python.
