---
title: "modifying the index/counter in a python for loop"
tags: 
- python
- javascript
---

When [porting tripadvisor's](
{% post_url 2017-03-17-web-scraping-tripadvisor.com %})
`asdf()` javascript function to Python we came across 
an interesting difference with regards to how `for` loops work in 
javascript and Python.

Inside the function they modify the index they are using in a for loop e.g.

```javascript
js> for (i = 0; i < 5; i++) {
        if (i == 2) {
            i = 4;
        }
        print(i);
    }
0
1
4
```

So in this example we get 3 iterations. Let's try write a similar loop in Python:

```python
>>> for i in range(5):
...     if i == 2:
...         i = 4
...     print(i)
...
0
1
4
3
4
```

We get 5 iterations.

`i` was modified as you can see however it was given a new value at the
start of the next iteration of the loop.

It may help to think of Python's `for` loop as a `foreach` e.g.

```python
for each item in ...:
    do something with item 
```

So it does not matter if you modify `item` inside the loop as it will be given its
new value at the start of the next iteration.

Now it is possible to <em>"break"</em> out and jump to the next iteration of a for loop 
using `continue`:

```python
>>> for i in range(5):
...     if i == 2:
...         continue
...     print(i)
...
0
1
3
4
```

However, using `continue` wasn't applicable here so a `while` loop was chosen:

```python
>>> i = 0
>>> while i < 5:
...     if i == 2:
...         i = 4
...     print(i)
...     i += 1
0
1
4
```

Not the prettiest of solutions or very <em>"pythonic"</em> but it was simpler than trying 
to decipher the original <em>"obfuscated"</em> javascript code.
