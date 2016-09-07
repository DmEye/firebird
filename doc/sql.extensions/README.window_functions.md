# Window Functions

By the SQL specification, window functions (also know as analytical functions) are a kind of aggregation, but which does not "filter" the result set of a query. The aggregated data is mixed with the query result set.

That sort of functions are used with the `OVER` clause. Window functions may appear only in the select list or the `ORDER BY` clause of a query.

Additional to the `OVER` clause, Firebird window functions may use partitions, order and frames (FB 4.0).

Syntax:

```
<window function> ::=
  <window function name>([<expr> [, <expr> ...]])
    OVER ([<window partition>] [<window order>] [<window frame>])

<window partition> ::=
  PARTITION BY <expr> [, <expr> ...]

<window order> ::=
  ORDER BY <expr> [<direction>] [<nulls placement>] [, <expr> [<direction>] [<nulls placement>]] ...

<window frame> ::=
  {RANGE | ROWS} <window frame extent>

<window frame extent> ::=
  {<window frame start> | <window frame between>}

<window frame start> ::=
  {UNBOUNDED PRECEDING | <expr> PRECEDING | CURRENT ROW}

<window frame between> ::=
  BETWEEN {UNBOUNDED PRECEDING | <expr> PRECEDING | <expr> FOLLOWING | CURRENT ROW} AND
          {UNBOUNDED FOLLOWING | <expr> PRECEDING | <expr> FOLLOWING | CURRENT ROW}

<direction> ::=
  {ASC | DESC}

<nulls placement> ::=
  NULLS {FIRST | LAST}
```

## 1. Aggregate functions used as window functions

All aggregate functions may be used as window functions, adding the `OVER` clause. Imagine a table EMPLOYEE with columns ID, NAME and SALARY, and the need to show each employee with his respective salary and the percentage of his salary over the payroll. With a "normal" query, this is possible in the following manner:

```sql
select
    id,
    department,
    salary,
    salary / (select sum(salary) from employee) percentage
  from employee
  order by id;
```

Results:

| id | department | salary | percentage |
|---:|------------|-------:|-----------:|
|  1 | R & D      |  10.00 |     0.2040 |
|  2 | SALES      |  12.00 |     0.2448 |
|  3 | SALES      |   8.00 |     0.1632 |
|  4 | R & D      |   9.00 |     0.1836 |
|  5 | R & D      |  10.00 |     0.2040 |

It's necessary to repeat the query in a subquery and wait so much to see the results, specially if EMPLOYEE is a complex view.

The same query could be specified in much more elegant and faster way using a window function:

```sql
select
    id,
    department,
    salary,
    salary / sum(salary) over () percentage
  from employee
  order by id;
```

Here, `sum(salary) over ()` is computed with the sum of all SALARY from the query (the employee table).

## 2. Partitioning

Like aggregate functions, that may operate alone or in relation to a group, window functions may also operate on a group, which is called "partition". Its syntax is:

When aggregation is done over a group, it could produce more than one row. So the result set generated by a partition is joined with the main query using the same expression list of the partition.

Continuing the employee example, instead of get the percentage of the employee salary over all employees, we would like to get the percentage based only on the employees in the same department:

```sql
select
    id,
    department,
    salary,
    salary / sum(salary) over (partition by department) percentage
  from employee
  order by id;
```

Results:

| id | department | salary | percentage |
|---:|------------|-------:|-----------:|
|  1 | R & D      |  10.00 |     0.3448 |
|  2 | SALES      |  12.00 |     0.6000 |
|  3 | SALES      |   8.00 |     0.4000 |
|  4 | R & D      |   9.00 |     0.3103 |
|  5 | R & D      |  10.00 |     0.3448 |

## 3. Ordering

The `ORDER BY` sub-clause can be used with or without partitions, and used with the standard aggregate functions and without frames, make them return the partial aggregations as the records are being processed. Example:

```sql
select
    id,
    salary,
    sum(salary) over (order by salary) running_salary
  from employee
  order by salary;
```

