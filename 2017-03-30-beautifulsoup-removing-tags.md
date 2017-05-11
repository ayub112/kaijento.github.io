---
title: "BeautifulSoup: removing tags"
date:  2017-03-30 06:43:38 +0100
tags:
 - python
 - beautifulsoup
 - webscraping
 - csv
---

The task is to extract the Nominal GDP sector composition table from the
[List_of_countries_by_GDP_sector_composition wikipedia page](https://en.wikipedia.org/wiki/List_of_countries_by_GDP_sector_composition)
and convert it to CSV using Python.

![](/img/1490852842-gdp-sector.png)

We could call this an example of *"scraping a wikipedia table"*.

We'll use `requests` for the fetching and `BeautifulSoup` for the parsing:

```python
>>> import csv, json, requests, sys
>>> from   bs4 import BeautifulSoup
>>> 
>>> url  = 'https://en.wikipedia.org/wiki/List_of_countries_by_GDP_sector_composition'
>>> r    = requests.get(url)
>>> soup = BeautifulSoup(r.content, 'html5lib')
>>>
>>> row = [ td.text for td in soup.table('tr')[2]('td') ]
>>> print(json.dumps(row, indent=4))
[
    "1", 
    "\u00a0United States", 
    "7007179469960000000\u266017,946,996", 
    "1.12%", 
    "19.1%", 
    "79.7%", 
    "7005215364000000000\u2660215,364", 
    "7006342787600000000\u26603,427,876", 
    "7007143037560000000\u266014,303,756"
]
```

## find without find()

The table we are after is the first `<table>` tag on the page so we can use `soup.find('table')`
to find it. 

`BeautifulSoup` has some "shorthand" syntax for simple cases of `find()` and `find_all()`:

* `soup.tag` is the same as `soup.find('tag')`
* `soup('tag')` is the same as `soup.find_all('tag')`

This means that: 

`soup.table('tr')[2]('td')`

is equivalent to:

`soup.find('table').find_all('tr')[2].find_all('td')`

Note that the find methods aren't only called on the `BeautifulSoup` object.

```python
>>> type(soup.table)
<class 'bs4.element.Tag'>
```

They can be called on `Tag` objects too to search from a particular starting point:

```python
>>> soup.table.find('a')
<a href="/wiki/List_of_countries_by_GDP_(nominal)" title="List of countries by GDP (nominal)">nominal GDP</a>
>>> soup.table.a
<a href="/wiki/List_of_countries_by_GDP_(nominal)" title="List of countries by GDP (nominal)">nominal GDP</a>
```

So the first `<tr>` in the `<table>` contains the header names and the second contains
the totals so the data we want starts in the third row, hence the `[2]`.

The `[ td.text for td ... ]` syntax is called a **List Comprehension**. It's as if we did:

```python
row = []
for td in soup.table('tr')[2]('td'):
    row.append(td.text)
```

Finally we're using `json.dumps(..., indent=4)` to pretty-print the resulting list for
formatting purposes.

## prettify()

So what is all of this extra data we're seeing? Let's inspect the contents of the `<tr>` tag:

```python
>>> print(soup.table('tr')[2].prettify())
<tr>
 <td>
  1
 </td>
 <td>
  <span class="flagicon">
   <img alt="" class="thumbborder" data-file-height="650" data-file-width="1235" height="12" src="//upload.wikimedia.org/wikipedia/en/thumb/a/a4/Flag_of_the_United_States.svg/23px-Flag_of_the_United_States.svg.png" srcset="//upload.wikimedia.org/wikipedia/en/thumb/a/a4/Flag_of_the_United_States.svg/35px-Flag_of_the_United_States.svg.png 1.5x, //upload.wikimedia.org/wikipedia/en/thumb/a/a4/Flag_of_the_United_States.svg/46px-Flag_of_the_United_States.svg.png 2x" width="23"/>
  </span>
  <a href="/wiki/United_States" title="United States">
   United States
  </a>
 </td>
 <td>
  <span class="sortkey" style="display:none">
   7007179469960000000♠
  </span>
  17,946,996
 </td>
 <td>
  1.12%
 </td>
 <td>
  19.1%
 </td>
 <td>
  79.7%
 </td>
 <td>
  <span class="sortkey" style="display:none">
   7005215364000000000♠
  </span>
  215,364
 </td>
 <td>
  <span class="sortkey" style="display:none">
   7006342787600000000♠
  </span>
  3,427,876
 </td>
 <td>
  <span class="sortkey" style="display:none">
   7007143037560000000♠
  </span>
  14,303,756
 </td>
</tr>
```

Okay so we have these extra `<span>` tags that we do not need. In certain situations
instead of matching what you do want it can be simpler to remove what
you do not want. To remove a tag using `BeautifulSoup` there are 2 options: 
`extract()` and `decompose()`.

## decompose()

`extract()` will return that tag that has been removed and `decompose()` will destroy
it. We would like to remove all of the `<span>` tags and destroy them so we will use
`decompose()`:

```python
>>> tr = soup.table('tr')[2]
>>> for span in tr('span'):
...     span.decompose()
... 
>>> [ td.text for td in tr('td') ]
['1', 'United States', '17,946,996', '1.12%', '19.1%', '79.7%', '215,364', '3,427,876', '14,303,756']
```

So we're done? Well not quite:

```python
>>> [ td.text for td in soup.table('tr')[27]('td') ]
['26', 'Nigeria', '415,080', '17.8%', '25.7%', '54.6%', '73,884', '106,676', '226,634[1]']
```

What is this `[1]`?

```python
>>> soup.table('tr')[27]('td')[-1]
<td>226,634<sup class="reference" id="cite_ref-Product_Report_NBS_1-0"><a href="#cite_note-Product_Report_NBS-1">[1]</a></sup></td>
```

It's a citation link inside an `<a>` tag contained in a `<sup>` tag. This means that
instead of removing just `<span>` tags we want to also remove all `<sup>` tags.

## find_all(list)

`find_all()` also accepts a list meaning that we can use `find_all(['span', 'sup'])`
or the shorthand syntax of `soup(['span', 'sup'])` to find all of them.

```python
>>> writer = csv.writer(sys.stdout)
>>> for tr in soup.table('tr')[2:]:
...     for tag in tr(['span', 'sup']):
...         tag.decompose()
...     writer.writerow([ td.text for td in tr('td') ])
... 
```

Which gives us output that looks like

```python
1,United States,"17,946,996",1.12%,19.1%,79.7%,"215,364","3,427,876","14,303,756"
2,China,"11,007,721",9.0%,40.5%,50.5%,"990,695","4,458,127","5,558,899"
3,Japan,"4,730,300",1.2%,27.5%,71.4%,"56,764","1,300,833","3,377,434"
4,Germany,"3,494,900",0.8%,28.1%,71.1%,"27,959","982,067","2,484,874"
5,United Kingdom,"2,649,890",0.7%,21%,78.3%,"18,549","556,477","2,074,864"
6,France,"2,488,280",1.9%,18.3%,79.8%,"47,277","455,355","1,985,647"
7,India,"2,250,990",17.4%,25.8%,56.9%,"391,672","580,755","1,280,813"
8,Italy,"1,852,500",2%,24.2%,73.8%,"37,050","448,305","1,367,145"
9,Brazil,"1,769,600",5.4%,27.4%,67.2%,"95,558","484,870","1,189,171"
10,Canada,"1,532,340",1.8%,28.6%,69.6%,"27,582","438,249","1,066,509"
11,South Korea,"1,404,380",2.7%,39.8%,57.5%,"37,918","558,943","807,519"
12,Russia,"1,267,750",3.9%,36%,60.1%,"49,442","456,390","761,918"
13,Australia,"1,256,640",4%,26.6%,69.4%,"50,266","334,266","872,108"
14,Spain,"1,252,160",3.3%,24.2%,72.6%,"41,321","303,023","909,068"
15,Mexico,"1,063,610",3.7%,34.2%,62.1%,"39,354","363,755","660,502"
16,Indonesia,"940,953",14.3%,46.9%,38.8%,"134,556","441,307","365,090"
17,Netherlands,"769,930",2.8%,24.1%,73.2%,"21,558","185,553","563,589"
18,Turkey,"755,716",8.9%,28.1%,63%,"67,259","212,356","476,101"
19,Switzerland,"662,483",1.3%,27.7%,71%,"8,612","183,508","470,363"
20,Saudi Arabia,"657,785",2%,66.9%,31.1%,"13,156","440,058","204,571"
21,Argentina,"541,784",10%,30.7%,59.2%,"54,178","166,328","320,736"
22,Taiwan,"519,149",1.3%,32%,66.9%,"6,749","166,128","347,311"
23,Sweden,"517,440",1.8%,26.9%,71.3%,"9,314","139,191","368,935"
24,Belgium,"470,179",0.7%,21.6%,77.7%,"3,291","101,559","365,329"
25,Poland,"467,350",3.4%,33.6%,63%,"15,890","157,030","294,431"
26,Nigeria,"415,080",17.8%,25.7%,54.6%,"73,884","106,676","226,634"
27,Iran,"412,340",11.2%,40.6%,48.2%,"46,182","167,410","198,748"
28,Thailand,"390,592",13.3%,34%,52.7%,"51,949","132,801","205,842"
29,Austria,"387,299",1.5%,29.5%,69%,"5,809","114,253","267,236"
30,Norway,"376,268",2.7%,38.3%,59%,"10,159","144,111","221,998"
31,United Arab Emirates,"416,444",0.7%,59.4%,39.8%,"2,915","247,368","165,745"
32,Colombia,"400,117",8.9%,38%,53.1%,"35,610","152,044","212,462"
33,Denmark,"347,196",4.5%,19.1%,76.4%,"15,624","66,314","265,258"
34,South Africa,"341,216",2.5%,31.6%,65.9%,"8,530","107,824","224,861"
35,Algeria,"246,397",3.3%,17.9%,78.9%,"8,131","44,105","194,407"
36,Venezuela,"209,226",4.7%,34.9%,60.4%,"9,834","73,020","126,373"
```

Full code is available [on github](https://github.com/kaijento/code/blob/master/webscraping/wikipedia.com/gdp-table-to-csv.py).
