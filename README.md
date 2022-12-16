# dremel-performance
This will be walkthroughs of data storage approaches used by bigdata DAGs. This will be shown both with and without sql.

<br>
## Rows v Columns
In dremel systems (Google whitepaper: https://research.google/pubs/pub36632/), data stored by columns instead of rows provide much different opportunites for optimization both in terms of workflow and performance. 
A table is no longer a table of n rows and m columns. A table is a series of key-value sets sharing one common row index. 
There are n fields that are arrays of indicies pointing back to a <i>row</i> (index). But they dont list all rows (not n x m datums);

This is where BigQuery (Dremel letp past vertica)
A single column is not the contents of all the data in the column. It is the locations where the unique values occur whenever possible. eg: If there are 4 Billion rows, but a single column only has 3 distinct values, then the entire 4 billion records could be represented as simply as:
<code>value1:
    [row1,row15,,row17,,row10000,row11000],
  value2:
    [row0,,row2,row14,,row18,row9999],
  value3:[row11001,row3999999999]
  }</code>
  
Which is telling us the values of all 4 billion rows with just the distinct object locations: The value of the field in rows 1-15, 14, and 10000-11000 are all `value1` same interpretation of `value 2` and that rows 11001 all the way up to the 4 Billionth are all going to have `value3` for the field. 

Imagine those three values (across all 4B) records were moderate length-string labels - "The United States Department of Agriucuture", "The United States Department of Defence" and "The Federeal Bureau of Investigaion". If we used the whole long-name every time that would take up a load of extra space. 
If we filtered on it, every string having "The " in the beginning means at least part of every single record before ruling it out. If not indexed, thats 4B reads and at least 20 Billion individual comparisions - 'T' 'h' 'e' ' ' 'F' - with <i>F</i>, the 5th being the first chatacher thats different across any value. 
"The United States Department of " is a huge amount of unnecessary matching. 

In a star schema we may replace them with FKs to a metadata table of primaries and the labels. So in row-oriented schema it would make sense to transform the fact data. For humans we'd probably keep an abbreviation: AG, DOD, FBI, using the keys to filter and join on later. `AG => 100, DOD => 101, FBI => 103`
We get a nice speed boost, can take clever shortcuts = `select data from facttable where fields=101` we know the key and never look it up and even just adding it back later: `with report as (select data from facttable where fields=101) select * from report left join mapping on org_id_fk = org_id_pk` which would only need to deal with the large string for a much smaller subset of rows. Smart SQL engines do this, but sometimes they dont <i>realize<i/>.

Major approach changes that will be illustrated:
Dont break up tables. This seems to go against 30 years of data manipulation but storage is cheap and processors fast. Reads are the bottleneck, but if we can greatly reduce the lookup space, then the reads can be fast too. Breaking apart meta and fact data is an antithesis to the fundamentals of dremel systems.

Two ways to use the repo's contents; to see the effect of this play out and eliminate the need for secrets for this project, the first set of examples will just be given as python. 
