---
# Documentation: https://sourcethemes.com/academic/docs/managing-content/

title: "Automate the boring stuff: SQL Query building for WRDS"
subtitle: "A simple python program to automate the SQL query building for Wharton's WRDS"
summary: ""
authors: []
tags: []
categories: []
date: 2021-01-22T17:30:21+11:00
lastmod: 2021-01-22T17:30:21+11:00
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---

The following is a simple `python` program to automate the SQL query building in order to extract data from Wharton's Research Data Service [WRDS](https://wrds-www.wharton.upenn.edu/).

## First things first
First, we need to load the library and connect to the WRDS database. This is easiest when using `Jupyter Notebooks`. WRDS provides a nice tutorial on how to connect to WRDS from `Jupyter Notebooks`, which can be found here:  
https://wrds-www.wharton.upenn.edu/documents/1443/wrds_connection.html  

You should now hopefully be able to connect to the WRDS database with your account:

```python
# load libraries
import wrds
import pandas as pd

# connect to WRDS
db = wrds.Connection(wrds_username = 'your_username')
```

    Loading library list...
    Done


## Now let's build some queries

Simplified, SQL queries take the general form:```select columns from library.dataset where column1 = value```. While it's of course an option to just manually write SQL queries, this gets very fast, very cumbersome when specifying many conditions (I also tend to sneak in typos).

The following `python` function can take different arguments--such as column names or SIC codes--and automatically builds the query for a specific dataset (the only required argument).

```python
def build_query(dataset, **kwargs):
    """Build Query for WRDS database.
        Args:
            dataset (str): dataset
            start (str): Start of time period with format YYYY-MM-DD, default to None
            end (str): End of time period with format YYYY-MM-DD, default to None
            columns (str or list): Filter of one or a list of columns, default to None
            sic (str or list): Filter of one or a list of SIC, default to None
            sich (str or list): Filter of one or a list of historical SIC, default to None
            gvkey (str or list):  Filter of one or a list of Global Company Key, default to None
            tic (str or list):  Filter of one or a list of ticker symbols, default to None
            cusip (str or list):  Filter of one or a list of CUSIP symbols, default to None
            limit (int): Return only a number of return records, defualt to None
        Returns:
            (str) with query
    """
    columns = kwargs.get("columns", None)
    sic = kwargs.get("sic", None)
    sich = kwargs.get("sich", None)
    start = kwargs.get("start", None)
    end = kwargs.get("end", None)
    gvkey = kwargs.get("gvkey", None)
    tic = kwargs.get("tic", None)
    cusip = kwargs.get("cusip", None)
    limit = kwargs.get("limit", None)


    if columns is not None:
        columns_filter = ",".join([x for x in columns])
    else:
        columns_filter = "*"

    date_filter = ""
    if start and end is not None:
        date_filter = "datadate between {} and {}".format("'" + start + "'" , "'" + end + "'")

    sic_filter = ""
    if sic is not None:
        sic_filter = "sic in ({}) ".format(
            ",".join(["'" + x + "'" for x in sic])
        )

    sich_filter = ""
    if sich is not None:
        sich_filter = "sich in ({}) ".format(
            ",".join(["'" + x + "'" for x in sich])
        )

    gvkey_filter = ""
    if gvkey is not None:
        gvkey_filter = "gvkey in ({}) ".format(
            ",".join(["'" + x + "'" for x in gvkey])
        )

    tic_filter = ""
    if tic is not None:
        tic_filter = "tic in ({}) ".format(
            ",".join(["'" + x + "'" for x in tic])
        )

    cusip_filter = ""
    if cusip is not None:
        cusip_filter = "cusip in ({}) ".format(
            ",".join(["'" + x + "'" for x in cusip])
        )

    # FIRST INSTANCE WITHOUT AND THEN SEQUENTIALLY ADD FILTERS (if non empty)
    filters = [date_filter, cusip_filter, tic_filter, gvkey_filter, sic_filter]
    filters_list = ""
    if any(filters):
        filters_list = " and ".join([x for x in filters if x != ""])
        filters_list = "where " + filters_list

    limit_number = ""
    if limit:
        limit_number += "LIMIT {} ".format(limit)

    cmd = (
        "select "
        + columns_filter
        + " from {} ".format(dataset)
        + filters_list
        + limit_number
        )

    print(cmd)

    return(cmd)
```

Now let's see the function in action. The example case uses the Compustat database (`comp`). Let's assume we're interested in companies in specific industrial sectors and we know their SIC codes. We would now like to retrieve their financial fundamentals.

Since `comp.funda` does not allow us to filter based on SIC codes directly, we first extract the codes of the relevant companies from the `comp.names` database.


```python
sic_codes = ['1011', '1021', '1031', '1041', '1044', '1061', '1081', '1094']

# construct query
query = build_query(dataset ="comp.names",
                    sic = sic_codes
                   )
# look at query
print(query)
```


```python
# let's send the SQL query to WRDS
df = db.raw_sql(query)
```

At this point the function "unfolds its power" and makes your life a lot easier.

To illustrate this and make things more interesting, let's assume that we do not want to download all of the variables but just the ones of interest. Moreover, we are only interested in a particular time period.  

```python
# get identifier of companies from query result
gvkeys = df["gvkey"].astype(str).to_list()


columns = ["gvkey","datadate","fyear","indfmt","consol",
           "popsrc","datafmt","tic","cusip", "conm","revt"]

# use gvkeys for new query
query = build_query(dataset ="comp.funda",
                    columns = columns,
                    gvkey = gvkeys,
                    start = '2000-01-01',
                    end = '2020-01-01',
                   )

# look at query
print(query)
```

Let's see what the output of the query looks like:

```python
# query database and look at output
df = db.raw_sql(query)
df.head()
```

## (Not so famous) last words

There are many more potential filters one could think of and implement in the function. I hope the base function above can build a nice foundation for your own use case. Please feel free to send me your expanded functions and ideas.

Oh and...

```python
# ...don't forget to close your session if you're done ;)
db.close()
```
