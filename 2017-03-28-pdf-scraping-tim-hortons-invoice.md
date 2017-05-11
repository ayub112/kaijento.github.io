---
title: "PDF scraping: Tim Hortons Invoice"
date:  2017-03-28 11:39:05 +0100
tags:
  - pdf
  - python
  - pdftotext
  - json
  - regex
---

The goal is to take a Tim Hortons Invoice that is in PDF format and
"scrape" some information from it and turn it into JSON using Python.

The PDF file looks like:

![](/img/1490698343-tim-hortons.pdf.png)

It has 8 pages but the number of pages differs we are only interested
in the last page.

We're going to be using `pdftotext` as discussed 
[in the previous PDF scraping article]({% post_url 2017-03-27-pdf-scraping-gwinnetttaxcommissioner.publicaccessnow.com %}#pdftotext).

So let's run the PDF through it to see how it does:

![](/img/1490698611-pdftotext-layout.png)

It has done a pretty decent job. Most bits of information should be easy to extract apart from 
the `Sold To` and `Deliver To` addresses. 

We will start with the output:

```bash
$ python timhortons.py
{
    "Bill of Lading": "82608215", 
    "Currency": "CAD", 
    "Customer PO": "4125310", 
    "Delivered To": [
        "102922", 
        "2922 TIM HORTONS", 
        "20345 CANON STREET", 
        "PO BOX 219", 
        "SOUTH LANCASTER ON K0C 2C0", 
        "CANADA"
    ], 
    "Due Date": "02/09/2017", 
    "Fuel Surcharge": "0.00", 
    "GST/HST Base": "656.02", 
    "GST/HST Total": "85.26", 
    "Grand Total": "3329.96", 
    "Invoice Date": "01/25/2017", 
    "Invoice Number": "5053548359", 
    "Order Number": "102512339", 
    "Payment Term": "Net 15 Days", 
    "Sales Tax Base": "0.00", 
    "Sold To": [
        "102922", 
        "102922 1697315 Ontario Inc.", 
        "20345 CANON STREET", 
        "PO BOX 219", 
        "SOUTH LANCASTER ON K0C 2C0", 
        "CANADA"
    ], 
    "Total": "3244.70"
}
```

Here is the code that generated it:

`pdftojson.py`

```python
import json, re, subprocess

pdfname = 'invoice.pdf'

output = subprocess.check_output(
    ['pdftotext', '-layout', pdfname, '-']).decode()

pages = output.strip('\f').split('\f')
page  = pages[-1]

address = re.search(r'(?s)(?<=Sold To:).+?(?=\n\n)', pages[-1]).group()
address = address.replace('Deliver To:', '').splitlines()

sold_to, delivered_to = zip(
    *(re.split(r'\s{2,}', line.strip()) for line in address))

bill_of_lading = re.search(r'Bill Of Lading:\s*(\S+)'    ,page).group(1)
currency       = re.search(r'Currency:\s*(\S+)'          ,page).group(1)
customer_po    = re.search(r'Customer PO:\s*(\S+)'       ,page).group(1)
due_date       = re.search(r'Due Date:\s*(\S+)'          ,page).group(1)
invoice_number = re.search(r'Invoice Number:\s*(\S+)'    ,page).group(1)
invoice_date   = re.search(r'Invoice Date:\s*(\S+)'      ,page).group(1)
order_number   = re.search(r'Order #:\s*(\S+)'           ,page).group(1)
payment_term   = re.search(r'Payment Term:\s*(.+?)\s{2,}',page).group(1)
sales_tax_base = re.search(r'Sales Tax Base\s*:\s*(\S+)' ,page).group(1)
gst_hst_base   = re.search(r'GST/HST Base\s*:\s*(\S+)'   ,page).group(1)

expenses = re.search(
    r'(?s)(Description\s*Extended Price\s*.+?Grand Total\s*\S+)', page).group()
expenses = re.sub(
    r'Price including\s*\n\s*Sales Tax', 'Price Including Sales Tax', expenses)

expenses = [
    re.split(r'\s{2,}', line.strip()) for line in expenses.splitlines()
]

total, fuel_surcharge, gst_hst_total, grand_total = [
    expense[-1] for expense in expenses[-4:]
]

summary = {
    'Sold To'       : sold_to,        'Delivered To'  : delivered_to, 
    'Bill of Lading': bill_of_lading, 'Currency'      : currency,
    'Customer PO'   : customer_po,    'Due Date'      : due_date,
    'Invoice Number': invoice_number, 'Invoice Date'  : invoice_date,
    'Order Number'  : order_number,   'Payment Term'  : payment_term,
    'Sales Tax Base': sales_tax_base, 'GST/HST Base'  : gst_hst_base,
    'Total'         : total,          'Fuel Surcharge': fuel_surcharge,
    'GST/HST Total' : gst_hst_total,  'Grand Total'   : grand_total
}

'''
summary['Expenses'] = [
    dict(zip(expenses[0], expense)) for expense in expenses[1:-4]
]
'''
print(json.dumps(summary, indent=4, sort_keys=True))
```

You'll notice at the end we have the `Expenses` commented out. This was just
to keep the output at a reasonable size, when uncommented the `Expenses` section
looks like this:

```bash
[
    {
        "Description": "Food & Beverage", 
        "Extended Price": "2614.61", 
        "Price Including Sales Tax": "2614.61", 
        "Sales Tax": "0.00"
    }, 
    {
        "Description": "Packaging", 
        "Extended Price": "490.71", 
        "Price Including Sales Tax": "490.71", 
        "Sales Tax": "0.00"
    }, 
    {
        "Description": "Merchandise", 
        "Extended Price": "0.00", 
        "Price Including Sales Tax": "0.00", 
        "Sales Tax": "0.00"
    }, 
    {
        "Description": "Cleaning", 
        "Extended Price": "120.93", 
        "Price Including Sales Tax": "120.93", 
        "Sales Tax": "0.00"
    }, 
    {
        "Description": "Smallwares", 
        "Extended Price": "5.00", 
        "Price Including Sales Tax": "5.00", 
        "Sales Tax": "0.00"
    }, 
    {
        "Description": "Office Supplies", 
        "Extended Price": "13.45", 
        "Price Including Sales Tax": "13.45", 
        "Sales Tax": "0.00"
    }
]
```

`pdftotext` separates pages in its output with the `ASCII 12 form feed` character. 
It may be represented as the control char `^L`. We can use `\f` to represent it inside a 
string in Python. (`0C` in hex is `12` in decimal)

```python
>>> '\f'
'\x0c'
```

So we can use `split('\f')` and `[-1]` to get the last page. Well not quite, there may be 
multiple trailing `\f` characters in the output so we can `strip('\f')` before we `split()`
to ensure `pages[-1]` will always give the last page.

```python
>>> 'page1\fpage2\f\f\f'.split('\f')
['page1', 'page2', '', '', '']
>>> 'page1\fpage2\f\f\f'.strip('\f').split('\f')
['page1', 'page2']
```

So the `address` lines start on the line containing `Sold To:` and they end with a blank line.
To extract that block we've used `r'(?s)(?<=Sold To:).+?(?=\n\n)'` 

`(?s)` is the same as passing `flags=re.DOTALL` it allows `.` to match `\n` meaning we can grab
multiple lines with `.+?`. The `(?<=)` is a positive lookbehind and the `(?=)` is a 
positive lookahead. These just exclude what is inside them from the match.

Once we have the address we then remove `Deliver To:` from it and split it up into a list of lines.
What about this `zip(*())` madness?!

```python
>>> address = ['ADDRESS1 LINE1   ADDRESS2 LINE1', 'ADDRESS1 LINE2   ADDRESS2 LINE2']
>>> list(zip(re.split(r'\s{2,}', line) for line in address))
[(['ADDRESS1 LINE1', 'ADDRESS2 LINE1'],), (['ADDRESS1 LINE2', 'ADDRESS2 LINE2'],)]
>>> list(zip(*(re.split(r'\s{2,}', line) for line in address)))
[('ADDRESS1 LINE1', 'ADDRESS1 LINE2'), ('ADDRESS2 LINE1', 'ADDRESS2 LINE2')]
```

Let's define a simple function that prints out all arguments passed to it to try
and see what's going on:

```python
>>> def args(*args):
...     print(len(args))
...     print(args)
... 
>>> args([re.split(r'\s{2,}', line) for line in address])
1
([['ADDRESS1 LINE1', 'ADDRESS2 LINE1'], ['ADDRESS1 LINE2', 'ADDRESS2 LINE2']],)
>>> args(*[re.split(r'\s{2,}', line) for line in address])
2
(['ADDRESS1 LINE1', 'ADDRESS2 LINE1'], ['ADDRESS1 LINE2', 'ADDRESS2 LINE2'])
```

So the `*` causes the list to be "unpacked" and each element inside it is passed
as an individual argument. This means instead of passing a single list to `zip()`
which doesn't make sense we are passing multiple lists.

```python
>>> address = [['ADDRESS1 LINE1', 'ADDRESS2 LINE1'], ['ADDRESS1 LINE2', 'ADDRESS2 LINE2']]
>>> list(zip(address))
[(['ADDRESS1 LINE1', 'ADDRESS2 LINE1'],), (['ADDRESS1 LINE2', 'ADDRESS2 LINE2'],)]
>>> list(zip(*address))
[('ADDRESS1 LINE1', 'ADDRESS1 LINE2'), ('ADDRESS2 LINE1', 'ADDRESS2 LINE2')]
```

The rest of the patterns are all pretty similar:

`r'Bill Of Lading:\s*(\S+)'` 

`\s*` matches any whitespace characters if present and `\S+` matches non-whitespace characters 
i.e. the first "word". So we match all of the labels and capture the first word that appears
after them.

The `Payment Term` pattern differs because we need to capture multiple words. We replace `\S+`
with `(.+?)\s{2,}` which captures "anything" up until we see 2 consecutive whitespace characters.
The `?` after `.+` here makes it "non-greedy" i.e. it will match up to the first instance.
Without `?` it would match up to the last instance.

```python
>>> re.search(r'f.*o', 'foofoobar').group()
'foofoo'
>>> re.search(r'f.*?o', 'foofoobar').group()
'fo'
```

The `?` makes it match up to the first instance of `o` instead of the last.

The following pattern is used to extract the Expenses "block":

`r'(?s)(Description\s*Extended Price\s*.+?Grand Total\s*\S+)'`

Note the `?` in `.+?Grand Total` as there is another `Grand Total` further down
so we want to stop at the first instance.

The `Price including Sales Tax` header is actually split over 2 lines so we 
use `re.sub()` to put all of the text back on the same line.

When it's all done `expenses` looks like this:

```python
>>> for expense in expenses:
...     expense
... 
['Description', 'Extended Price', 'Sales Tax', 'Price Including Sales Tax']
['Food & Beverage', '2614.61', '0.00', '2614.61']
['Packaging', '490.71', '0.00', '490.71']
['Merchandise', '0.00', '0.00', '0.00']
['Cleaning', '120.93', '0.00', '120.93']
['Smallwares', '5.00', '0.00', '5.00']
['Office Supplies', '13.45', '0.00', '13.45']
['Total', '3244.70', '0.00', '3244.70']
['Fuel Surcharge', '0.00', '0.00', '0.00']
['GST/HST/VAT', '85.26']
['Grand Total', '3329.96']
```

`expenses[0]` contains the headers which we want to use as the keys in our dict.
If you have a list of keys and list of values you can use `dict(zip(keys, values))`
to turn them into a dict:

```python
>>> headers = ['Description', 'Extended Price', 'Sales Tax', 'Price Including Sales Tax']
>>> expense = ['Food & Beverage', '2614.61', '0.00', '2614.61']
>>> print(json.dumps(dict(zip(headers, expense)), indent=4, sort_keys=True))
{
    "Description": "Food & Beverage", 
    "Extended Price": "2614.61", 
    "Price Including Sales Tax": "2614.61", 
    "Sales Tax": "0.00"
}
```

We want to ignore the first element in `expenses` and also the last 4 so we use the slice
`[1:-4]` to do that.

```python
summary['Expenses'] = [
    dict(zip(expenses[0], expense)) for expense in expenses[1:-4]
]
```

Finally we use `json.dumps()` to turn our dict into a JSON string.
The `indent=4` pretty-prints the output instead of producing a 
single-line string. `sort_keys` sorts the keys to give us
consistent ordering of the output.

The final code and PDF file used in this article are available [here](https://github.com/kaijento/code/tree/master/pdfscraping/timhortons).
