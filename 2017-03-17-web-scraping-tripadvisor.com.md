---
title: "Web scraping: tripadvisor.com"
tags: 
- python
- regex
- requests
- javascript
- webscraping
---

It seems everybody wants to scrape tripadvisor and a common issue people 
run into is how to get the `Website URL` of the Restaurant, Hotel, whatever.

## Inspecting the DOM

If you inspect the HTML or DOM of a tripadvisor listing page and view the `Website`
section you will see some sort of obfuscation going on with the URL. 

![](/img/1489761575-tripadvisor-inspector.png)

It looks like a string of nonsense with several instances of `QQQQoqn` repeating. 
The first clue here is the `isAsdf` parameter.

```html
<div class="fl">
  <span class="taLnk" onclick="
    ta.trackEventOnPage('Eatery_Listing', 'Website', 9750183, 1); 
    ta.util.cookie.setPIDCookie(15190); 
    ta.call('ta.util.link.targetBlank', event, this, {'aHref':
      'LqMWJQzZYUWJQpEcYGII26XombQQoqnQQQQoqnqgoqnQQQQoqnQQQQoqnQQQQoqnqgoqnQQQQoqn...', 
      'isAsdf':true})">Site Web
  </span>
</div>
```

## "Scraping" the email address from tripadvisor

Incidentally, the email address is stored in plaintext inside the `onclick` attribute of
another `<div>` tag.

```html
<div class="taLnk fl" onclick="
  ta.trackEventOnPage('Eatery_Listing', 'Email', 9750183, 1);
  return ta.call(
    'ta.locationDetail.checkEmailAction',event,this,'foo@bar.com'
```

The simplest way to <em>"scrape"</em> this is probably using the `re` module

```python
>>> re.findall(r'ta\.locationDetail\.checkEmailAction.+\'(.+)\'', html)
['foo@bar.com']  
```

## asdf() in javascript

So back to the URL - because it's inside an `onclick` attribute we know we need to inspect
the javascript code so we view the <strong>Network</strong> tab of <strong>Dev Tools</strong> as 
we refresh the page and set the filter to <strong>JS</strong> requests. 

![](/img/1489761601-tripadvisor-networktab.png)

We see a request for `tripadvisor-c-v21056403377b.js` which seems like a good place to start.

Another perhaps simpler approach would be to open the <strong>Debugger</strong> tab
and perform a <em>Search all scripts</em>.

So if we search for `asdf` inside `tripadvisor-c-v21056403377b.js` we will find the 
following:

