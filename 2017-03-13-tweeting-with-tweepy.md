---
title: "tweeting with tweepy"
tags: 
- python
- twitter
- tweepy
---

This is a simple introduction which shows you how to register an application on
your twitter account and send tweets from `Python` using `tweepy`.

## Creating your application

Go to [http://apps.twitter.com](http://apps.twitter.com) and click 
on the `Create New App` button. Here you'll be prompted for the details of your application.

You can use `http://127.0.0.1` for `Website` and leave `Callback URL` empty.

![](/img/1489409040-create-twitter-app.png)

## Generating your access token

After your application has been created you will want to click the `Keys and Access Tokens` tab.
Scroll down to the bottom and click on the `Create my access token` button.

After this completes you should be able to see your

* Consumer Key (API Key)
* Consumer Secret (API Secret)
* Access Token
* Access Token Secret

## Storing your keys and tokens (credentials)

You'll want to keep your credentials separate from your source code so we'll store them in a 
JSON formatted file which can be loaded easily using the `json` module.

`.tweepy.json`

```javascript
{
    "consumer_key":        "XXX",
    "consumer_secret":     "XXX",
    "access_token":        "XXX",
    "access_token_secret": "XXX"
}
```

## Authentication

Assuming you have **tweepy** installed (e.g. by using `pip install tweepy --user`) -
the next step would be to create a file with the following code:

`tweet.py`

```python
# -*- coding: utf-8 -*-
import json, tweepy

config_file = '.tweepy.json'
with open(config_file) as fh:
    config = json.load(fh)

auth = tweepy.OAuthHandler(
    config['consumer_key'], config['consumer_secret']
)
auth.set_access_token(
    config['access_token'], config['access_token_secret']
)

twitter = tweepy.API(auth)
```

The code should be easy to follow - we just load up the credentials from the config file using `json.load()`.
We pass them to the `OAuthHandler` and finally call `tweepy.API()` to log in.

## Tweeting

If you run the above code you should get no output (assuming your credentials are correct) as we did not
do anything after logging in. Instead we'll use  `python -i tweet.py` to run the code and leave us in an
interactive session.

To post a tweet use `update_status()`

```python
>>> tweet = twitter.update_status('tweeting from the command-line ♡')
>>> tweet.id
841288722830237696
>>> tweet.user.screen_name
'kaijento'
```

![](/img/1489409040-tweet.png)

To delete a tweet use `destroy_status(status_id)` 

```python
>>> tweet = twitter.destroy_status(tweet.id)
```

`destroy_status()` returns the **Status** object - so we assign it to a variable here 
to prevent output from the implicit `print()` of the interactive session.

## That's it!

You now have an interactive session for sending tweets. To learn more about
`tweepy` you can read the [tweepy docs](http://docs.tweepy.org/).
