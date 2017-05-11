---
title: "Web scraping: requests ~ EVENTTARGET ~ VIEWSTATE"
date:  2017-05-04 09:10:16 +0100
tags:
 - python
 - webscraping
 - beautifulsoup
 - requests
 - lxml
 - scrapy
---

Given the URL [http://www.privataaffarer.se/borsguiden/analyser/](http://www.privataaffarer.se/borsguiden/analyser/)
the goal is to extract or *"scrape"* the stock data from the table on the page
using Python. There are multiple pages of results so we would like to loop
or *"crawl"* through multiple pages of the results.

First we'll open up the URL in our browser and use the `Inspector` tab 
to view the DOM. I'm using Firefox here so I *"right-click"* on the page 
and choose `Inspect Element`. You can also navigate using the menu 
`Tools -> Web Developer -> Inspector`

If you're using Chrome it's currently under 
[Developer Tools](https://developers.google.com/web/tools/chrome-devtools/)
but you could just ask *The Internet* where to find it.

![](/img/1493785856-privataaffarer.se-inspector.png)

## Network

To debug HTTP requests we can view the `Network` tab. So let's switch
to the `Network` tab and then click on the *"page 2"* button under the
stock data table.

![](/img/1493785960-privataaffarer.se-network.png)

Usually we're looking for `POST` requests (the `Method` column) and luckily 
in this case it's the first request that was made. We select the request and 
then use the `Params` tab <small>(over on the right)</small> to view the 
`POST` params that were sent.

The first thing we see is this `__EVENTTARGET` *"variable"*.

It appears these variables used by webapps written in ASP.net for determining 
*"state"* (`__VIEWSTATE`) and also for validating requests (`__EVENTVALIDATION`)

Let's search for `__EVENTTARGET` inside the HTML source and see what we find.

If possible I normally fetch the HTML with the `curl` command (or Python's 
`requests` library) and save it locally to perform searches using my editor e.g.

```bash
$ curl -A Mozilla/5.0 'http://blah.com' -o page.html
```

You can also *"right-click"* on a request in the `Network` tab and select
`Copy as cURL` which will give you the full `curl` command to replicate
the request.

![](/img/1493960470-privataaffarer.se-copy-as-curl.png)

You can add `-o filename` to the end of the command to save the output
to `filename` 

Searching for `__EVENTTARGET` in the HTML finds the following

```html
<input type="hidden" name="__EVENTTARGET" id="__EVENTTARGET" value="" />
<input type="hidden" name="__EVENTARGUMENT" id="__EVENTARGUMENT" value="" />
<input type="hidden" name="__LASTFOCUS" id="__LASTFOCUS" value="" />
<input type="hidden" name="__VIEWSTATE" id="__VIEWSTATE" value="/wEPDwUKLTk4NDEz...
```

`__VIEWSTATE` and `__EVENTVALIDATION` contain **HUGE** strings of what appears 
to be some form of *"hashed data"*. What they contain isn't exactly important
we just know we need to extract their `value` and send them with our request.

`__EVENTTARGET` has an empty `value` in the HTML which doesn't match what our
`POST` request sent but we'll address that in a moment.

If we look at the rest of the `POST` params being sent we see these `ctl00`
type variables.

![](/img/1493788866-privataaffarer.se-post-params.png)

Let's search through the HTML for the pattern
`<input type="hidden" name="ctl00` and see what we find.

The first result we get is

```html
<input type="hidden" name="ctl00$FhMainContent$FhContent$ctl00$analysisFilter$filterParams" id="ctl00.." value="InstrumentIdFilter:...
```

It appears (for this page at least) the `ctl00` variables have their `value`
attibute just like the `__` type variables which we can extract directly
from the HTML.

We mentioned that `__EVENTTARGET` had an empty value in the HTML but in the `POST`
params it had a value beginning with `ctl00` so what exactly is going on?

If we view the HTML for the *"page 2"* button or we *right-click* it and *"Copy Link Location"* we 
will see the `href` value contains

```html
javascript:__doPostBack('ctl00$FhMainContent$FhContent$ctl00$AnalysesCourse$CustomPager$dataPager$ctl01$ctl01','')
```

This `ctl00` value matches the contents of `__EVENTTARGET` in the `POST` request.

If we inspect the *"page 3"* button we see

```html
javascript:__doPostBack('ctl00$FhMainContent$FhContent$ctl00$AnalysesCourse$CustomPager$dataPager$ctl01$ctl02','')
```

All that changes here is `01` to `02` - this is the *"pagination"* works.

* Page 1 of the results is `00`
* Page 2 of the results is `01`
* Page 3 of the results is `02`

... and so on.

When we click on a page button it sets the `__EVENTTARGET` variable to the corresponding `ctl00` page
value (using Javascript) and then posts the form.

Looking again at the `POST` params sent we also notice couple of interesting values.

```
ctl00$FhMainContent$FhContent$ctl00$AnalysesCourse$CustomPager$max:   30
ctl00$FhMainContent$FhContent$ctl00$AnalysesCourse$CustomPager$total: 360
```

This suggests there are `360` pages of results and there are `30` items per
result page. We could try sending a value higher than `30` to see if
we can more results per page (meaning we could send less requests). 
There are usually limits in place though so the only way to find it out would
be by trial and error. We will just stick to using the default value of `30`.

## Code

We'll be using `requests` to fetch the HTML and `BeautifulSoup` with `html5lib`
to parse it. These can be installed using `pip` e.g.

    pip install requests beautifulsoup4 html5lib --user

We will start with some (truncated) example output.

```
Behåll,Electrolux B,Danske Bank Markets,263,1,,02-maj
Sälj,Hexpol B,SEB Equities,98,45,82,02-maj
Köp,Swedish Orphan Biovitrum,Pareto Securities,136,8,200,02-maj
Behåll,Electrolux B,Pareto Securities,263,1,280,02-maj
Behåll,Hexpol B,Kepler Cheuvreux,98,45,102,02-maj
Sälj,Hexpol B,DNB Markets,98,45,89,02-maj
Behåll,Nobia,Danske Bank Markets,91,65,100,02-maj
Köp,Intrum Justitia,UBS,348,392,28-apr
Behåll,SCA B,Danske Bank Markets,297,7,315,28-apr
Behåll,SKF B,Danske Bank Markets,191,2,,28-apr
[...]
```

The code that generated it.

```python
import requests
from   bs4 import BeautifulSoup

url = 'http://www.privataaffarer.se/borsguiden/analyser/'

with requests.session() as s:
    s.headers['user-agent'] = 'Mozilla/5.0'

    r    = s.get(url)
    soup = BeautifulSoup(r.content, 'html5lib')

    target = (
        'ctl00$FhMainContent$FhContent$ctl00'
        '$AnalysesCourse$CustomPager$dataPager$ctl01$ctl{:02d}'
    )

    # unsupported CSS Selector 'input[name^=ctl00][value]'
    data  = { tag['name']: tag['value'] 
                  for tag in soup.select('input[name^=ctl00]') if tag.get('value') }
    state = { tag['name']: tag['value'] for tag in soup.select('input[name^=__]') }

    data.update(state)

    # data['ctl00$FhMainContent$FhContent$ctl00$AnalysesCourse$CustomPager$total']
    last_page = int(soup.find('div', 'custom_pager_total_pages').input['value'])

    # for page in range(last_page + 1):
    for page in range(3):
        data['__EVENTTARGET'] = target.format(page)

        r    = s.post(url, data=data)
        soup = BeautifulSoup(r.content, 'html5lib')

        # unsupported CSS Selector 'tr:not(.tr_header)'
        for tr in soup.select('.analysis_table tr'):
            row = [ td.text.strip() for td in tr('td')[1:-1] ]
            if row:
                print(','.join(row))
```

## Code breakdown

When making multiple requests with `requests` you'll usually want to use a 
[session object](http://docs.python-requests.org/en/latest/user/advanced/#session-objects)
to maintain *"state"* and keep track of cookies.

You'll also pretty much always want to change the default `User-Agent` header 
which we set here to `Mozilla/5.0`

We must first send a `GET` request to the page so that we can extract the needed
parameters to send in our subsequent `POST` requests.

`target` is just a string. 

```python
target = (
    'ctl00$FhMainContent$FhContent$ctl00'
    '$AnalysesCourse$CustomPager$dataPager$ctl01$ctl{:02d}'
)
```

Python will implictily join adjacent *"strings"*

```python
>>> 'this'    'is'      'one'      'string'
'thisisonestring'
```

To split it over multiple lines we need to add the surrounding `()` which we
do just for code formatting reasons to keep under a certain line length.

```python
>>> ( 'this'     'is'
...   'a'               'large'  '   string'
... )
'thisisalarge   string'
```

The `{:02d}` at the end of `target` is a `.format()` string specifier which 
will be used to *"zero-pad"* the page number in `__EVENTTARGET`

```python
>>> '{:02d}'.format(1)
'01'
>>> '{:02d}'.format(100)
'100'
```

The `2` specifies the minimum length of the result and the leading `0` will
mean it will pad with `0` instead of a *space* character.

```python
>>> '{:03d}'.format(1)
'001'
>>> '{:3d}'.format(1)
'  1'
```

The `d` specifies we want it to be outputted as a *Decimal Integer*. For more
information about `format()` see 
[the docs](https://docs.python.org/3/library/string.html#format-specification-mini-language).

The declaration of `data` may look *"odd"* if you've not seen the syntax before.

```python
data = { tag['name']: tag['value'] 
             for tag in soup.select('input[name^=ctl00]') if tag.get('value') }
```

It is called a *dict comprehension* which is just *"shorthand"* syntax for building
a dict. We could use a *"regular"* `for` loop instead.

```python
data = {}
for tag in soup.select('input[name^=ctl00]'):
    if tag.get('value'):
        key, value = tag['name'], tag['value']
        data[key] = value
```

Feel free to use this form if you prefer but it's good to be aware of what 
comprehensions look like.

We're using [BeautifulSoup's select()](https://www.crummy.com/software/BeautifulSoup/bs4/doc/#css-selectors)
here to find all `<input>` tags whose `name` attribute 
*startswith* `ctl00` i.e.  `select('input[name^=ctl00]')`

The `^=` here represents the *"startswith"* operator within a *CSS Selector*.

There are some matching `ctl00` tags that do not have a `value` attribute
and we want to skip these. We can use the selector `[attribute]` to specify
that `attribute` must exist.  

Combined with `input[name^=ctl00]` we would get `input[name^=ctl00][value]` 
which is a valid selector however `BeautifulSoup` doesn't support it. 

For *"more complete"* CSS Selector support popluar choices include `lxml.html.cssselect()` 
and [parsel](http://parsel.readthedocs.io/en/latest/).

As we cannot use the unsupported selector we add the `tag.get('value')` check 
to the code. What does it do?

You can access the values of an attribute by using *dict indexing* on 
a *beautifulsoup tag object* (you can also access `.attrs` dict directly 
e.g. `tag.attrs[name]`)

If we try to index the `value` key and it's not present we will raise a `KeyError` exception.

```python
>>> print({'name': 'input'}['value'])
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
KeyError: 'value'
```

If you use the `dict.get()` method however it will just return `None` (you can 
however supply `get()` with a default return if desired).

```python
>>> print({'name': 'input'}.get('value'))
None
>>> print({'name': 'input'}.get('value', 'o hai'))
o hai
```

`None` is a *"falsey"* value so `if tag.get('value')` will evaluate to `False`
if there is no `value` attribute present.

## find(find_all()) is False

Some people may ask why dont't you just use `find()` or `find_all()` 
instead of using `select()`? 

It is certainly possible to use `find_all()` in this instance.

```python
soup.find_all(lambda tag: tag and tag.name == 'input' and tag.get('name') and tag['name'].startswith('ctl00'))
```

But who wants to be typing all of that, amirite?

We're extracting all of the `__` variables (e.g. `__VIEWSTATE` etc) and storing them
in the `state` dict. 

```python
state = { tag['name']: tag['value'] for tag in soup.select('input[name^=__]') }
```

It's identical to the `data` declaration apart from the *startswith* condition 
and there is no need to test if there is a `value` attribute present as
they always are (although it wouldn't hurt to add it).

We're then using `data.update(state)` to add all the items from the `state` dict into
the `data` dict. You could think of this as a *"merge"* operation.

```python
>>> data  = {'a': 1, 'b': 2}
>>> state = {'c': 3, 'd': 4}
>>> data.update(state)
>>> data
{'a': 1, 'c': 3, 'b': 2, 'd': 4}
```

We don't use the `state` variable anywhere else so we could have
passed the *dict comprehension* directly to `update()` instead.

```python
data.update(
    { tag['name']: tag['value'] for tag in soup.select('input[name^=__]') }
)
```

## BeautifulSugar

The total number of pages is contained in one of the `ctl00` variables but
we could also extract it from the HTML using `find()` as shown in the code.

```python
soup.find('div', 'custom_pager_total_pages').input['value']
```

This is using some *syntactic sugar* provided by `BeautifulSoup` and is
shorthand for 

```python
find('div', {'class': 'custom_pager_total_pages'}).find('input')['value']
```

If you supply a second argument to `find()` or `find_all()` it defaults
to matching it against the `class` attribute. `find('tag')` can be
replaced with `.tag` as `BeautifulSoup` makes it available as an attribute.
Also, the default method called is `find_all()` if no name is supplied
meaning `find_all('table')` can be shortened to just `('table')`

Now that we have all our `POST` params set up we just need to loop
through the page numbers and set `__EVENTTARGET` then make our request.
We've just looped through the first 3 pages here as an example but you
could loop through them all by using `last_page + 1` in the `range()`
instead just like the commented loop line in the code. It all depends on
how far you want to go back. 

To [send form-encoded data](http://docs.python-requests.org/en/latest/user/quickstart/#more-complicated-post-requests)
 along with our request (`POST` params) we pass a dictionary to the 
`data` argument a lá `s.post(url, data=data)`

You may wonder why we're sending a request for *Page 1* as we already
have that data from our initial `.get()` call. 

The reason for this is that we would have to duplicate the code for 
*"scraping"* the stock data; once for the first page and then again 
inside the `for` loop for the subsequent pages. 

## Scraping tables

The stock data is contained inside a `<table>` tag.

```html
<table class=" table_header analysis_table table">
```

We want to get *"all"* `<tr>` tags inside of this table.
We've used `soup.select('.analysis_table tr')` to do so. 

In a CSS Selector `.word` tests if `word` exists inside the `class`
attribute of an item. We've omitted the tag name here but we could have
used `table.analysis_table` which would state we only want to search
for `<table>` tags but `analysis_table` is unique in the HTML so it's
not needed. How explicit / specific you need to be with your
selectors depends on the HTML you're dealing with.

There are `2` tables on the page, the data we want is in the first one.
This means we could also use `soup.table('tr')` (which is equivalent to
`soup.find('table').find_all('tr')`) to get the same results as the 
`select()` in this instance.

Obviously this assumes the data is always in the first `<table>` found
which may be considered a more *"fragile"* approach compared to 
explicitly testing for the unique class word. 

We said earlier we wanted *all* `<tr>` tags however that is not entirely
true. The first `<tr>` in the table is inside the `<thead>` which contains
the *"headers"* in `<th>` tags.

```html
<tr>
  <th class="stock_status" data-columns="data_stock_course_0">
```

The *"headers"* are also repeated in multiple rows throughout the table.

```html
<tr class="tr_header">
  <th class="stock_status" data-columns="data_stock_course_0" data-priority="1">
```

So when we try to find `<td>` tags inside those `<tr>` tags we will
get an empty list as the result.

```python
>>> [ td.text for td in soup.table.tr('td') ]
[]
>>> [ td.text.strip() for td in soup.table('tr')[-1]('td') ]
['', 'Neutral', 'Investor B', 'Swedbank', '399,6', '405', '25-apr', '']
```

An empty list is *"falsey"* meaning that our `if row:` check will 
only be `True` if there are any `<td>` tags found which in turn has
the side-effect of skipping the the unwanted header *"rows"*.

The `[ ... for ... ]` syntax used here is called a *list comprehension*. 
It is just shorthand syntax for building lists similar to *dict comprehensions*
for building dicts.

It can be written using *"regular"* `for` loop instead.

```python
for tr in soup.select('.analysis_table tr'):
    row = []
    for td in tr('td'):
        row.append(td.text.strip())
```

So we're processing each `<tr>` tag then finding all the `<td>` tags it contains
and extracting their `.text` content and calling `strip()` to remove the surrounding
whitespace.

You may have noticed from the previous result both the first and last entries 
in the resulting list are empty strings. This is why we have `[1:-1]` in the code.

```python
>>> row = ['', 'Neutral', 'Investor B', 'Swedbank', '399,6', '405', '25-apr', '']
>>> row[1:-1]
['Neutral', 'Investor B', 'Swedbank', '399,6', '405', '25-apr']
```

The `[start:end]` syntax is called [slicing](https://docs.python.org/3/tutorial/introduction.html#lists).
It gives us back a new list starting and ending with the given indexes.

Finally we use a simple `print(','.join(row))` to produce the final output.
Please note however that you should use the `csv` module if you actually want to 
produce *proper* CSV data.

## That's it!

So what we have works (the code is also
[on github](https://github.com/kaijento/code/blob/master/webscraping/privataaffarer.se/scrape.py))
however when needing to loop or *"crawl"* through 
multiple pages of results you may want to consider using [Scrapy](http://scrapy.org) 
as it handles a lot of things for you such as parallel requests, error 
handling, request retries, etc. 
