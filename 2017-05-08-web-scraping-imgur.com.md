---
title: "Web scraping: imgur.com"
date:  2017-05-08 08:02:06 +0100
tags:
 - python
 - webscraping
 - requests
 - beautifulsoup
 - json
 - curl
---

*I'm trying to "scrape" images from Imgur 
using BeautifulSoup and requests but I'm only getting the
first page of results. Why?*

The example URL we're given is [http://imgur.com/r/funny](http://imgur.com/r/funny)

The code uses `requests` to fetch the HTML and `BeautifulSoup`
with `html5lib` to parse it.

If you don't have these install already you can use install them
using `pip` e.g.

`pip install beautifulsoup4 requests html5lib --user`

One thing to note is that [imgur.com has an api](https://api.imgur.com/).
* You use it, take the blue pill—the article ends.
* You take the red pill—you stay in Wonderland, 
and I show you how deep a `JSON` response goes.

Remember: all I'm offering is the truth. Nothing more.

## "Developer Tools"

Usually the first step to take is to debug the page in question
inside your browser using the *"Developer Tools"* (although 
currently it seems to called *"Web Developer"* in `Firefox`) mainly 
the `Inspector` tab and the `Network` tab.

So normally when you scroll down to the end of an *Imgur* results
page it will automatically load the next page of images.

I'm using `Firefox` as my browser (although `Chrome` also has 
*"Developer Tools"*) with Javascript disabled
(using the `NoScript` extension) so when I scroll to the
end of the page I see a *"loading"* icon and the next page
is never loaded.

![](/img/1494227189-imgur-loading.png)

This means that the *"load next page"* functionality
is implemented using Javascript. Neither `requests` 
nor the `urllib` modules can execute Javascript meaning
we will only get the first page of results when
using them to fetch the HTML.

One option is to debug the HTTP requests being made
by the Javascript and try to replicate them with `requests`.

To debug HTTP requests we can view the `Network` tab. 

You may have heard of the `Inspector` tab which you can 
access by *right-clicking* on a page and selecting 
`Inspect Element`.

![](/img/1494227500-imgur-inspect-element.png)

From there we can select the `Network` tab.

![](/img/1494227380-imgur-network-xhr.png)

After opening the `Network` tab I have selected the `XHR` filter
to show only those type of requests. This stands for `XMLHttpRequest`
which is what the requests made by Javascript are classed as. 

As we know the page is using Javascript to fetch the data we know
it will be an `XHR` request so viewing only those requests will
simplify things.

I then enable Javascript (as I have it disabled) and reload the page.
The `Network` tab will only show requests that have been made after it was
opened.