The result set produced will be:

| id | salary | running_salary |
|---:|-------:|---------------:|
|  3 |   8.00 |           8.00 |
|  4 |   9.00 |          17.00 |
|  1 |  10.00 |          37.00 |
|  5 |  10.00 |          37.00 |
|  2 |  12.00 |          49.00 |

Then running_salary returns the partial/accumulated (or running) aggregation (of the `SUM` function). It may appear strange that 37.00 is repeated for the ids 1 and 5, but that is how it should work. The `ORDER BY` keys are grouped together and the aggregation is computed once (but summing the two 10.00). To avoid this, you can add the ID field to the end of the `ORDER BY` clause.

The fact of running aggregations and grouping results by the order key happens because when an explicit frame is not used, it computes the window functions as a `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` frame. More on frames later in this document.

It's possible to use multiple windows with different orders, and `ORDER BY` parts like `ASC` / `DESC` and `NULLS FIRST` / `NULLS LAST`.

With a partition, `ORDER BY` works the same way, but at each partition boundary the aggregation is reset.

All aggregation functions are usable with `ORDER BY`, except the `LIST` function.

## 4. Exclusive window functions

Beyond aggregate functions, there is also exclusive window functions, currently divided in ranking and navigational categories.

Both set of functions can be used with/without partition/ordering, but the usage does not make much sense without ordering.

## 4.1. Ranking functions

Syntax:

```
<ranking window function> ::=
    DENSE_RANK() |
    RANK() |
    PERCENT_RANK() |
    CUME_DIST() |
    NTILE(<expr>) |
    ROW_NUMBER()
```

The rank functions compute the ordinal rank of a row within the window partition. In this category are the functions: `DENSE_RANK`, `RANK` and `ROW_NUMBER`.

With these functions, one can create different type of incremental counters. Think about `SUM(1) OVER (ORDER BY SALARY)`, these functions do this type of thing, but all of them in different ways. Following is an example query, also comparing with the `SUM` behavior.

```sql
select
    id,
    salary,
    dense_rank() over (order by salary),
    rank() over (order by salary),
    percent_rank() over (order by salary),
    cume_dist() over (order by salary),
    ntile(3) over (order by salary),
    row_number() over (order by salary),
    sum(1) over (order by salary)
  from employee
  order by salary;
```

And the result set:

| id | salary | dense_rank | rank |      percent_rank |          cume_dist | ntile | row_number | sum |
|---:|-------:|-----------:|-----:|------------------:|-------------------:|------:|-----------:|----:|
|  3 |   8.00 |          1 |    1 | 0.000000000000000 | 0.2000000000000000 |     1 |          1 |   1 |
|  4 |   9.00 |          2 |    2 | 0.250000000000000 | 0.4000000000000000 |     1 |          2 |   2 |
|  1 |  10.00 |          3 |    3 | 0.500000000000000 | 0.8000000000000000 |     2 |          3 |   4 |
|  5 |  10.00 |          3 |    3 | 0.500000000000000 | 0.8000000000000000 |     2 |          4 |   4 |
|  2 |  12.00 |          4 |    5 | 1.000000000000000 | 1.0000000000000000 |     3 |          5 |   5 |

The difference between `DENSE_RANK` and `RANK` is that there is a gap related to duplicate rows (in relation to the window ordering) only in `RANK`. `DENSE_RANK` continues assigning sequential numbers after the duplicate salary. On the other hand, `ROW_NUMBER` always assigns sequential numbers, even when there is duplicate values.

`PERCENT_RANK` is a ratio of `RANK` to group count.

`CUME_DIST` is cumulative distribution of a value in a group.

`NTILE` distributes the rows into a specified number of groups. `NTILE` argument is restricted to integral positive literal, variable (`:var`) and DSQL parameter (`?`).

## 4.2. Navigational functions

Syntax:

