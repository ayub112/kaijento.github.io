---
title: "Web scraping: serebii.net"
date:  2017-05-01 09:01:53 +0100
tags:
 - python
 - webscraping
 - beautifulsoup
 - requests
---

We're trying extract information about a Pokémon attack from [http://serebii.net](http://serebii.net) 
which *"lists all the moves currently available through the Pokémon in Sun & Moon"*
using Python.

The example URL given to work with is [http://serebii.net/attackdex-sm/heavyslam.shtml](http://serebii.net/attackdex-sm/heavyslam.shtml)
and we're trying to extract the `Battle Type` information from the page.

![](/img/1493625937-attackdex-sm.png)

## HTML

Let's take a look at the HTML.

```html
<table  class="dextable">
  <tr>
    <td class="fooevo" width="33%">
    <b>Attack Name</b>      </td>
    <td class="fooevo" width="33%">
    <b>Battle Type</b>      </td>
    <td class="fooevo" width="33%">
    <b>Category</b> </td>
  </tr>
  <tr>
    <td class="cen">
    Heavy Slam<br /> &#12504;&#12499;&#12540;&#12508;&#12531;&#12496;&#12540;
      <!--<table width="300" cellspacing="0" cellpadding="0">
            <tr> <td><b>English</b>: </td><td>Heavy Slam</td></tr>
            <tr> <td><b>Japanese</b>: </td><td>&#12504;&#12499;&#12540;&#12508;&#12531;&#12496;&#12540;</td></tr>
            <tr> <td><b>French</b>: </td><td>Tacle Lourd</td></tr>
            <tr> <td><b>German</b>: </td><td>Rammboss</td></tr>
            <tr> <td><b>Spanish</b>: </td><td>Cuerpo Pesado</td></tr>
            <tr> <td><b>Italian</b>: </td><td>Pesobomba</td></tr>
            <tr> <td><b>Korean</b>: </td><td>&#54756;&#48708;&#48388;&#48260;</td></tr>
          </table>-->
    </td>
    <td class="cen">
      <a href="/attackdex-sm/steel.shtml"><img src="/pokedex-bw/type/steel.gif" border="0"></a>
    </td>
    <td class="cen">
      <a href="/attackdex-sm/physical.shtml"><img src="/pokedex-bw/type/physical.png" border="0"></a>
    </td>
  </tr>
```

What we're trying to locate here is the second `<a>` tag i.e.

```html
<a href="/attackdex-sm/steel.shtml"><img src="/pokedex-bw/type/steel.gif" border="0"></a>
```

There doesn't seem to be any unique markers we can use to isolate this particular
tag as all of the `class` attributes are the same. In situations like this we 
usually have to resort to *"finding by position"* or *"navigating the tree"*.

We know that what we want is contained in the second `<td>` tag which itself is contained
in the `<tr>` tag after the `<tr>` tag that contains `<b>Battle Type</b>`

This means we can locate `<b>Battle Type</b>` move up to the parent `<tr>` move to 
the next `<tr>` and get the second `<td>` tag.

## Code

We'll be using `requests` to fetch the HTML and `BeautifulSoup` with `html5lib` to
parse it.

```python
>>> import requests
>>> from   bs4 import BeautifulSoup
>>> 
>>> url  = 'http://serebii.net/attackdex-sm/heavyslam.shtml'
>>> r    = requests.get(url, headers={'user-agent': 'Mozilla/5.0'})
>>> soup = BeautifulSoup(r.content, 'html5lib')
```

The default `html.parser` has certain issues which is why I prefer
to use `html5lib` instead. It needs to be installed separately e.g. using 
`pip install html5lib --user` 

So we need to find `<b>Battle Type</b>` for which we can use the 
[string argument](https://www.crummy.com/software/BeautifulSoup/bs4/doc/#the-string-argument)
of `find()` 

```python
>>> soup.find('b', string='Battle Type')
<b>Battle Type</b>
```

So `b` is the tag name to search for and `string='Battle Type'` means its 
*"string content"* must equal exactly `Battle Type`

`string` can also be given a regex for *non-exact* or regex matching.

So from the `<b>` tag we want to find the parent `<tr>` tag which we can do 
with `find_parent('tr')` which searches *"upwards"*.

Our call to `find()` is returning a `bs4.element.Tag` object.

```python
>>> type(soup.find('b', string='Battle Type'))
>>> <class 'bs4.element.Tag'>
```

Which allows us to simply chain the call to `find_parent()`

```python
>>> soup.find('b', string='Battle Type').find_parent('tr')
<tr>\n\t\t<td class="fooevo" width="33%">\n\t\t<b>Attack Name</b>...
```

From there we need to find the next `<tr>` tag which we can do with `find_next('tr')`

```python
>>> soup.find('b', string='Battle Type').find_parent('tr').find_next('tr')
<tr>\n\t\t<td class="cen">\n\t\tHeavy Slam<br/>...
```

The chain of method calls is beginning to get quite large so we'll create a 
variable to store this tag.

```python
>>> tr = soup.find('b', string='Battle Type').find_parent('tr').find_next('tr')
```

I've chosen `tr` here as the name as I like to name the variable the same
as the tag it represents but this does not need to be the case.

So now that we are positioned at the `<tr>` tag that hold the information
we have a couple of choices. We could simply find the first `<a>` tag as
we know the first `<td>` tag doesn't contain one.

```python
>>> tr.find('a')
<a href="/attackdex-sm/steel.shtml"><img border="0" src="/pokedex-bw/type/steel.gif"/></a>
>>> tr.a
<a href="/attackdex-sm/steel.shtml"><img border="0" src="/pokedex-bw/type/steel.gif"/></a>
```

That may change however so the more *"proper"* approach would be to
isolate the second `<td>` specifically and work from there as we had 
originally described.

`find_all('td')` will return a list of `<td>` tags.

```python
>>> len(tr.find_all('td'))
3
```

Which we can then index using `[1]` to get the second tag.

```python
>>> tr.find_all('td')[1]
<td class="cen">\n\t\t<a href="/attackdex-sm/steel.shtml"><img border="0" src="/pokedex-bw/type/steel.gif"/></a></td>
```

`BeautifulSoup` has some *"shortcut"* syntax for `find()` and `find_all()`. 
As you may have noticed above `find('a')` and `.a` produced the same result.
With `find_all()` we can actually omit the method name as it is the default method
to be called.

```python
>>> tr('td')[1]
<td class="cen">\n\t\t<a href="/attackdex-sm/steel.shtml"><img border="0" src="/pokedex-bw/type/steel.gif"/></a></td>
```

So `find_all('td')` and `('td')` here are equivalent.

From there we can navigate to the `<img>` tag and extract the `src` attribute.

```python
>>> tr('td')[1].img
<img border="0" src="/pokedex-bw/type/steel.gif"/>
>>> tr('td')[1].img['src']
'/pokedex-bw/type/steel.gif'
>>> tr('td')[1].img['src'].split('/')
['', 'pokedex-bw', 'type', 'steel.gif']
>>> tr('td')[1].img['src'].split('/')[-1]
'steel.gif'
>>> tr('td')[1].img['src'].split('/')[-1].split('.')
['steel', 'gif']
>>> tr('td')[1].img['src'].split('/')[-1].split('.')[0]
'steel'
```

Again you may want to *split* it up and use a variable as we 
did with `tr` instead of having a large chain of method calls.

## Extracting a comment using BeautifulSoup

So after we extracted the `Battle Type` it was then requested
that we extract the `Attack Name`.

The `Attack Name` is contained in the first `<td>` inside our `<tr>`
tag however it the data that we want is inside a  `<table>` 
tag that is commented out.

This `<table>` contains translations of the name in different languages
and we would like to build a dict using this data.

```html
<tr>
<td class="cen">
Heavy Slam<br /> &#12504;&#12499;&#12540;&#12508;&#12531;&#12496;&#12540;
  <!--<table width="300" cellspacing="0" cellpadding="0">
        <tr> <td><b>English</b>: </td><td>Heavy Slam</td></tr>
        <tr> <td><b>Japanese</b>: </td><td>&#12504;&#12499;&#12540;&#12508;&#12531;&#12496;&#12540;</td></tr>
        <tr> <td><b>French</b>: </td><td>Tacle Lourd</td></tr>
        <tr> <td><b>German</b>: </td><td>Rammboss</td></tr>
        <tr> <td><b>Spanish</b>: </td><td>Cuerpo Pesado</td></tr>
        <tr> <td><b>Italian</b>: </td><td>Pesobomba</td></tr>
        <tr> <td><b>Korean</b>: </td><td>&#54756;&#48708;&#48388;&#48260;</td></tr>
      </table>-->
</td>
```

There's currently no nice way to search for comments. The simplest
method seems to be by passing a callback to the `string` argument
which uses `isinstance()` to check the object type.

```python
>>> from bs4.element import Comment
>>> tr.find(string=lambda tag: isinstance(tag, Comment))
'<table width="300" cellspacing="0" cellpadding="0">\n\t\t\t<tr>...
```

We could also navigate to the `<br />` tag then use multiple `.next` calls
to navigate to the comment.

```python
>>> tr.br.next
' \u30d8\u30d3\u30fc\u30dc\u30f3\u30d0\u30fc\n\t\t\t\t'
>>> tr.br.next.next
'<table width="300" cellspacing="0" cellpadding="0">\n\t\t\t<tr>...
```

As we now have the HTML as a *"string"* we need to feed it to a new
`BeautifulSoup` object in order to have it parsed.

```python
>>> comment = tr.br.next.next
>>> table   = BeautifulSoup(comment, 'html5lib')
```

The language names are in the `<b>` tags and the translations are 
then in the following `<td>` tags.

```python
>>> table.b
<b>English</b>
>>> table.b.find_next('td')
<td>Heavy Slam</td>
```

We can then use a *dict comprehension* to build our dict.

```python
>>> import json
>>>
>>> name = { b.text: b.find_next('td').text for b in table('b') }
>>> print(json.dumps(name, indent=4, sort_keys=True))
{
    "English": "Heavy Slam", 
    "French": "Tacle Lourd", 
    "German": "Rammboss", 
    "Italian": "Pesobomba", 
    "Japanese": "\u30d8\u30d3\u30fc\u30dc\u30f3\u30d0\u30fc", 
    "Korean": "\ud5e4\ube44\ubd04\ubc84", 
    "Spanish": "Cuerpo Pesado"
}
```

You could also build the dict using a regular `for` loop if you wish.

```python
name = {}
for b in table('b'):
    name[b.text] = b.find_next('td')
```

We've made use of the `json` module in the previous example to 
*pretty-print* the dict for display purposes.
