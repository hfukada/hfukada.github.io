---
layout: post
title: "Modern Data Engineering Concepts: Multidimensional Cubes"
description: Diving into a core concept of modern data analytics
img: "cube/header.jpg"
tags: [engineering, code, data, OLAP]
comments: false
---

# EDITORS NOTE, I FOUND THIS BLOGPOST HALF WRITTEN FROM 2021 IN 2026 AND DECIDED TO PUBLISH IT ANYWAYS.

## OLAP Cubes / Multidimensional Cubes
What the heck are they and why are they useful? This blog post is inspired by the fact that it is really difficult to find articles that intuitively describe this concept. 

## History of OLAP
OLAP stands for "On Line Analytical Processing". The term arose during the periods where engineers would do Business Intelligence (abreviated BI) for the use of user tracking to gain intuition on what users were doing on a platform. Back in the 1990s, we relied heavily on relational databases. Everything was a model and we cared a lot about individual components being logically correct all the time. If someone wanted to drill down to the specifics of a customer, they would be able to string everything together through the tight relationships that the keys provided. The flipside of this is that doing aggregate analysis is incredibly expensive. Back in the 1990s 512MB of RAM was considered substantial. Aggregate queries were expensive and even sometimes unfeasible. Aggregate queries could take hours or even days. This is where the OG data engineers came into play. They would try to create generalized summarized views on top of the tightly relational databases. In order to reduce future work they would develop batch process that routinely dumped the current database state into a queriable format. The format they chose was to aggregate as much information as they could into a single view that could be held in memory. If the business need was to get a generalized view of customers and the things they bought, a relationship model might look something like this:

### users

| id | country | age |
|-------|--------|---------|
| 1 | USA | 23 |
| 2 | Canada | 56 |
| 3 | USA | 59 |

### sessions

|   time    | session_id|user_id| duration | is_mobile |
|-----------|-----------|-------|----------|-----------|
| 2020-01-01|    S1|      1|      100 |      true |
| 2020-01-02|    S2|      2|     1500 |     false |
| 2020-01-02|    S3|      2|      500 |      true |
| 2020-01-03|    S4|      3|     5900 |     false |

### item_sales

| session_id |user_id| item_id  | quantity |
|------------|-------|----------|----------|
| S1 |      1|      10  |        4 |
| S2|      2|      30  |        9 |
| S2|      2|      10  |        9 |
| S4|      3|      20  |        8 |
| S4|      3|      10  |        4 |


## Building a summarized view or table to analyze general behavior

A summarized table or view might look something like so:

### user_behavior

|user_id|country|age|mainly_mobile_app|session_duration_avg|item_sales_sum|
|-------|-------|---|----------------|------------|-----------------------|
|1      |    USA| 23|            true|         100|                      4|
|2      | Canada| 56|           false|        1000|                     18|
|3      |    USA| 59|           false|        5900|                     12|

The query to generate this table using SQL would look something like:
```
SELECT 
     users.id as user_id,
     users.country as country,
     users.age as age,
     desktop_duration.duration < mobile_duration.duration as mainly_mobile_app,
     session_duration_avg,
     sales.total_sold as item_sales_sum,
FROM users
LEFT JOIN (
    SELECT user_id, sum(duration) as duration FROM sessions GROUP BY user_id 
      WHERE is_mobile=true
    ) AS mobile_duration
    ON users.id = mobile_duration.user_id
LEFT JOIN (
    SELECT user_id, sum(duration) AS duration FROM sesssions GROUP BY user_id WHERE is_mobile=false
    ) AS desktop_duration
    ON users.id = desktop_duration.user_id
LEFT JOIN (
    SELECT user_id, avg(duration) AS avg_duration FROM sessions GROUP BY user_id
    ) as durations
    ON users.id = durations.user_id
LEFT JOIN (
    SELECT user_id, sum(quantity) AS total_sold FROM item_sales GROUP BY user_id
    ) AS sales
    ON users.id = sales.user_id
```

This is a fairly simple table, but it cost a LOT of memory and time to generate it. It helps answer general questions about our platform like "what age group purchaces the most/spends the most time on the website" but fails to answer any questions it was not designed for. Specifically, since this table is user oriented, it fails to answer questions with specific timestamp constraints. So how do we build up systems that are data-analysis-friendly?

## Introducing the OLAP Cube and Current Multidimensional Analysis Model

In a modern system we will aim to create a table that contains enough information that captures all the information above, in a recordized format. We will duplicate some information, but since disk space is much cheaper than RAM we can get away with this. Each time a sale occurs, we can quickly and computationally cheaply generate a summary of what happened. An example table below shows what things might look like.

### extended_user_behavior

| time|user_id|country|age|session_is_mobile|session_duration|item_sales_sum|
|-----|-------|-------|---|----------------|------------|-----------------------|
| 2020-01-01 |1      |    USA| 23|            true|         100|                      4|
| 2020-01-02 |2      | Canada| 56|           false|        1500|                     18|
| 2020-01-02 |2      | Canada| 56|           false|         500|                     12|
| 2020-01-03 |3      |    USA| 59|           false|        5900|                     12|

Using this table we can immediately make use of this. We can immediately answer the previous questions straight out of the box. 

- "What age group purchaces the most"/"what age group spends the most time?"

```
SELECT age/10 as agegroup, sum(item_sales_sum) FROM extended_user_behavior GROUP BY agegroup
```

We can even try to answer this question for certain periods of time, unlike before.

```
SELECT age/10 as agegroup, sum(item_sales_sum) FROM extended_user_behavior 
WHERE time BETWEEN '2020-01-01' AND '2020-01-02'
GROUP BY agegroup
```

Same table. Same columns. Different question entirely. That is the whole point.

## So Where Does the "Cube" Come In?

Look at `extended_user_behavior` again. Each column that you can filter or group by is a **dimension**. Time is a dimension. Country is a dimension. Age is a dimension. `session_is_mobile` is a dimension. The numeric columns like `session_duration` and `item_sales_sum` are your **measures**; the things you actually want to measure.

Now imagine each dimension as an axis. Time on one axis, country on another, maybe age on a third. You can slice this space along any combination of those axes. Filter to a single country and group by time? That is a slice of the cube. Group by country and age while ignoring time? Now you have squashed the cube along the time axis and created another flat "slice". Every combination of dimensions gives you a different cut of the same underlying data.

## Why This Matters

Referencing back to the `user_behavior` table, it was user-oriented and flat. It answered the questions it was designed for and nothing else. The cube approach attempts to multiply and expand upon this idea. Instead of pre-answering specific questions, you store enough dimensional context that analysts can ask their own questions on the fly.

This is the foundation that modern tools like Hadoop, Apache Druid, ClickHouse, and Google BigQuery are built on. They store data in columnar formats optimized for exactly these kinds of slice-and-dice operations. The expensive joins and aggregations that took hours now take seconds. The underlying concept is the same as before; flatten, denormalize, and precompute. But modern tooling expands upon this idea and made it faster.

## The Takeaway

A multidimensional cube is a denormalized dataset where each record carries its own dimensional context. You trade disk space for analytical flexibility. You duplicate some data so that you never have to join your way back to an answer. The result is a single table you can slice along any axis, at any granularity, for any time range.