```
<navigational window function> ::=
    FIRST_VALUE(<expr>) |
    LAST_VALUE(<expr>) |
    NTH_VALUE(<expr>, <offset>) [FROM FIRST | FROM LAST] |
    LAG(<expr> [ [, <offset> [, <default> ] ] ) |
    LEAD(<expr> [ [, <offset> [, <default> ] ] )
```

The navigational functions gets the simple (non-aggregated) value of an expression from another row (inside the same partition) of the query.

It's important to note that `FIRST_VALUE`, `LAST_VALUE` and `NTH_VALUE` also operates on a window frame, and the default frame is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. This is likely to get strange results for `NTH_VALUE` and specially `LAST_VALUE`. More on frames later in this document.

```sql
select
    id,
    salary,
    first_value(salary) over (order by salary),
    last_value(salary) over (order by salary),
    nth_value(salary, 2) over (order by salary),
    lag(salary) over (order by salary),
    lead(salary) over (order by salary)
  from employee
  order by salary;
```

And the result set:

| id | salary | first_value | last_value |    nth_value |          lag |         lead |
|---:|-------:|------------:|-----------:|-------------:|-------------:|-------------:|
|  3 |   8.00 |        8.00 |       8.00 | &lt;null&gt; | &lt;null&gt; |         9.00 |
|  4 |   9.00 |        8.00 |       9.00 |         9.00 |         8.00 |        10.00 |
|  1 |  10.00 |        8.00 |      10.00 |         9.00 |         9.00 |        10.00 |
|  5 |  10.00 |        8.00 |      10.00 |         9.00 |        10.00 |        12.00 |
|  2 |  12.00 |        8.00 |      12.00 |         9.00 |        10.00 | &lt;null&gt; |

`FIRST_VALUE` and `LAST_VALUE` gets respectively the first and last value of the ordered partition.

`NTH_VALUE` gets the n-th value, starting from the first (default) or the last record, from the ordered partition. If offset is 1 from first, it's equivalent to `FIRST_VALUE`. If offset is 1 from last, it's equivalent to `LAST_VALUE`.

`LAG` and `LEAD` get the value within a distance respect to the current row and the offset (which defaults to 1) passed. In the case the offset points to outside of the partition, the default parameter (which defaults to NULL) is returned. `LAG` looks for a preceding row, and `LEAD` for a following row.

## 5. Frames (FB 4.0)

It's possible to specify the frame that some window functions work. The frame is divided in three piecies: unit, start bound and end bound.

The unit `RANGE` or `ROWS` defines how the bounds `<expr> PRECEDING`, `<expr> FOLLOWING` and `CURRENT ROW` works.

With `RANGE`, the `ORDER BY` should specify only one expression, and that expression should be of a numeric, date, time or timestamp type. For `<expr> PRECEDING` and `<expr> FOLLOWING` bounds, `<expr>` is respectively subtracted or added to the order expression, and for `CURRENT ROW` only the order expression is used. Then, all rows (inside the partition) between the bounds are considered part of the resulting window frame.

With `ROWS`, order expressions is not limited by number or types. In this case, `<expr> PRECEDING`, `<expr> FOLLOWING` and `CURRENT ROW` relates to the row position under the partition, and not to the order keys values.

`UNBOUNDED PRECEDING` and `UNBOUNDED FOLLOWING` work identically with `RANGE` and `ROWS`. `UNBOUNDED PRECEDING` looks for the first row and `UNBOUNDED FOLLOWING` the last one, always inside the partition.

The frame syntax with `<window frame start>` specifies the start frame, with the end frame being `CURRENT ROW`.

Some window functions discard frames. `ROW_NUMBER`, `LAG` and `LEAD` always work as `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. And `DENSE_RANK`, `RANK`, `PERCENT_RANK` and `CUME_DIST` always work as `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`.

`FIRST_VALUE`, `LAST_VALUE` and `NTH_VALUE` respect frames, but the `RANGE` unit works identically as `ROWS`.


Author:
    Adriano dos Santos Fernandes <adrianosf at gmail.com>