So we can see a `GET` request was made to 
[http://imgur.com/r/funny/new/page/1/hit?scrolled](http://imgur.com/r/funny/new/page/1/hit?scrolled)
and its return type was `HTML`.

If we scroll down further to load the next page of images we see 
another `GET` request is made, this time to
[http://imgur.com/r/funny/new/page/2/hit?scrolled](http://imgur.com/r/funny/new/page/2/hit?scrolled)

All that changes in the URL is the page number i.e. `1 -> 2` and as `1` gives 
us the second page of results we can assume that `0` will gives us the first i.e.

[http://imgur.com/r/funny/new/page/0/hit?scrolled](http://imgur.com/r/funny/new/page/0/hit?scrolled)

This suggests we could loop through a `range()` of page numbers to build the
needed URLs and then parse out the image links from the HTML.

The images on the results page however are just *"preview"* images and not the 
actual content we're looking for. Also, it's not just *"images"* we're dealing 
with as some are *"videos"*, some are *"albums"* of *"images"*.

So let's scroll back up and click on one of the images to open the individual 
image page. Before clicking we will hit click the *"basket"* icon to the left 
of `Inspector` to clear out the `Network` tab to make it easier to focus
on the new requests being made.

![](/img/1494279683-imgur-image-page.png)

We can see a similar request is made to the previous `XHR` requests except `hit?scrolled`
is replaced with `hit.json` and we get back a `JSON` response.

If you click on the `Response` tab over on the right-hand side panel you can view
details about the response.

Another useful feature is that *right-click* on a request and `Copy as cURL`
which will give you the full `curl` command that can be used to replicate the
request which can be very useful for testing things out from the command-line.

![](/img/1494280248-imgur-copy-as-curl.png)

We can add `-o filename` to the end of the command to store the output into `filename`
instead of just printing the result if needed.

Using the exact command is not always necessary but at the very least you will want 
to set the `User-Agent` header which is what we'll do here.

We're also going to pipe the result through `jq` which will *pretty-print* the `JSON`
and finally using shell redirection to write the result to file.

```bash
$ curl -A Mozilla/5.0 'http://imgur.com/r/funny/new/page/0/hit.json' | jq . > imgur.json
```

You could of course use `requests` along with `json.dump()` to do it directly from
Python and there's also [httpie](https://httpie.org/).

## JSON

The `JSON` response received has a structure like

```javascript
{
  "data": [
    {
      "id": 4735008625,
      "hash": "pNuFUQQ",
      "author": "SlimJones123",
      "account_id": null,
      "account_url": null,
      "title": "Abra Kadraba Alakaslam",
      "score": 18352,
      "size": 18087775,
      "views": "18551",
      "is_album": false,
      "album_cover": null,
      "album_cover_width": 0,
      "album_cover_height": 0,
      "mimetype": "image/gif",
      "ext": ".gif",
      "width": 720,
      "height": 720,
      "animated": true,
      "looping": true,
      "reddit": "/r/funny/comments/6a02ma/abra_kadraba_alakaslam/",
      "subreddit": "funny",
      "description": "",
      "create_datetime": "2017-05-05 14:42:36",
      "bandwidth": "312.50 GB",
      "timestamp": "2017-05-08 18:50:03",
      "section": "funny",
      "nsfw": false,
      "prefer_video": true,
      "video_source": "https://www.instagram.com/p/BTW71oaA8DL/",
      "video_host": null,
      "num_images": 1,
```

`data` has `100` entries meaning using `hit.json` gives us
`100` results per page and each entry contains lots of 
information, `hash`, `author`, `views`, etc.

```bash
$ jq '.data | length' imgur.json
100
```

We could also use Python's `json.load()` to get that information.

```python
>>> import json
>>>
>>> with open('imgur.json') as f:
...     j = json.load(f)
...
>>> len(j['data'])
100
```

... or just use `requests` to make the request as mentioned.

```python
>>> import requests
>>> 
>>> url = 'http://imgur.com/r/funny/new/page/0/hit.json'
>>> r   = requests.get(url, headers={'user-agent': 'Mozilla/5.0'})
>>> 
>>> len(r.json()['data'])
100
```

`requests` has a [.json()](http://docs.python-requests.org/en/master/user/quickstart/#json-response-content) 
method on response objects which saves us having to call `json.loads()` manually. 
Also note that HTTP header names are case-insensitive although feel free
to use `User-Agent` if you prefer.

If you're not familiar with `JSON` it's just a *"file format"* that is used
for passing data around. It looks similar to a Python structure and in this
case it's valid Python too.

```python
>>> j = {
...   "data": [
...     {
...       "id": 4735008625,
...       "hash": "pNuFUQQ"
...     }
...   ]
... }
>>> type(j)
<type 'dict'>
```

So we have a `dict` with the key `data` whose value is a `list`.

```python
>>> type(j['data'])
<type 'list'>
```

And this is exactly what we get when from 
from the call to `.json()` or one of the `json.load` methods as
they turn `JSON` data into an equivalent Python structure.

So as mentioned above we're not just dealing with single static images.
If you *"hover"* over the images in the list of results it shows you
the image *"type"* and the amount of views. 

![](/img/1494280811-imgur-alt.png)

There appear to be `3` types

* `image`
* `animated` 
* `gallery`

We can see this information inside the `JSON` response

```javascript
"hash"    : "pNuFUQQ",
"is_album": false,
"animated": true
```

So if `is_album` is `false` and `animated` is `false` it's a
regular single image.

If `is_album` is `false` and `animated` is `true` like in this
example it's a single *"video"*.

```javascript
"hash"    : "pNuFUQQ",
"mimetype": "image/gif",
"ext"     : "gif"
```

It does say it's a `gif` but *Imgur* also serves `mp4`
versions of the `gif` files.


As we have the `hash` and the `ext` to build the URL we 
can simply replace `.gif` with `.mp4`

```bash
$ curl -I http://i.imgur.com/pNuFUQQ.gif
HTTP/1.1 200 OK
Last-Modified: Fri, 05 May 2017 14:42:40 GMT
ETag: "b5467c538aa52d7ea066a7a7bfa6d574"
Content-Type: image/gif
Fastly-Debug-Digest: 3a99bef1064a006c1e74d60486dfbae1791a3220c40c0fe005a08f3b801784bc
cache-control: public, max-age=31536000
Content-Length: 18087775
[...]
```

If we change extension to `mp4` 

```bash
$ curl -I http://i.imgur.com/pNuFUQQ.mp4
HTTP/1.1 200 OK
Last-Modified: Fri, 05 May 2017 14:42:38 GMT
ETag: "56a70bd6a18458c2a95c33934df5a692"
Content-Type: video/mp4
Fastly-Debug-Digest: 95f433cb7680d8ce3e9289d820027d124861898b9723658660d494906c2eb2e7
cache-control: public, max-age=31536000
Content-Length: 903448
[...]
```

Note the substantial difference in the file size `903448` vs. `18087775` 
bytes. We will just use the `mp4` extension when we encounter a `gif`.

So that is `image` and `animated` types taken care of what about `album`?

I looked around to located an `album` with a large number of images and found
[http://imgur.com/r/funny/rtYyh](http://imgur.com/r/funny/rtYyh)

![](/img/1494307287-imgur-load-more-images.png)

If we look at the `Network` tab as we click on the `Load more images` button we
can see the `XHR` request being made.

![](/img/1494307453-imgur-gallery-xhr.png)

The request is made to 
[http://imgur.com/ajaxalbums/getimages/rtYyh/hit.json](http://imgur.com/ajaxalbums/getimages/rtYyh/hit.json)
and we get back a `JSON` response with details about the images contained in the album.
Note that the `hash` (i.e. `rtYyh`) is passed along in the `URL` itself.

We'll use the `curl | jq` combo from earlier to get a *pretty-printed* version of the
`JSON` response saved to disk.

```bash
$ curl -A Mozilla/5.0 'http://imgur.com/ajaxalbums/getimages/rtYyh/hit.json' | jq . > imgur-gallery.json
```

[httpie](https://httpie.org/) mentioned earlier is a *"curl-like"* tool
written in Python that gives you a single `http` command which you could
use instead.

```bash
$ http --pretty format http://imgur.com/... User-Agent:Mozilla/5.0 > imgur-gallery.json
```

Either way the `JSON` response has a structure like

```javascript
{
  "data": {
    "count": 26,
    "images": [
      {
        "hash": "5MXcg9i",
        "title": "",
        "description": null,
        "width": 701,
        "height": 943,
        "size": 6597,
        "ext": ".png",
        "animated": false,
        "prefer_video": false,
        "looping": false,
        "datetime": "2017-05-03 06:00:48"
      },
      {
        "hash": "otDOcL6",
        "title": "",
```

So we have the `count` of images in the album and the
list of images with each `hash` and `ext` meaning we can 
build their URL e.g. `http://i.imgur.com/hash.ext`

Do recall that we got `100` results from our `page/N/hit.json` which
would suggest there is a limit of `100` results per call. If an 
album has multiple `Load more images` buttons there is probably a 
similar *"page"* type syntax for the `getimages/hash/hit.json` call
however without debugging such a page we cannot be sure how it works.

## Code

So now that we know what requests are being made we can attempt
to replicate them with Python.

We will first show the output.

```bash
$ python imgur.py | tee imgur.log
http://i.imgur.com/rNM1Z0V.jpg
http://i.imgur.com/SOnDLwR.mp4
http://i.imgur.com/n3PXWRe.mp4
http://i.imgur.com/9i1aYWN.jpg
http://i.imgur.com/JaeMo6M.mp4
http://i.imgur.com/sH4DqXA.jpg
http://i.imgur.com/uw9ppv4.png
http://i.imgur.com/xF0mdmc.mp4
http://i.imgur.com/pNuFUQQ.mp4
http://i.imgur.com/k4j6ZNZ.jpg
http://i.imgur.com/Ac5p6fB.jpg
http://i.imgur.com/2wdzJkM.jpg
http://i.imgur.com/HP3nCsV.jpg
http://i.imgur.com/90p9w6i.jpg
http://i.imgur.com/3V9qV6w.jpg
http://i.imgur.com/ZxfEVSF.jpg
http://i.imgur.com/CH0FWLj.jpg
http://i.imgur.com/vUfHVz2.jpg
http://i.imgur.com/1Ici9Si.jpg
http://i.imgur.com/ZSyEt89.jpg
[...]
$ wc -l imgur.log
152 imgur.log
```

So from the `100` entries we ended up with `152` *"images*" due to some
of them being albums containing multiple *"images"*.

Here is the code.

```python
import requests

imgur     = 'http://i.imgur.com/{}{}'
page_api  = 'http://imgur.com/r/funny/new/page/{}/hit.json'
album_api = 'http://imgur.com/ajaxalbums/getimages/{}/hit.json'

with requests.session() as s:
    s.headers['user-agent'] = 'Mozilla/5.0'
    
    url = page_api.format(0)

    j = s.get(url).json()
    for entry in j['data']:
        if entry['ext'] == '.gif':
            entry['ext'] = '.mp4'
        if entry['is_album']:
            url = album_api.format(entry['hash'])
            j   = s.get(url).json()
            for image in j['data']['images']:
                if image['ext'] == '.gif':
                    image['ext'] = '.mp4'
                url = imgur.format(image['hash'], image['ext'])
                print(url)
        else:
            url = imgur.format(entry['hash'], entry['ext'])
            print(url)
```

So firstly you'll notice there is no `BeautifulSoup`. 

This is because we're kind of making *"API calls"* directly and 
getting the data in `JSON` meaning there is no `HTML` involved
at all.

The `{}` in our `imgur` variable is a *placeholder* for use with 
[str.format()](https://docs.python.org/3/library/stdtypes.html#str.format).

Each `{}` gets replaced with the corresponding argument passed to
the `format()` call e.g.

```python
>>> imgur = 'http://i.imgur.com/{}{}'
>>> imgur.format('hash', '.ext')
'http://i.imgur.com/hash.ext'
```

Obviously in this example we could also use `imgur + hash + ext` as
we just need to append to the end but perhaps a better example of
`format()` is when you need variable data *"inside"* a string.

```python
>>> page_api = 'http://imgur.com/r/funny/new/page/{}/hit.json'
>>> page_api.format(0)
'http://imgur.com/r/funny/new/page/0/hit.json'
```

When making multiple requests with `requests` you'll usually want to use a 
[session object](http://docs.python-requests.org/en/latest/user/advanced/#session-objects)
to maintain *"state"* and keep track of cookies.

You'll also pretty much always want to set the `User-Agent` header 
which we set here to `Mozilla/5.0` as the default `User-Agent` will
often be blocked.

So we `get()` the first page of results from `page/0/hit.json` however
you could use a loop to process multiple pages.

```python
for n in range(3):
    url = page_api.format(n)
    ...
```

We then loop through each entry changing any `.gif` extension to `.mp4` 
but you can of course skip this part if you wish.

If the `entry['is_album']` we then generate the URL to the 
`ajaxalbums/getimages` *"API call"* and process the resulting 
`['data']['images']` list.

There is some slight code duplication here which suggests we
could create a function but we'll leave that for now.

The final goal is probably to save the images to disk which you could 
implement [using requests](http://docs.python-requests.org/en/latest/user/quickstart/#raw-response-content)
instead of just printing the URLs.
