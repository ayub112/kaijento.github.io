---
title: "pandas: \"cleaning\" email addresses"
tags: 
- python
- pandas
- regex
---

<em>Hi all,  I have a pandas dataframe with a column for email addresses.  
Most of the addresses are fine, but some have some extraneous text after them.  
I've been looking around for a way to remove the text without impacting other 
correct email addresses. Here is an example:</em>

    joe.sample@gmail.com08/03/0601:20:03

So the problem is that some email addresses contain a date/timestamp that needs
to be removed from the end. The format is always the same.

```python
>>> df
                                  email
0                           foo@bar.com
1  joe.sample@gmail.com08/03/0601:20:03
2           omg@lol.bbq01/01/1701:01:01
3                           keep@me.com
4            pls@no.com12/12/1217:30:00

[5 rows x 1 columns]
```

We can use the `replace()` DataFrame method with the `regex=True` parameter.

```python
>>> df.email.replace(r'\d{2}/\d{2}/\d{4}:\d{2}:\d{2}$', '', regex=True)
0             foo@bar.com
1    joe.sample@gmail.com
2             omg@lol.bbq
3             keep@me.com
4              pls@no.com
Name: email, dtype: object
```

You can either pass `inplace=True` or you can assign the result back yourself
depending on what style you prefer.

```python
>>> df.email = df.email.replace(r'\d{2}/\d{2}/\d{4}:\d{2}:\d{2}$', '', regex=True)
>>> df
                  email
0           foo@bar.com
1  joe.sample@gmail.com
2           omg@lol.bbq
3           keep@me.com
4            pls@no.com

[5 rows x 1 columns]
```

So what does `\d{2}/\d{2}/\d{4}:\d{2}:\d{2}$` mean exactly?

It may look complex but it's actually quite simple. All you need to know is:

* `\d`  matches a <em>"digit"</em> i.e. `[0-9]`
* `{}`  is for repetition, `{2}` means exactly 2 times.
* `$`   matches the end of the string

So we have:

* `\d{2}/` matches <em>"2 digits followed by /"</em>
* `\d{2}/` matches <em>"2 digits followed by /"</em>
* `\d{4}:` matches <em>"4 digits followed by :"</em>
* `\d{2}:` matches <em>"2 digits followed by :"</em>
* `\d{2}$` matches <em>"2 digits at the end of the string"</em>

So the `$` <em>"anchors"</em> the whole match to the end of the string. 

The opposite of `$` is `^` which matches the start of the string.

Also, as well as `{x}` to match <em>"exactly <strong>x</strong> times"</em> we have

* `{x,y}` to match <em>"between <strong>x</strong> and <strong>y</strong> times inclusive"</em> 
* `{x,}`  to match <em>"greater than or equal to <strong>x</strong> times."</em>

Some people prefer to use `\d\d` instead of `\d{2}` which both do the same thing. 
So you could also see the regex written as `\d\d/\d\d/\d{4}:\d\d:\d\d$` 