```javascript
a.asdf = function (f) {
    var j = {
      '': [
        '&', '=', 'p', '6', '?', 'H', '%', 'B', '.com', 'k', '9', '.html',
        'n', 'M', 'r', 'www.', 'h', 'b', 't', 'a', '0', '/', 'd', 'O', 'j',
        'http://', '_', 'L', 'i', 'f', '1', 'e', '-', '2', '.', 'N', 'm',
        'A', 'l', '4', 'R', 'C', 'y', 'S', 'o', '+', '7', 'I', '3', 'c',
        '5', 'u', 0, 'T', 'v', 's', 'w', '8', 'P', 0, 'g', 0
      ],
      q: [
        0, '__3F__', 0, 'Photos', 0, 'https://', '.edu', '*', 'Y', '>',
        0, 0, 0, 0, 0, 0, '`', '__2D__', 'X', '<', 'slot', 0, 'ShowUrl',
        'Owners', 0, '[', 'q', 0, 'MemberProfile', 0, 'ShowUserReviews',
        '"', 'Hotel', 0, 0, 'Expedia', 'Vacation', 'Discount', 0, 
        'UserReview', 'Thumbnail', 0, '__2F__', 'Inspiration', 'V', 'Map', 
        ':', '@', 0, 'F', 'help', 0, 0, 'Rental', 0, 'Picture', 0, 0, 0, 
        'hotels', 0, 'ftp://'
      ],
      x: [
        0, 0, 'J', 0, 0, 'Z', 0, 0, 0, ';', 0, 'Text', 0, '(', 'x',
        'GenericAds', 'U', 0, 'careers', 0, 0, 0, 'D', 0, 'members', 
        'Search', 0, 0, 0, 'Post', 0, 0, 0, 'Q', 0, '$', 0, 'K', 0, 
        'W', 0, 'Reviews', 0, ',', '__2E__', 0, 0, 0, 0, 0, 0, 0, 
        '{', '}', 0, 'Cheap', ')', 0, 0, 0, '#', '.org'
      ],
      z: [
        0, 'Hotels', 0, 0, 'Icon', 0, 0, 0, 0, '.net', 0, 0, 'z', 0, 0,
        'pages', 0, 'geo', 0, 0, 0, 'cnt', '~', 0, 0, ']', '|', 0,
        'tripadvisor', 'Images', 'BookingBuddy', 0, 'Commerce', 0, 0,
        'partnerKey', 0, 'area', 0, 'Deals', 'from', '\\', 0, 'urlKey', 0,
        '\'', 0, 'WeatherUnderground', 0, 'MemberSign', 'Maps', 0, 'matchID',
        'Packages', 'E', 'Amenities', 'Travel', '.htm', 0, '!', '^', 'G'
      ]
    };

    var e = '';
    for (var d = 0; d < f.length; d++) {
        var k = f.charAt(d);
        var g = k;
        if (j[k] && d + 1 < f.length) {
            d++;
            g += f.charAt(d)
        } else {
            k = ''
        }
        var h = getOffset(f.charCodeAt(d));
        if (h < 0 || typeof j[k][h] == 'String') {
            e += g
        } else {
            e += j[k][h]
        }
    }
    return e
};
```

It refers to another function `getOffset` which we can also find in the file

```javascript
a.getOffset = function (d) {
    if (d >= 97 && d <= 122) {
      return d - 61
    }
    if (d >= 65 && d <= 90) {
      return d - 55
    }
    if (d >= 48 && d <= 71) {
      return d - 48
    }
    return - 1
};
```

## Writing asdf() in Python

It's not entirely clear what is going on in these functions or what kind of 
<em>"obfuscation"</em> tecnhique they are using so we will just port them straight
to Python.

```python
def asdf(d):
    def get_offset(n):
        if 97 <= n <= 122:
            return n - 61 
        if 65 <= n <= 90:
            return n - 55
        if 48 <= n <= 71:
            return n - 48
        return -1

    h = {
        '': [
            '&', '=', 'p', '6', '?', 'H', '%', 'B', '.com', 'k', '9', 
            '.html', 'n', 'M', 'r', 'www.', 'h', 'b', 't', 'a', '0', '/', 
            'd', 'O', 'j', 'http://', '_', 'L', 'i', 'f', '1', 'e', '-', 
            '2', '.', 'N', 'm', 'A', 'l', '4', 'R', 'C', 'y', 'S', 'o', 
            '+', '7', 'I', '3', 'c', '5', 'u', 0, 'T', 'v', 's', 'w', '8', 
            'P', 0, 'g', 0
        ],
        'q': [
            0, '__3F__', 0, 'Photos', 0, 'https://', '.edu', '*', 'Y', '>',
            0, 0, 0, 0, 0, 0, '`', '__2D__', 'X', '<', 'slot', 0, 'ShowUrl', 
            'Owners', 0, '[', 'q', 0, 'MemberProfile', 0, 'ShowUserReviews', 
            '"', 'Hotel', 0, 0, 'Expedia', 'Vacation', 'Discount', 0, 
            'UserReview', 'Thumbnail', 0, '__2F__', 'Inspiration', 'V', 
            'Map', ':', '@', 0, 'F', 'help', 0, 0, 'Rental', 0, 'Picture', 
            0, 0, 0, 'hotels', 0, 'ftp://'
        ],
        'x': [
            0, 0, 'J', 0, 0, 'Z', 0, 0, 0, ';', 0, 'Text', 0, '(', 'x', 
            'GenericAds', 'U', 0, 'careers', 0, 0, 0, 'D', 0, 'members', 
            'Search', 0, 0, 0, 'Post', 0, 0, 0, 'Q', 0, '$', 0, 'K', 0, 'W',
            0, 'Reviews', 0, ',', '__2E__', 0, 0, 0, 0, 0, 0, 0, '{', '}', 0,
            'Cheap', ')', 0, 0, 0, '#', '.org'
        ],
        'z': [
            0, 'Hotels', 0, 0, 'Icon', 0, 0, 0, 0, '.net', 0, 0, 'z', 0,
            0, 'pages', 0, 'geo', 0, 0, 0, 'cnt', '~', 0, 0, ']', '|', 0, 
            'tripadvisor', 'Images', 'BookingBuddy', 0, 'Commerce', 0, 0, 
            'partnerKey', 0, 'area', 0, 'Deals', 'from', '\\', 0, 'urlKey',
            0, '\'', 0, 'WeatherUnderground', 0, 'MemberSign', 'Maps', 0, 
            'matchID', 'Packages', 'E', 'Amenities', 'Travel', '.htm', 0, 
            '!', '^', 'G'
        ]
    }

    b = ''
    i = 0
    while i < len(d):
        j = d[i]
        f = j
        if h.get(j) and i + 1 < len(d):
            i += 1
            f += d[i]
        else:
            j = ''
        g = get_offset(ord(d[i]))
        if g < 0 or type(h[j][g]) == 'str':
            b += f
        else:
            b += h[j][g]
        i += 1

    return b
