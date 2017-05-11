---
title: "pandas: read_csv() and epoch time"
date:  2017-03-29 19:50:34 +0100
tags:
 - python
 - pandas
 - csv
---

Given a CSV file that contains dates formatted as [epoch time](https://en.wikipedia.org/wiki/Unix_time)
how do we convert them into `datetime` objects within a pandas dataframe?

## pandas.read_csv()

To turn a CSV file into a dataframe we can use `pandas.read_csv()`

```python
>>> import pandas
>>> pandas.read_csv('epoch.csv')
     who        when
0    bob  1490772583
1  alice  1490771000
2    ted  1490772400
```

To access a particular "field" or "column" we can use "dict indexing":

```python
>>> pandas.read_csv('epoch.csv')['when']
0    1490772583
1    1490771000
2    1490772400
Name: when, dtype: int64
```

Another way you may see is the following:

```python
>>> pandas.read_csv('epoch.csv').when
0    1490772583
1    1490771000
2    1490772400
Name: when, dtype: int64
```

So pandas takes the column headers and makes them available as attributes. This
may not always work however as there may be name clashes with existing `pandas.DataFrame`
attributes or methods.

```python
>>> pandas.read_csv('values.csv')
     who  values
0    bob       1
1  alice       3
2    ted       2

[3 rows x 2 columns]
>>> pandas.read_csv('values.csv').values
array([['bob', 1],
       ['alice', 3],
       ['ted', 2]], dtype=object)
```

`pandas.DataFrame.values` already exists which pandas will not overwrite.
Using the dict indexing approach however will always give us column data:

```python
>>> pandas.read_csv('values.csv')['values']
0    1
1    3
2    2
Name: values, dtype: int64
```

## pandas.to_datetime()

To convert to a datetime we can use `pandas.to_datetime()`

```python
>>> pandas.to_datetime(1490772583)
Timestamp('1970-01-01 00:00:01.490772583')
```

The default unit is nanoseconds and not seconds which is what we have.
To convert seconds to nanoseconds we can multiply by `1e9`:

```python
>>> pandas.to_datetime(1490772583 * 1e9)
Timestamp('2017-03-29 07:29:43')
```

Or we can just specify the unit value as `s` for seconds:

```python
>>> pandas.to_datetime(1490772583, unit='s')
Timestamp('2017-03-29 07:29:43')
>>> pandas.to_datetime(1490772583, unit='s').weekday_name
'Wednesday'
```

## tz_localize()

The resulting object has no timezone associated with it. If you'd like to 
give it one you could `tz_localize()` it:

```python
>>> pandas.to_datetime(1490772583, unit='s').tz_localize('Europe/Dublin')
Timestamp('2017-03-29 07:29:43+0100', tz='Europe/Dublin')
```

If it was UTC time which you wanted to convert to another timezone you could 
`tz_localize('UTC')` first then `tz_convert()`:

```python
>>> pandas.to_datetime(1490772583, unit='s').tz_localize('UTC').tz_convert('Europe/Dublin')
Timestamp('2017-03-29 08:29:43+0100', tz='Europe/Dublin')
```

So can we convert them all at once?

```python
>>> pandas.to_datetime(df.when, unit='s')
0   2017-03-29 07:29:43
1   2017-03-29 07:03:20
2   2017-03-29 07:26:40
Name: when, dtype: datetime64[ns]
```

Nice. What about a timezone?

```python
>>> pandas.to_datetime(df.when, unit='s').tz_localize('Europe/Dublin')
Traceback (most recent call last):
TypeError: index is not a valid DatetimeIndex or PeriodIndex
>>> type(pandas.to_datetime(df.when, unit='s'))
<class 'pandas.core.series.Series'>
```

## pandas.DataFrame.values

We can instead pass `.values` so that we get back a `DatetimeIndex` instead of `Series`:

```python
>>> df.when.values
array(['2017-03-29T07:29:43.000000000', '2017-03-29T07:03:20.000000000',
       '2017-03-29T07:26:40.000000000'], dtype='datetime64[ns]')
>>> pandas.to_datetime(df.when.values, unit='s')
DatetimeIndex(['2017-03-29 07:29:43', '2017-03-29 07:03:20',
               '2017-03-29 07:26:40'],
              dtype='datetime64[ns]', freq=None)
```

This allows us to give them a timezone all at once:

```python
>>> pandas.to_datetime(df.when.values, unit='s').tz_localize('Europe/Dublin')
DatetimeIndex(['2017-03-29 07:29:43+01:00', '2017-03-29 07:03:20+01:00',
               '2017-03-29 07:26:40+01:00'],
              dtype='datetime64[ns, Europe/Dublin]', freq=None)
```

To update our changes we can assign the result back into our dataframe
and overwrite the `df.when` column:

```python
>>> df.when = pandas.to_datetime(df.when, unit='s')
>>> df
     who                when
0    bob 2017-03-29 07:29:43
1  alice 2017-03-29 07:03:20
2    ted 2017-03-29 07:26:40
```

Let's check a particular row value:

```python
>>> df.when.iloc[0]
Timestamp('2017-03-29 07:29:43')
>>> df.when.iloc[0].weekday_name
'Wednesday'
>>> df.loc[0, 'when']
Timestamp('2017-03-29 07:29:43')
>>> df.loc[0, 'when'].weekday_name
'Wednesday'
```

## parse_dates and date_parser 

Finally it is also possible to use the `parse_dates` and `date_parser` arguments when calling `read_csv()` to
convert dates:

```python
>>> pandas.read_csv('epoch.csv', parse_dates=['when'], date_parser=lambda epoch: pandas.to_datetime(epoch, unit='s'))
     who                when
0    bob 2017-03-29 07:29:43
1  alice 2017-03-29 07:03:20
2    ted 2017-03-29 07:26:40
```
