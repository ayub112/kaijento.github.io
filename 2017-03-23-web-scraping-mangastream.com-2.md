---
title: "Web scraping: mangastream.com #2"
date:  2017-03-23 16:57:42 +0100
tags:
  - webscraping
  - python
  - bash
  - curl
  - grep
  - regex
---

In [Part 1]({% post_url 2017-03-23-web-scraping-mangastream.com %}) 
we saw how to scrape all of the comic images from a particular "chapter" 
on a mangastream.com comic such as:

[http://mangastream.com/r/demons_plan/010/3997/](http://mangastream.com/r/demons_plan/010/3997/)

The final version of the code can be seen [here](https://github.com/kaijento/code/blob/master/webscraping/mangastream.com/mangastream.com.py).

Now we will convert it to a bash function that uses `curl` as the HTTP client
and GNU `grep` as the "parser".

We will start with the output:

```bash
$ mangastream http://mangastream.com/r/demons_plan/010/3997/
mkdir: created directory ‘demons_plan’
mkdir: created directory ‘demons_plan/010’
GET:    http://mangastream.com/r/demons_plan/010/3997
######################################################################## 100.0%
CREATE: demons_plan/010/01.png
######################################################################## 100.0%
GET:    http://mangastream.com/r/demons_plan/010/3997/2
######################################################################## 100.0%
CREATE: demons_plan/010/01a.jpg
######################################################################## 100.0%
[...]
GET:    http://mangastream.com/r/demons_plan/010/3997/20
######################################################################## 100.0%
CREATE: demons_plan/010/19.png
######################################################################## 100.0%
```

Here is how we did it:

```bash
mangastream () {
    url=${1%/}

    IFS=/ read -a parts <<< $url

    dirpath=${parts[-3]}/${parts[-2]}
    mkdir -p -v "$dirpath"

    echo "GET:    $url"
    html=$(curl -# -A Mozilla/5.0 "$url")
    last=$(grep -P -o 'Last Page \(\K\d+' <<< $html)

    for ((n=1; n<=last; n++))
    do
        echo "GET:    $url/$n"
        html=$(curl -# -A Mozilla/5.0 "$url/$n")
         img=$(grep -P -o '<img id="manga-page" src="\K[^"]+' <<< $html)

        filename=${img##*/}

        echo "CREATE: $dirpath/$filename"
        curl -# "$img" -o "$dirpath/$filename"
    done
}
```

Again we're fetching the first page twice as it allows us to avoid duplicating code and
an extra request isn't going to hurt.

The `-#` option of curl makes it print out the nice progress bar you see and `-A` sets
the `user-agent` header.

## I got (?:problems){99} but (?!regex)

If you're read the internet at all you've probably read <em>"never use regular expressions lol
now you have problems!!11"</em>. It seems to be some sort of meme. You will probably have also
read never use regular expressions on HTML. 

It depends on what you're doing really. If you're trying to find the third `<a>` tag in the 
last `<div>` of a `<span>` that has `id="content"` then yes, using a parser that supports
CSS Selectors and/or XPath would be the way to go.

If you're just doing straight text extraction then the fact the data is HTML is pretty much
irrelevant in most cases.

## grep -P -o 

The standard `grep` tool was built to find lines that match patterns and to print the full line.
GNU `grep` got `-o` which will print "what was matched" instead of just printing the whole line.
Using capture groups with `-o` make no difference it will always print what was matched:

```bash
$ echo foobarbaz | grep -E -o 'foo(...)'
foobar
```

It wouldn't be unreasonable to expect that to only return `bar` i.e. what was captured by the group
but alas that is not the case.  

GNU `grep` was then given a `-P`

```bash
$ grep --help | grep -- -P
  -P, --perl-regexp         PATTERN is a Perl regular expression
```

That allows you to use lookarounds:

```bash
$ echo foobarbaz | grep -P -o 'foo\K...'
bar
$ echo foobarbaz | grep -P -o '(?<=foo)...'
bar
```

`\K` is like a like a lookbehind it will exclude everything that comes before it from "what
was matched". 

To exclude what comes after you can use a positive lookahead assertion `(?=)`

```
$ echo foobarbaz1barbaz2 | grep -P -o '^.+?bar'
foobar
$ echo foobarbaz1barbaz2 | grep -P -o '^.+?(?=bar)'
foo
```

Essentially `grep -P -o` is like using `perl -nle 'print for /pattern/g'`

So `\K` is what allows us to extract the last page number and the `src` attribute value:

`'Last Page \(\K\d+'` shouldn't need much explanation. 

* `Last Page\(` matches `Last Page(` 
* `\K` means exclude everything previous from "what was matched"
* `\d+` matches one or more digits i.e. the `20` in `Last Page(20)`

Our second pattern `'<img id="manga-page" src="\K[^"]+'`:

* `<img id="manga-page" src="` matches literally
* `\K` means exclude everything previous from "what was matched"

`[^"]+` means not a `"` character 1 or more times. It matches up to but not including the next
`"` character (if it exists)

```bash
$ echo '<img id="manga-page" src="omglol">' | grep -P -o '<img id="manga-page" src="\K[^"]+'
omglol
```

## That's it!

Well that's all for now, perhaps next time we can look at extracting each chapter URL from the
[Full List](http://mangastream.com/manga/demons_plan) page.
