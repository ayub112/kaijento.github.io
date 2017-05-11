---
title: "Web scraping: dotabuff.com"
date:  2017-04-09 10:59:00 +0100
tags:
 - python
 - webscraping
 - beautifulsoup
 - regex
 - pandas
---

The task is to extract out the stats from the <em>WORST VERSUS</em> table
on a [Defense of the Ancients](https://dotabuff.com) hero page using 
Python. The person did not want to use `BeautifulSoup` as they are 
only learning Python and feel it would be better for learning purposes
to complete the task using <em>"vanilla Python"</em>.

The example URL we've been given is of `Abaddon`:

[https://www.dotabuff.com/heroes/abaddon](https://www.dotabuff.com/heroes/abaddon)

Let's open up the page in our browser and check out the `Inspector` tab:

![](/img/1491732263-dotabuff.com.png)

The HTML for the table we're after looks something like this:

```html
<section>
  <header>Worst Versus<small>This Week
    <div class="more">
      <a href="/heroes/abaddon/matchups"> <i class="fa fa-plus-square"></i> more </a>
    </div></small>
  </header>
  <article>
    <table>
      <thead>
        <tr>
          <th colspan="2">Item</th><th class="cell-large">Advantage<span class="r-none-mobile"><i ...
          <th class="cell-large">Win Rate</th>
          <th class="cell-large">Matches</th>
        </tr>
      </thead>
      <tbody>
        <tr data-link-to="/heroes/anti-mage">
          <td class="cell-icon"><div ...
          <td><a class="link-type-hero" href="/heroes/anti-mage">Anti-Mage</a></td>
          <td>-4.54% ...
          <td>54.25% ...
```

We've truncated the full contents for formatting purposes but we can see that there are no
distinct markers for the `<table>` in question. Normally you may see a specific 
`class` or `id` attribute for the different sections to allows us to easily locate it.

## regexing html

According to <em>"the internet"</em> you're never supposed to use regular expressions 
and you're never never supposed to use regular expressions on HTML. 

So I guess it's time for some <em>"Epic lulz"</em>:

```python
>>> import re, requests
>>> 
>>> url = 'https://www.dotabuff.com/heroes/abaddon'
>>> r   = requests.get(url, headers={'user-agent': 'Mozilla/5.0'})
>>> 
>>> html = r.text
>>>
>>> section = re.search(r'<section><header>Worst Versus.+?</section>', html).group()
>>> headers = re.findall(r'<th[^>]*>([^<]+)', section)
>>> headers
['Item', 'Advantage', 'Win Rate', 'Matches']
>>> rows = re.findall(r'<td>(?:<a[^>]*>)?([^<]+)', section)
>>> rows[:8]
['Anti-Mage', '-4.57%', '54.15%', '48,635', 'Undying', '-3.60%', '52.26%', '10,975']
>>> for n in range(0, len(rows), 4):
...     rows[n:n+4]
... 
['Anti-Mage', '-4.57%', '54.15%', '48,635']
['Undying', '-3.60%', '52.26%', '10,975']
['Outworld Devourer', '-3.02%', '56.10%', '23,404']
['Lone Druid', '-2.97%', '64.06%', '7,340']
['Io', '-2.91%', '66.74%', '4,092']
['Lycan', '-2.81%', '52.12%', '6,978']
['Doom', '-2.75%', '61.36%', '12,941']
['Broodmother', '-2.49%', '64.96%', '5,559']
['Lina', '-2.39%', '56.13%', '52,591']
['Invoker', '-2.32%', '56.28%', '86,323']
```

So the regex `<section><header>Worst Versus.+?</section>` isolates the 
<em>WORST VERSUS</em> `<section>` for us. If there were nested `<section>` tags
this would not work correctly which is one of the reasons that parsing HTML
with regex is usually discouraged.

The data we're after comes after every plain `<td>` tag inside the section.
The only one that differs is the Item name as that has an `<a ...>` tag.

The `(?:<a[^>]*>)?` part of our second regex handles this case. `<a[^>]*>` 
matches any `<a>` tag the `(?:)` makes the group non-capturing and the 
trailing `?` makes it optional meaning that it may or may not be present.

Finally we have `([^<]+)` we matches and captures <em>not "<" 1 or more times</em>
which is how you can match up to the `<` character i.e. the start of the closing
tag.

Because `re.findall()` gives us back a flat list of matches we need to process
it 4 items at a time which we can do with a combination of `range()` and 
slicing.

## BeautifulSoup

So that gives us the data needed, how could we approach it using a parser
such as `BeautifulSoup`?

```python
>>> from bs4 import BeautifulSoup
>>>
>>> soup = BeautifulSoup(r.content, 'html5lib')
```

As the particular section / table we're after doesn't have any distinct 
attributes we could just use number position. It is the fifth table
on the page (this was discovered by trial and error i.e. by
inspecting `soup('table')[0]`, `soup('table')[1]`, etc.)

```python
>>> table = soup('table')[4]
```

Note here we're using the shorthand form of `soup.find_all('table')` which
you can use if you prefer.

In cases where you can't isolate a tag by a distinctive attribute another 
useful approach can be to use the `string` argument to the find
methods. This allows you to match based on the content of a tag. Here
we could search for the `Worst Versus` string inside the `<header>` tag:

```python
>>> soup.find('header', string='Worst Versus')
>>>
```

Hmm... pls y u no result?

```python
>>> for header in soup('header'):
...     header.string
... 
'Lane Presence'
'Time Played'
'Dotabuff Plus'
```

Only `3` results that does not look right...

```python
>>> len(soup('header'))
14
```

We should have `14` ...

## .string vs .text 

```python
>>> for header in soup('header'):
...     header.string, header.text
... 
('Lane Presence', 'Lane Presence')
(None, 'Trending Guide2017-04-07 more')
(None, 'Most Popular Ability Build16.01% Build Rate more')
(None, 'Most Used ItemsThis Week more')
(None, 'Best VersusThis Week more')
(None, 'Worst VersusThis Week more')
(None, 'Win RateThis Week more')
(None, 'Pick RateThis Week more')
(None, 'Meta TrendsThis Week more')
(None, 'Talent Usage more')
(None, 'Hero Rankings more')
('Time Played', 'Time Played')
(None, 'Hero Attributes more')
('Dotabuff Plus', 'Dotabuff Plus')
```

The issue is that because some of the `<header>` tags contain children tags
`.string` will be empty as the content is contained inside the children.

```html
<header>Lane Presence</header>
...
<header>Worst Versus<small>This Week
```

`.text` or `.get_text()` on the other hand will give you the combined 
string content of a tag and all of its children.

So this means we cannot use the `find('header', string=...)` approach. It is still
possible to locate it based on the `Worst Versus` string but it's rather
verbose:

```python
>>> next(header for header in soup('header') if next(header.strings) == 'Worst Versus')
<header>Worst Versus<small>This Week<div...
```

`.strings` gives you a generator of all the contained strings so we can use 
`next(headers.strings)` to test against the first one. 

I've not checked but `.text` mentioned above probably just 
does `''.join(tag.strings)` in the background.

From the `<header>` tag we could `find_next('table')` to then get the correct table. 
We will just rely on the position approach for the rest of this example by taking the 
<em>"fifth table"</em>.

```python
>>> table = soup('table')[4]
```

## pandas.read_html()

So now that we have the table in question how do we extract the stats?

One possible approach could be to load it up as a `pandas` DataFrame
using `pandas.read_html()`

`read_html()` can take a URL, file object or even a string and will
return a list of dataframes containing the parsed tables found. The
reason we don't pass it our URL directly is that it uses `urllib` to
fetch the data and there is no way (currently) to change the 
`user-agent` header for the request and in this case `dotabuff.com` 
blocks the default `urllib` and `requests` user-agent strings.

Let's pass it `str(table)` and see how it does:

```python
>>> import pandas
>>> pandas.read_html(str(table))
[   Item          Advantage Win Rate Matches  Unnamed: 4
0   NaN          Anti-Mage   -4.59%  54.14%       49012
1   NaN            Undying   -3.56%  52.31%       11056
2   NaN  Outworld Devourer   -3.00%  56.12%       23590
3   NaN         Lone Druid   -2.99%  64.04%        7386
4   NaN                 Io   -2.91%  66.75%        4111
5   NaN              Lycan   -2.87%  52.06%        7026
6   NaN               Doom   -2.71%  61.43%       13045
7   NaN        Broodmother   -2.44%  65.00%        5600
8   NaN               Lina   -2.39%  56.13%       53033
9   NaN              Meepo   -2.38%  61.92%       10465]
```

The problem is that we have `4` `<th>` tags but each row contains `5`
`<td>` tags. The first `<td>` holding the Item image:

```html
<tr data-link-to="/heroes/anti-mage"><td class="cell-icon">
```

We could remove the first `<td>` tag from each `<tr>` inside the `<tbody>`:

```python
>>> for tr in table.tbody('tr'):
...     tr.td.decompose()
... 
>>> pandas.read_html(str(table))
[                Item Advantage Win Rate  Matches
0          Anti-Mage    -4.59%   54.14%    49012
1            Undying    -3.56%   52.31%    11056
2  Outworld Devourer    -3.00%   56.12%    23590
3         Lone Druid    -2.99%   64.04%     7386
4                 Io    -2.91%   66.75%     4111
5              Lycan    -2.87%   52.06%     7026
6               Doom    -2.71%   61.43%    13045
7        Broodmother    -2.44%   65.00%     5600
8               Lina    -2.39%   56.13%    53033
9              Meepo    -2.38%   61.92%    10465]
```

We can use `.decompose()` to remove / destroy a tag. There is also `extract()` 
which returns the removed tag if needed. Written out fully using `find_all()`
and `find()` calls it would look like:

```python
for tr in table.find('tbody').find_all('tr'):
    tr.find('td').decompse()
```

I prefer to use the shorthand variations though.

It wasn't specified what was to be done to the data but after it's loaded into
a DataFrame you could use methods such as `to_csv()`, `to_excel()` or 
`to_sql()` depending on the exact goal.

```python
>>> import sys
>>> 
>>> stats = pandas.read_html(str(table))[0]
>>> stats.to_csv(sys.stdout, index=False)
```

Gives us:

```
Item,Advantage,Win Rate,Matches
Anti-Mage,-4.59%,54.14%,49012
Undying,-3.56%,52.31%,11056
Outworld Devourer,-3.00%,56.12%,23590
Lone Druid,-2.99%,64.04%,7386
Io,-2.91%,66.75%,4111
Lycan,-2.87%,52.06%,7026
Doom,-2.71%,61.43%,13045
Broodmother,-2.44%,65.00%,5600
Lina,-2.39%,56.13%,53033
Meepo,-2.38%,61.92%,10465
```

## processing a table with BeautifulSoup

You could also extract the data just using `BeautifulSoup` and some
<em>list comprehensions</em>.

The headers:

```python
>>> table = soup('table')[4]
>>> [ th.text for th in table('th') ]
['Item', 'Advantage', 'Win Rate', 'Matches']
```

The rows:

```python
>>> [ [ td.text for td in tr('td') ] for tr in table.tbody('tr') ]
[['Anti-Mage', '-4.59%', '54.14%', '49,012'], ['Undying', '-3.56%', '52.31%', '11,056'], ...

```

From there you could easily use the `csv` module to export it to CSV or perhaps
even use `dict(zip(headers, row))` to build a list of dict and export it to JSON

```python
>>> headers = [ th.text for th in table('th') ]
>>> rows    = [ [ td.text for td in tr('td') ] for tr in table.tbody('tr') ]
>>>
>>> stats   = [ dict(zip(headers, row)) for row in rows ]
>>> print(json.dumps(stats, indent=4, sort_keys=True))
[
    {
        "Advantage": "-4.59%",
        "Item": "Anti-Mage",
        "Matches": "49,012",
        "Win Rate": "54.14%"
    },
    {
        "Advantage": "-3.56%",
        "Item": "Undying",
        "Matches": "11,056",
        "Win Rate": "52.31%"
    },
    ...
```

It all depends on what you want to do with the data.