```

## partner_key

You may notice that we have a `while` loop instead of the original `for` loop. If you'd like
to know why that is I've 
[written a post about it]({% post_url 2017-03-15-modifying-index-in-a-python-for-loop %}).

Passing our original string to the above function gives us back a cryptic string that we will 
call `partner_key`

    /ShowUrl-a_partnerKey.1-a_url.http%253A__5F____5F__2F__5F____5F____5F____5F__2F__5F____5F__www__5F____5F__2E__5F____5F__

If we go back and actually click on the original link we can see it makes a request to 
`http://tripadvisor.com/ShowUrl-a_partnerKey.-aurl....` which then redirects 
to the final URL we want.

We could replicate that by using `requests` and accessing the `.url` attribute of the 
response.

```python
>>> import requests
>>> partner_key = asdf(obfuscated_url)
>>> requests.get('http://tripadvisor.com' + partner_key).url
'http://www.my-website.com/'
```

However looking at the `partner_key` we can see `3A`, `2E` and `2F` which are `:`, `/` and `.` in 
hexadecimal.

```python
>>> chr(int('2E', 16))
'.'
```

We can use `int(hexcode, 16)` to convert from hex to int and then use `chr(integer)` to convert
from integer to ascii character.

This may look familiar to you if you have seen 
[percent encoding](https://en.wikipedia.org/wiki/Percent-encoding) before.

```python
>>> import urllib
>>> urllib.unquote('%2F%2F%2E')
'//.'
```

So it looks like if we strip the excess `__5F__5F` type strings we can <em>"decode"</em>
the leftover hexcodes to get the URL.

Once again we <strong>re</strong>ach for <strong>re</strong>gex with the `re` module.

```python
>>> import re
>>> url = partner_key.replace('__', '').replace('%253A', ':')
>>> url = re.sub(r'.*?(http.*?)-a_url.*', r'\1', url)
>>> url = re.sub(r'5F5F([A-Z\d]{2})5F5F', lambda match: chr(int(match.group(1), 16)), url)
>>> url
'http://www.my-website.com/'
```

## That's it!

That pretty much sums up this post. 
[Here](https://github.com/kaijento/code/blob/master/webscraping/tripadvisor.com/tripadvisor.py)
is a full Python example that takes a tripadvisor URL and extracts the obfuscated URL, decodes 
it and also extracts the email address for educational purposes only.
