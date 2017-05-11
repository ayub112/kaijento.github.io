---
title: "pandas: create new column from sum of others"
date:  2017-04-22 18:34:24 +0100
tags:
 - python
 - pandas
---

<em>I have a pandas DataFrame with 2 columns `x` and `y`.
How do I create a new column `z` which is the sum of the
values from the other columns?</em>

Before

```python
>>> df
   x  y
0  1  4
1  2  5
2  3  6

[3 rows x 2 columns]
```

After

```python
>>> df
   x  y  z
0  1  4  5
1  2  5  7
2  3  6  9

[3 rows x 3 columns]
```

Let's create our DataFrame

```python
>>> import pandas
>>> df = pandas.DataFrame({'x': [1, 2, 3], 'y': [4, 5, 6]})
>>> df
   x  y
0  1  4
1  2  5
2  3  6

[3 rows x 2 columns]
```

## DataFrame.iterrows()

To iterate over rows of a dataframe we can use 
[DataFrame.iterrows](http://pandas.pydata.org/pandas-docs/stable/generated/pandas.DataFrame.iterrows.html)
which gives us back tuples of `index` and `row` similar to how
Python's `enumerate()` works.

```python
>>> z = []
>>> for index, row in df.iterrows():
...     z.append(row.x + row.y)
... 
>>> z
[5, 7, 9]
```

We could also write it using a *list comprehension*

```python
>>> z = [ row.x + row.y for index, row in df.iterrows() ]
```

We could then assign this list to our new column.

```python
>>> df['z'] = z
>>> df
   x  y  z
0  1  4  5
1  2  5  7
2  3  6  9

[3 rows x 3 columns]
```

Using `iterrows()` though is usually a *"last resort"*. If you're
using it more often than not there is a better way.

## DataFrame.apply()

We can use [DataFrame.apply](http://pandas.pydata.org/pandas-docs/stable/generated/pandas.DataFrame.apply.html)
to apply a function to all *columns* `axis=0` <small>(the default)</small> or `axis=1` *rows*. 

```python
>>> df = pandas.DataFrame({'x': [1, 2, 3], 'y': [4, 5, 6]})
>>> df.apply(lambda row: row.x + row.y, axis=1)
0    5
1    7
2    9 
dtype: int64
```

We've used `lambda` here which is like creating an *"inlined function"*
without having to give it a name i.e. with `def my_function()`

This doesn't modify the dataframe so we would have to assign the
result into our new column.

```python
>>> df['z'] = df.apply(lambda row: row.x + row.y, axis=1)
>>> df
   x  y  z
0  1  4  5
1  2  5  7
2  3  6  9

[3 rows x 3 columns]
```

While not in the *"last resort"* category there are still some cases 
where using `apply()` can be replaced with a *"better"* approach.

## Vectorized operations

So when we get all the values of a particular column
we are getting back a `Series` object.

```python
>>> df = pandas.DataFrame({'x': [1, 2, 3], 'y': [4, 5, 6]})
>>> df.x
0    1
1    2
2    3
Name: x, dtype: int64
>>> type(df.x)
<class 'pandas.core.series.Series'>
```

With a `Series` object we can perform what's called *vectorized*
math and string operations.

```python
>>> df.x * 2
0    2
1    4
2    6
Name: x, dtype: int64
```

This means we can simply use `+` to add multiple `Series` objects
and it does what we expect.

```python
>>> df.x + df.y
0    5
1    7
2    9
dtype: int64
>>> df['z'] = df.x + df.y
>>> df
   x  y  z
0  1  4  5
1  2  5  7
2  3  6  9

[3 rows x 3 columns]
```

Voilá.
