---
title: "Web scraping: steamcommunity.com"
date:  2017-04-01 06:49:48 +0100
tags:
 - python
 - requests
 - beautifulsoup
 - scrapy
 - webscraping
---

Given a URL to a steamcommunity.com group page we would like to access
the profile page of each member. The example URL given was 
[http://steamcommunity.com/groups/KeyVendorNet](http://steamcommunity.com/groups/KeyVendorNet).

We will start by opening the page in our browser and viewing the
`Inspector` tab. You can access this by *right-clicking* and selecting
`Inspect Element`. I'm using `Firefox` here but the same should 
apply for `Chrome`.

![](/img/1494227500-imgur-inspect-element.png)

You can also access it from the menu (*Web Developer / Developer Tools*) 
or through keyboard shortcuts.

![](/img/1491025694-steamcommunity.png)

So the HTML for the first group member looks like:

```html
<div class="member_block_content  online">
  <div><a class="linkFriend" href="http://steamcommunity.com/id/jambozx">Jambo | B: skins 73%+ &gt; CS.DEALS</a></div>
  <div class="friendSmallText">Online</div>
</div>
```

Let's use `requests` to fetch the HTML and `BeautifulSoup` with `html5lib` 
to fetch parse it. You can install these using 
`pip install beautifulsoup4 requests html5lib --user` if you have not
already.

```python
>>> import requests
>>> from   bs4 import BeautifulSoup
>>>
>>> url = 'http://steamcommunity.com/groups/KeyVendorNet#members'
>>> r   = requests.get(url, headers={'user-agent': 'Mozilla/5.0'})
>>>
>>> soup = BeautifulSoup(r.content, 'html5lib')
>>>
>>> soup.select_one('.linkFriend')
>>>
```

Hmm.. no result let's see if the `href` is in the response we got:

```python
>>> 'http://steamcommunity.com/id/jambozx' in r.text
True
```

What about the class name?

```python
>>> 'linkFriend' in r.text
False
```

## Looks can be deceiving

So the HTML response we've received differs from the DOM representation we see in the `Inspector` 
tab. In this instance it is because Javascript is enabled in the web browser which is fetching 
the group member data to populate the page. `requests` does not execute Javascript so we don't 
get the same result.

Let's inspect the page navigation area:

![](/img/1491044149-steamcommunity-paging.png)

The HTML looks like:

```html
<div class="group_paging">
  <div class="pageLinks">
    <span class="pagebtn disabled">&lt;</span>&nbsp;1&nbsp;&nbsp;
    <a class="pagelink" href="http://steamcommunity.com/groups/KeyVendorNet#members/?p=2">2</a>
    &nbsp;&nbsp;
    <a class="pagelink" href="http://steamcommunity.com/groups/KeyVendorNet#members/?p=3">3</a>
    &nbsp;...&nbsp;
    <a class="pagelink" href="http://steamcommunity.com/groups/KeyVendorNet#members/?p=911">911</a>
    &nbsp;
    <a class="pagebtn" href="http://steamcommunity.com/groups/KeyVendorNet#members/?p=2">&gt;</a>
    </div>
    1 - 51 of 46,447 Members 
</div>
```

So to get to page 2 it's passing `p=2` as a GET parameter in the request, let's try pass `p=1`
to our first page request:

```python
>>> url = 'http://steamcommunity.com/groups/KeyVendorNet#members/?p=1'
>>> r   = requests.get(url, headers={'user-agent': 'Mozilla/5.0'})
>>> 
>>> 'http://steamcommunity.com/id/jambozx' in r.text
True
>>> 'linkFriend' in r.text
False
```

Still no luck. Let's open up the `Network` tab and click on the page 2 button to see what request
is being made.

## The Network Tab

![](/img/1491027094-steamcommunity-network.png)

We're filtering the `XHR` category which stands for `XMLHttpRequest`. This is generally where you 
will find the requests that are fetching the data via Javascript. More often than not the response
type of these requests will be `JSON` but in this case the API endpoint is returning `HTML`.

So it's making a request to:

    http://steamcommunity.com/groups/KeyVendorNet/members/?p=2&content_only=true

Note the `/` after the group name as opposed to the `#` we see in the URL we clicked on. 
Let's fetch page 1 using this URL to see if it works:

```python
>>> url = 'http://steamcommunity.com/groups/KeyVendorNet/members/?p=1'
>>> r   = requests.get(url, headers={'user-agent': 'Mozilla/5.0'})
>>> 
>>> 'linkFriend' in r.text
True
>>> soup = BeautifulSoup(r.content, 'html5lib')
>>> soup.select_one('.linkFriend')
<a class="linkFriend" href="http://steamcommunity.com/id/jambozx">Jambo | B: skins 73%+ &gt; CS.DEALS</a>
>>> soup.select_one('.linkFriend')['href']
'http://steamcommunity.com/id/jambozx'
```

Okay so now we're getting somewhere.

`.linkFriend` is a CSS Selector. It returns any tag that contains the word `linkFriend` in their `class` 
attribute. We could have used `a.linkFriend` to limit the search to only `<a>` tags but there are no
other uses of this class attribute in the HTML. 

There are various ways to isolate particular items using `BeautifulSoup`:

```python
>>> soup.find('a', 'linkFriend')
<a class="linkFriend" href="http://steamcommunity.com/id/jambozx">Jambo | B: skins 73%+ &gt; CS.DEALS</a>
>>> soup.find('a', 'linkFriend')['href']
'http://steamcommunity.com/id/jambozx'
```

`find(tag, value)` is shorthand for `find(tag, {'class': 'value'})` i.e. it defaults to matching against 
the class attribute. 

Do note that like the *CSS Selector* approach it will test if `value` is a exists as a *"word"* in the
`class` attribute definition.

```python
>>> BeautifulSoup('<div class="foo bar">', 'html5lib').find('div', 'bar')
<div class="foo bar"></div>
>>> BeautifulSoup('<div class="foo bar">', 'html5lib').find('div', 'bar')['class']
['foo', 'bar']
```

`select_one()` returns the first match like `find()`. `select()` returns a list of matches similar to 
`find_all()`.

```python
>>> len(soup.select('.linkFriend'))
51
```

If you notice from the navigation area it says `1 - 51 of 46,447 Members` meaning there
are `51` members on te page which is the result we got.

So now we know how to get the profile pages of each member on a page, how do we get all
the pages? Well we could extract the URL from the next page link however we know that
to request page `N` of the results we simply use the `p=N` parameter. This means we could 
just extract the last page number and use `range()` to loop over all of the page numbers.

## Looping through the pages

The page links are all `<a class="pagelink" ...` tags and we want the last one. There is
no `select_last()` so we can `select()` them all and using `[-1]` to index the list to get 
the last element.

```python
>>> soup.select('.pagelink')[-1]
<a class="pagelink" href="http://steamcommunity.com/groups/KeyVendorNet#members/?p=912">912</a>
>>> soup.select('.pagelink')[-1].text
'912'
```

We can use `int()` to convert it into a number to use within `range()` e.g.

```python
>>> last_page = soup.select('.pagelink')[-1].text
>>>
>>> for number in range(1, int(last_page) + 1):
        ....
```

In this example we have `912` pages to process just to get the profile page
URLs of each member. Then you would need to make `46,447` requests to process
each profile page. I personally would classify this type of task as "crawling"
and for such cases using a tool such as [Scrapy](http://doc.scrapy.org/) would
be a good idea.

If we were to do this manually with a for loop using `requests` each request
would be made sequentially. `Scrapy` automatically does parallel requests. Also,
how would we handle errors? retries? rate-limiting? Once again `Scrapy` can handle
all of this for you and more.

## Scrapy

Let's take a look at how we would do it with `Scrapy`:

`steamcommunity.py`

```python
import scrapy

group = 'KeyVendorNet'

class SteamCommunity(scrapy.Spider):
    name = 'Steam Community'

    base_url = 'http://steamcommunity.com/groups/{}/members/'.format(group)
    base_url += '?p={}'

    start_urls = [
        base_url.format(1)
    ]

    def parse(self, response):
        last_page = response.css('.pagelink::text').extract()[-1]
        #for n in range(2, int(last_page) + 1):
        for n in range(1, 10):
            yield scrapy.Request(
                self.base_url.format(n), callback=self.extract_members)

    def extract_members(self, response):
        for href in response.css('.linkFriend::attr(href)'):
            yield { 'href': href.extract() }
```

In the simplest form we can use the `scrapy runspider` command to run our "spider", "crawler" or
whatever you wish to call it. You define a `class` that inherits from `scrapy.Spider`. It must
have a `name` and it can have `start_urls` which a list of URLs to process initially.

Each URL in `start_urls` is fetched and the result is passed to the `parse()` method (or <em>"callback"</em>)
which you are to define. To use CSS Selectors we call the `css()` method on the `response` object.
The syntax differs slightly to `BeautifulSoup` as we can extract text and attributes using the
CSS3 pseudo-elements e.g. `::text` or `::attr(name)`.

So `.pagelink::text` extracts the text content of each tag whose `class` attribute contains the
"word" `pagelink`. `css()` returns a `Selector` object which we must call `extract()` on to actually 
access the actual data which we then index using `[-1]` to get the last element.

We have the `for` loop commented out because I don't want to run 900+ requests for demonstration
purposes. So in our for loop we `yield` a `scrapy.Request()` object. All this does is that it
fetches the URL we construct and `callback=` specifies what function/method to call and pass
the result to.

So in this example we are yielding:

    http://steamcommunity.com/groups/KeyVendorNet/members/?p=1
    http://steamcommunity.com/groups/KeyVendorNet/members/?p=2
    http://steamcommunity.com/groups/KeyVendorNet/members/?p=3
    [...]
    http://steamcommunity.com/groups/KeyVendorNet/members/?p=9

...and passing the results to `extract_members()` inside which we use a CSS Selector to 
get the profile URLs as we did in our initial `BeautifulSoup` example.

Finally we `yield` the dict `{ 'href': href.extract() }`. If we were extracting multiple
items it would perhaps look more "obvious" what was going on e.g. 

```python
yield {
    'name' : foo,
    'level': bar,
    'href' : baz
}
```

We `yield` a dict as it links header/field names to values which `Scrapy` can then use
in the final output e.g. to put column headers into a CSV file.

If you have never seen `yield` before we can just think of it like `return` for now
except that it allows you to "return" multiple times. If you'd like actual
details you can read about
[<em>generators</em> here](https://docs.python.org/3/tutorial/classes.html#generators).

As well as `css()` you can use XPath expressions via `xpath()` if you prefer.

So let's run it with `scrapy runspider` we can use `-s` to modify the global settings
which we do here to change the `user-agent` header. We use `-o` to specify an output file.

Depending on the extension you give `Scrapy` will use the appropriate 
[FEED EXPORTER](https://doc.scrapy.org/en/1.3/topics/feed-exports.html#feed-exporters)
to format the output. We'll choose `CSV` for this example as it's just a single value
and that will give us a simple list of lines.

## runspider

```bash
$ scrapy runspider steamcommunity.py -s USER_AGENT=Mozilla/5.0 -o links.csv
[...]
2017-04-01 14:15:26 [scrapy.core.engine] INFO: Closing spider (finished)
2017-04-01 14:15:26 [scrapy.extensions.feedexport] INFO: Stored csv feed (459 items) in: links.csv
2017-04-01 14:15:26 [scrapy.statscollectors] INFO: Dumping Scrapy stats:
{'downloader/request_bytes': 3745,
 'downloader/request_count': 10,
 'downloader/request_method_count/GET': 10,
 'downloader/response_bytes': 131536,
 'downloader/response_count': 10,
 'downloader/response_status_count/200': 10,
 'finish_reason': 'finished',
 'finish_time': datetime.datetime(2017, 4, 1, 13, 15, 26, 730336),
 'item_scraped_count': 459,
 'log_count/DEBUG': 470,
 'log_count/INFO': 8,
 'request_depth_max': 1,
 'response_received_count': 10,
 'scheduler/dequeued': 10,
 'scheduler/dequeued/memory': 10,
 'scheduler/enqueued': 10,
 'scheduler/enqueued/memory': 10,
 'start_time': datetime.datetime(2017, 4, 1, 13, 15, 24, 173021)}
2017-04-01 14:15:26 [scrapy.core.engine] INFO: Spider closed (finished)
```

The `item_scraped_count` is `459`. We crawled `9` pages with `51` members per page:

```python
>>> 51 * 9
459
```

The `request` count is `10` as we fetched the first page twice. The reason for this
was to avoid duplicating the `extract_members()` code inside the `parse()` method.
It was simpler to just request the page again instead of special-casing the first
request. 

It saved the `459` results into `links.csv`:

`INFO: Stored csv feed (459 items) in: links.csv`

Instead of simply extracting the URLs we could have yielded `scrapy.Request()` objects
from inside `parse_members()` to actually fetch the profile pages and pass them on
to another callback e.g. `parse_profile()` in which we could have extracted whatever
pieces of information from the profiles were deemed important.

Another possible modification would be to pass the group name as a command line argument.
Doing this you would need to define your own `start_requests` method as described
[here](https://doc.scrapy.org/en/latest/topics/spiders.html#spider-arguments).

The final Scrapy example is 
[on github](https://github.com/kaijento/code/blob/master/webscraping/steamcommunity.com/get-group-members.py).
