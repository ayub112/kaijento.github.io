---
title: "Web scraping: NASA Image of the Day"
date:  2017-05-06 07:42:50 +0100
tags:
 - python
 - webscraping
 - requests
 - json
 - curl
---

The goal is trying to *"scrape"* images from 
[NASA's Image of the Day page](https://www.nasa.gov/multimedia/imagegallery/iotd.html) 
using Python's `BeautifulSoup` module. The images 
are there when I look in the `Inspector` tab but they're
not there when I fetch the page using `requests`. What am
I doing wrong?

## "Developer Tools"

I'm using `Firefox` as my browser with Javascript disabled
(using the `NoScript` extension) so if I open
the page I see *"nothing"*.

![](/img/1494053422-nasa-noscript.png)

The page does not work without Javascript enabled. Neither
`requests` nor the `urllib` modules can execute Javascript 
which is why there are no images present when we use them 
to fetch the HTML.

One option is to debug the HTTP requests being made
by the Javascript and try to replicate them with `requests`.

To debug HTTP requests we can view the `Network` tab. 

You may have heard of the `Inspector` tab which you can 
access by *right-clicking* on a page and selecting 
`Inspect Element`.

![](/img/1494053653-nasa-inspect-element.png)

From there we can select the `Network` tab.

![](/img/1494053054-nasa-network-xhr.png)

We've filtered the requests being shown to only `XHR` which 
stands for `XMLHttpRequest`. These are the type of requests
that are made via Javascript to fetch data.


If you *right-click* on an individual request you can (amongst
other things) `Copy as cURL` which will give you the full 
`curl` command to use to replicate the request. You can also
view more details about the request using the panel on the right.

![](/img/1494053100-nasa-copy-as-curl.png)

Only requests that are made while the `Network` tab is open
are shown so you may need to refresh the page.

So if we copy the URL of the selected request we see it's made to

[https://www.nasa.gov/api/1/query/ubernodes.json?unType%5B%5D=image&routes%5B%5D=1446&page=0&pageSize=24](https://www.nasa.gov/api/1/query/ubernodes.json?unType%5B%5D=image&routes%5B%5D=1446&page=0&pageSize=24)

(We chose this request as it was the first in the list with a `200` response.)

The `GET` params should mostly be self documenting

* `page=0` is the first page of results
* `pageSize=24` specifies the number of results per page
* `unType[]=image` selects only *"images"*

I tried changing the `pageSize` to a larger number but it seems `24`
is the limit (and the default if omitted). Not sure what `routes[]` 
represents however it too can be omitted and the request still works.

If you open this link in your browser you will see something like

```javascript
{
  "ubernodes": [
    {
      "type": "ubernode",
      "promoDateTime": "2017-05-05T09:31:00-04:00",
      "nid": "401247"
    },
    {
      "type": "ubernode",
      "promoDateTime": "2017-05-04T10:04:00-04:00",
      "nid": "401205"
    },
    ...
  ],
  "meta": {
    "total_rows":  3216
  }
}
```

So it's `JSON` data consisting of a list of *ubernodes*. I'm assuming `3216` 
is the total number of *pages* but it could also be the total number of 
*images* meaning `3216 / 24` *pages*.

Each *ubernode* has an `id` (although it's called `nid` here) the first being `401247`.

You may notice this number as it is present in the next request made in the `Network` tab.
[https://www.nasa.gov/api/1/record/node/401247.json](https://www.nasa.gov/api/1/record/node/401247.json)

Opening this link in a browser doesn't seem to load but if you fetch it using `curl` 
(or `requests`) you should see it looks something like

```javascript
{
  "images": [
    {
      "fid": 556374,
      "uid": "324",
      "filename": "pia11241_reduced.jpg",
      "uri": "public://thumbnails/image/pia11241_reduced.jpg",
      "filemime": "image/jpeg",
      "filesize": "18009851",
      "timestamp": "1493922525",
      "uuid": "19f072fa-e4a2-4274-99bd-58a7263fde2f",
      "alt": "360-degree scene from the Mastcam on NASA's Curiosity Mars rover",
      "title": "",
      "width": "10000",
      "height": "2279",
    }
  ],
  "ubernode": {
    "title": "Panorama with Active Linear Dune in Gale Crater, Mars",
    "nid": "401247",
    "type": "ubernode",
    "changed": "1493994090",
```

Here we can see the `uri` value appears to hold what looks like the URL 
to the image but what is this `public://`?

Well the answer is I'm not exactly sure. 

If you click on one of the *Images Of The Day* it brings up a sort of 
slideshow and allows you to download the image. So we can do this with 
the `Network` tab open to see the actual location of the image.

So, we click on the first image of the day but before clicking the download
button we click the *"basket"* in the `Network` tab to clear the current
list of requests. 

We don't need them anymore and they are just cluttering up our view. We 
also deselect the `XHR` filter as we're not sure if that is the method
that will be used. We click the download button and see just a single
request is made.

![](/img/1494088453-nasa-dl-img.png)

If we copy the URL we see it goes to 
[https://www.nasa.gov/sites/default/files/thumbnails/image/pia11241_reduced.jpg](https://www.nasa.gov/sites/default/files/thumbnails/image/pia11241_reduced.jpg)

This means we can replace `public:/` from the `uri` value in the `JSON` data
with `https://www.nasa.gov/sites/default/files` in order to build the full
URL to the image.

If we use `curl -I` to send a `HEAD` request to the URL we can see it's correct.

```bash
$ curl -I 'https://www.nasa.gov/sites/default/files/thumbnails/image/pia11241_reduced.jpg'
HTTP/1.1 200 OK
Date: Sat, 06 May 2017 16:46:00 GMT
Content-Type: image/jpeg
Content-Length: 18009851
Connection: keep-alive
x-amz-id-2: bihPWnpoFwaerzUuXH+wWti2wWJFDjrdZp7BtpRZQXBXGa139pCCKgXiz4d1TULAkTB9AMpV2iM=
x-amz-request-id: 28DF2A5EDF3C42A4
Cache-Control: max-age=3600
Server: AmazonS3
Age: 3308
Last-Modified: Fri, 05 May 2017 14:22:33 GMT
Expires: Sat, 06 May 2017 16:50:52 GMT
X-UA-Compatible: IE=edge
Content-Security-Policy: frame-ancestors 'self' *.nasa.gov
Strict-Transport-Security: max-age=31536000
```

So now we know what requests are to be made now we will move on to
generating the code to perform them.

## Code

First we will show some of the output.

```
https://www.nasa.gov/sites/default/files/thumbnails/image/pia21603.jpg
https://www.nasa.gov/sites/default/files/thumbnails/image/pia21608.jpg
https://www.nasa.gov/sites/default/files/thumbnails/image/pia21606.jpg
https://www.nasa.gov/sites/default/files/thumbnails/image/pia21605.jpg
https://www.nasa.gov/sites/default/files/thumbnails/image/pia21390.jpeg
[...]
https://www.nasa.gov/sites/default/files/thumbnails/image/pia20527-1041.jpg
https://www.nasa.gov/sites/default/files/thumbnails/image/iss050e014337.jpg
https://www.nasa.gov/sites/default/files/thumbnails/image/pia21494_eagle_esp_050177_1780.jpg
https://www.nasa.gov/sites/default/files/thumbnails/image/final_earth_obs_fleet06hw.2100_1920x1080.jpg
```

And the code that generated it.

```python
import requests

api    = 'https://www.nasa.gov/api/1/'
public = 'https://www.nasa.gov/sites/default/files'

with requests.session() as s:
    s.headers['user-agent'] = 'Mozilla/5.0'

    # first 2 pages
    for page in range(2):
        r = s.get(api + 'query/ubernodes.json', 
                params={'page': page, 'unType[]': 'image'})
        for ubernode in r.json()['ubernodes']:
            nid = ubernode['nid']
            r = s.get(api + 'record/node/{}.json'.format(nid))
            for image in r.json()['images']:
                uri = image['uri'].replace('public:/', public, 1)
                print(uri)
```

## Code breakdown

When making multiple requests with `requests` you'll usually want to use a 
[session object](http://docs.python-requests.org/en/latest/user/advanced/#session-objects)
to maintain *"state"* and keep track of cookies.

You'll also pretty much always want to change the default `User-Agent` header 
which we set here to `Mozilla/5.0`

We use `range()` to loop through a range of page numbers. We've just used `0` and `1`
here to get the first `2` pages of results.

```python
>>> list(range(2))
[0, 1]
```

The API call takes a `GET` request so we the `params` argument to pass the data as 
opposed to using the `data` argument if it was a `POST` request.

`requests` comes with [.json()](http://docs.python-requests.org/en/latest/user/quickstart/#json-response-content)
for decoding a `JSON` response (which I imagine just calls `json.loads()` *"under the hood"*).

This takes a `JSON` string and turns it into a Python structure.

```javascript
{
  "ubernodes": [
    {
      "type": "ubernode",
      "promoDateTime": "2017-05-05T09:31:00-04:00",
      "nid": "401247"
    },
``` 

The `JSON` we're working with is (in this instance) also valid Python - 
it's basically like a *dict of lists* which is exactly what we get when we 
call `.json()` on it.

So `r.json()['ubernodes']` is a *list of dicts* which we loop over and
extract the `nid` value.

We then use the `nid` to build the URL to request the *ubernode* record 
details from the API.

Similar to the previous structure `r.json()['images']` is a list
(even though it looks like they all have a single element) which 
we loop through to extract the `uri` value and replace the 
`public:/` part of it to generate the full URL.

Finally we are just printing out the URL but you could
fetch them and [save the images](http://docs.python-requests.org/en/latest/user/quickstart/#raw-response-content)
to disk if that is your goal.

