---
title: "Poloniex API: Invalid command."
date:  2017-05-05 19:30:15 +0100
tags:
 - python
 - requests
 - urllib
 - netcat
---

*We're trying to make a request to the `returnBalances` 
endpoint of the [Poloniex API](https://poloniex.com/support/api/)
using Python's `requests` library but we're getting back
an "Invalid command." error. How do we fix it?*

## requests 

Here is the `requests` version of the code.

```python
import hashlib, hmac, time, requests

api    = 'https://poloniex.com/tradingApi'
key    = 'mykey'
secret = 'mysecret'

nonce = int(time.time() * 1000)
data  = 'command=returnBalances&nonce={}'.format(nonce)

signature = hmac.new(secret.encode(), data.encode(), digestmod=hashlib.sha512)

headers = {
    'Key' : key,
    'Sign': signature.hexdigest(),
}

r = requests.post(api, data=data, headers=headers)

try:
    print(r.json()['BTC'])
except:
    print(r.text)
```

Running the code using Python `3.6` we get

```bash
$ python3 poloniex.py
{"error":"Invalid command."}
```

Hmmm... everything looks okay in the code. My first thought is to
see if we can replicate the behaviour when using `urllib.request`.

<small>(If we were using Python `2.7` it would be `urllib2`)</small>

## urllib.request

Here's the code rewritten using `urllib.request`

```python
import urllib.parse,  urllib.request
import json, hashlib, hmac, time, requests

api    = 'https://poloniex.com/tradingApi'
key    = 'mykey'
secret = 'mysecret'

data = {
    'command': 'returnBalances',
    'nonce'  : int(time.time() * 1000)
}
data = urllib.parse.urlencode(data).encode()

signature = hmac.new(secret.encode(), data, hashlib.sha512)
headers = {
    'Key' : key,
    'Sign': signature.hexdigest()
}

request = urllib.request.Request(url=api, data=data, headers=headers, method='POST')
text    = urllib.request.urlopen(request).read().decode()

try:
    print(json.loads(text)['BTC'])
except:
    print(text)
```

And let's see the output.

```bash
$ python3 poloniex-urllib.py
0.00000000
```

It works as expected. <small>(but I gotz no bitcoinz ;_;)</small>

So it seems like a good idea would be to debug both requests to 
see how they differ.

## netcat

One way to do that is to use the `nc` (or `netcat`) command to listen
on a local port and point our requests at it. We can use the command
`nc -l 8080` to listen on port `8080`

This means we will send our `POST` request to `http://localhost:8080`
instead of `https://poloniex.com/tradingApi`. We will also set the `nonce` value to `1` in both versions 
of the code as to have the exact same `POST` data.

Here's the `netcat` output for the `requests` version.

```bash
$ nc -l 8080
POST / HTTP/1.1
Accept-Encoding: gzip, deflate
Accept: */*
Host: localhost:8080
Connection: keep-alive
User-Agent: python-requests/2.13.0
Key:  mykey
Sign: mysignature

Content-Length: 30

command=returnBalances&nonce=1
```

And here is the output for the `urllib.request` version.

```bash
$ nc -l 8080
POST / HTTP/1.1
Accept-Encoding: identity
Content-Type: application/x-www-form-urlencoded
Connection: close
Host: localhost:8080
User-Agent: Python-urllib/3.6
Key:  mykey
Sign: mysignature

Content-Length: 30

command=returnBalances&nonce=1
```

As we can see the `Content-Length` is identical as well as the `POST` data.
This suggests to us that the problem lies in the headers.

The first thing I notice is the `Content-Type` header in the `urllib.request`
version that does not exist in the `requests` version.

So let's go and manually add that header to our `headers` dict in our
`requests` code.

```python
headers = {
    'Content-Type': 'application/x-www-form-urlencoded',
    'Key'         : key,
    'Sign'        : signature.hexdigest(),
}
```

We also point our `POST` request back at the Poloniex API and run
the code.

```bash
$ python3 poloniex.py
0.00000000
```

Okay great! So it nows works with `requests` and it looks like 
the `Content-Type` header was the culprit. 

So why is it not being set by `requests`?

## The data argument

Usually when passing form data you supply a dict to the `data` argument.

```python
>>> requests.post('http://localhost:8080', data={'foo': 'bar'})
```

Let's look at the `netcat` output.

```bash
$ nc -l 8080
POST / HTTP/1.1
Host: localhost:8080
User-Agent: python-requests/2.13.0
Accept-Encoding: gzip, deflate
Accept: */*
Connection: keep-alive
Content-Length: 7
Content-Type: application/x-www-form-urlencoded

foo=bar
```

So when we pass a dict it sets the header. However if we pass
a string...

```python
>>> requests.post('http://localhost:8080', data='foo=bar')
```

... the header does not get set.

```bash
$ nc -l 8080
POST / HTTP/1.1
Host: localhost:8080
User-Agent: python-requests/2.13.0
Accept-Encoding: gzip, deflate
Accept: */*
Connection: keep-alive
Content-Length: 7

foo=bar
```

From [the docs:](http://docs.python-requests.org/en/latest/user/quickstart/#more-complicated-post-requests)

> There are many times that you want to send data that is not form-encoded. 
> If you pass in a string instead of a dict, that data will be posted directly.

So not only does it skip the *"urlencoding"* of the data it also skips the
sending of the `Content-Type` header which is causing the Poloniex API 
to *"reject"* our request.

We could change `data` to a dict.

```python
data = 'command=returnBalances&nonce={}'.format(nonce)
```

But as we need to turn it into a *"string"* for use in generating the `signature`
it seems simpler to just leave `data` as it is and manually send the
`Content-Type` header with the request.
