---
layout: post
title: ""Norikra vs InfluxDB""
date: 2014-05-07 22:53:08 +0900
comments: true
categories: 
---

Today, I give a comparison between Norikra and InfluxDB in terms of query features.

## What is Norikra, and What is InfluxDB

[Norikra](http://norikra.github.io/) is a Schema-less Stream Processor based on Esper, which is a kind of CEP (Complex Event Processing) engine. With Norikra, we can use highly-functional SQL-like query for processing time series streaming data.

[InfluxDB](http://influxdb.org/) is a distributed time series database, which has a SQL-like query language designed for working with time series and analytics. 
It has a feature called "Continuous Query" which applies queries charged in advance, and stores the resulted data into a Series (which is like a "table" of RDBMS) progressively. 

It looks both can do similar things. 

So, I compared Norikra and InfluxDB in terms of query features. I felt that it would be nice to use InfluxDB if it has a sufficent features in anticipation of the data persistence.

## TR; DR

Norikra won

## Feature Catalog

I just write catalog tables here. 

### Aggregation Fuctions 

|            | Norikra | InfluxDB | NOTE                        |
|------------|---------|----------|-----------------------------|
| COUNT      | YES     | YES      |                             |
| MIN        | YES     | YES      |                             |
| MAX        | YES     | YES      |                             |
| AVG        | YES     | YES      |                             |
| MEAN       | YES     | YES      |                             |
| MODE       | NO      | YES      |                             |
| MEDIAN     | NO      | YES      |                             |
| DISTINCT   | YES     | YES      |                             |
| PERCENTILE | YES     | YES      | Already [implemented as a UDF](https://github.com/norikra/norikra-udf-percentile) |
| HISTOGRAM  | NO      | YES      |                             |
| DERIVATIVE | NO      | YES      |                             | 
| SUM        | YES     | YES      |                             |
| STDDEV     | NO      | YES      |                             |
| FIRST      | YES     | YES      |                             |
| LAST       | YES     | YES      |                             |
| MAXBY      | YES     | NO       |                             |
| MINBY      | YES     | NO       |                             |

NOTE: Norikra supports UDF, so users can create functions by themselves if want

### Features of Query

|                               | Norikra | InfluxDB | NOTE                                                              |
|-------------------------------|---------|----------|-------------------------------------------------------------------|
| Time batch window             | YES     | YES      | Of course, both support the feature to aggregate in each interval |
| Externally timed batch window | YES     | NO       | A feature to aggregate based on the time field of messages        |
| Sub query                     | YES     | NO       |                                                                   |
| JOIN                          | YES     | YES      |                                                                   |
| MERGE                         | NO      | YES      | A feature of InfluxDB to merge results from multiple tables. Possible to specify regular expressions for tables |
| Query group                   | YES     | NO       | Name a group for queries                                          |
| Program codes                 | YES     | NO       | We can write Java codes on Norikra queries                        |
| Nested JSON                   | YES     | NO       | We can specify fields like `parrent.child` when JSON is nested    |
| GROUP BY                      | YES     | YES      |                                                                   |
| HAVING                        | YES     | NO       |                                                                   |
| ORDER BY                      | YES     | NO       | InfluxDB has `order`, but it is only for the `time` field         |
| UDF                           | YES     | NO       |                                                                   |

## Conclusion

Norikra won overwhelmingly. That said, because InfluxDB is storage, but Norikra is an on-memory engine, the value of InfluxDB shall not be impaired in any way.

