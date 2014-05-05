---
layout: post
title: "How to use Norikra to count HTTP status with Fluentd"
date: 2014-05-04 20:13:07 +0900
comments: true
categories: 
---

In this article, I exaplain how to use Norikra to count HTTP status with Fluentd. 
Previously, it was common to use [fluent-plugin-datacounter](https://github.com/tagomoris/fluent-plugin-datacounter) for the purpose, but, 
today I will explain how to achieve the same feature with [Norikra](http://norikra.github.io/). 

REMARK: This article is a short translated version of Japanese article written at [http://blog.livedoor.jp/sonots/archives/37921050.html](http://blog.livedoor.jp/sonots/archives/37921050.html)

## What Not Covered

In this article, I will not cover, how to install Norikra, how to install Fluentd.

## What is Norikra, and Why to Use Norikra

Norikra is a Schema-less Stream Processor based on Esper, which is a kind of CEP (Complex Event Processing) engine. 
With Norikra, we can use highly-functional SQL-like query for processing streaming data. 

We can achieve similar things by using plugins like fluent-plugin-datacounter.
However, a Fluentd process can utilize only one CPU core because Fluentd uses CRuby,
in contrast, a Norikra process can utilize multiple CPU cores because Norikra uses JVM (Esper is wirtten with Java, and Norikra is written with JRuby).
This is the 1st reason to use Norikra. This feature is attractive especially when we need to processs heavy log data where one CPU core is not sufficient to process it in real-time.

The 2nd reason to use Norikra is because it does not require to restart for adding or removing queries. 
Fluentd requires us to restart its processes when we change their configuration files to add or remove new settings. 

## Data Counting using fluent-plugin-datacounter

Let me describe how to perform data counting using fluent-pugin-datacounter first. We will replace this to Norikra later.

### Requirement

Assume there is a demand as followings:

1. Want to count HTTP status codes in a specified time interval by seeing status fields written in logs.
2. Want to count the status code for each host.
3. Want to count the status code as an aggregation of all hosts. 

### Specification of Input Data (Log)

Assume that log data are sent from Fluentd agent with tags like "visualizer.*logname*.*hostname*" where *logname* stands for an arbitrary string which operation engineers name to distinguish logs. 

Also, assume that messages contain `time`, `status`, `reqtime`, `method`, `uri` fields. 

Example) visualizer.api_restful.host001

```
{"status":"403","reqtime":"0.123","method":"GET","uri":"http://example.com/v1/restful/people?id=1"}
```

### Configuration

The status code counting can be achieved with the following configuration using fluent-plugin-datacounter. 

```apache
<source>
  type forward
  port 24224
</source>
<match visualizer.api_restful.*>
  type copy
  # for each host
  <store>
    type record_reformer
    enable_ruby false
    tag ${tag_prefix[-2]}.hosts.${tag_parts[-1]}
  </store>
  # for all
  <store>
    type record_reformer
    enable_ruby false
    tag ${tag_prefix[-2]}.all.all
  </store>
</match>
<match visualizer.api_restful.{hosts,all}.*>
  type datacounter
  count_interval 60
  aggregate tag
  tag_prefix status_count
  count_key status
  pattern1 2xx ^2\d\d$
  pattern2 3xx ^3\d\d$
  pattern3 400 ^400$
  pattern4 4xx ^4\d\d$
  pattern5 503 ^503$
  pattern6 5xx ^5\d\d$
  output_per_tag yes
</match>
```

The output messages become as followings (unnecessary fields are ommitted):

```js
status_count.visualizer.api_restful.hosts.host001:
{"2xx_count":5,"3xx_count":0,"400_count":0,"4xx_count":0,"503_count":0,"5xx_count":0}
```

Let us think of replacing this with Norikra.

## Data Counting using Norikra

Referring the [document](http://norikra.github.io/) of Norikra, and README of [fluent-plugin-norikra](https://github.com/norikra/fluent-plugin-norikra), 
we will try to replace fluent-plugin-datacounter with out_norikra, Norikra Query (accurately, EPL of Esper), and in_norikra. 

**in_norikra**

Configure Fluentd to receive data, and transfer the data to Norikra. 

This configuration sets the `target` of Norikra (which is like the `table` of RDBMS) to be `logname` of "visualizer.*logname*.*hostname*" tag. 
Also, this sets `hostname` to the `host` field of messages by extracting it from tags. 

```apache
<source>
  type forward
  port 24224
</source>

<match visualizer.api_restful.**>
  type record_reformer
  enable_ruby false
  tag reformed.${tag_prefix[-2]} # Remove the hostname from the tag
  <record>
    host ${tag_parts[-1]} # Set the hostname to `host` field to use in GROPU BY statement.
  </record>
</match>

<match reformed.visualizer.api_restful>
  type norikra
  norikra localhost:26571
  remove_tag_prefix reformed.visualizer
  target_map_tag true # target will be `api_restful`
  <default>
    auto_field false  # norikra includes fields only used in queries.
  </default>
</match>
```

**Norikra Query**

Surprisingly, **we can write Java code** in a Norikra query. So, we can realize same conditions with the case of fluent-plugin-datacounter by using `String#matches`. 
Also, we can take couting for each host using `GROUP BY` statement. Cool. 

EDIT: I replaced `COUNT(1, status.matches('^2\d\d$'))` to `COUNT(1, status REGEXP '^2\d\d$')` because the master [tagomoris](https://twitter.com/tagomoris), the author of Norikra, said the latter is better in performance. Thinking of performance, replacing REGEXP to LIKE, or converting status field to interger and using `COUNT(1, status / 100 = 2)` would achieve better performances. 

```bash
# creation of a target
norikra-client target open api_restful

# all. `AS 2xx_count` was NG, so I made `As count_2xx`.
norikra-client query add status_count.all.api_restful "$(cat <<EOF
SELECT \
COUNT(1, status REGEXP '^2\d\d$') AS count_2xx, \
COUNT(1, status REGEXP '^3\d\d$') AS count_3xx, \
COUNT(1, status REGEXP '^400$') AS count_400, \
COUNT(1, status REGEXP '^4(?!00)\d\d$') AS count_4xx, \
COUNT(1, status REGEXP '^503$') AS count_503, \
COUNT(1, status REGEXP '^5(?!03)\d\d$') AS count_5xx \
FROM api_restful.win:time_batch(60 sec)
EOF
)"
# each host
norikra-client query add status_count.host.api_restful "$(cat <<EOF
SELECT \
host, \
COUNT(1, status REGEXP '^2\d\d$') AS count_2xx, \
COUNT(1, status REGEXP '^3\d\d$') AS count_3xx, \
COUNT(1, status REGEXP '^400$') AS count_400, \
COUNT(1, status REGEXP '^4(?!00)\d\d$') AS count_4xx, \
COUNT(1, status REGEXP '^503$') AS count_503, \
COUNT(1, status REGEXP '^5(?!03)\d\d$') AS count_5xx \
FROM api_restful.win:time_batch(60 sec) \
GROUP BY host
EOF
)"
```

Notice that the query can be also constructed as followings:

```bash
# 2xx
$ norikra-client query add status_count.all.2xx.api_restful \
  "SELECT COUNT(status) AS count_2xx \
  FROM api_restful.win:time_batch(60 sec) \
  WHERE status REGEXP '^2\d\d$'"
# 3xx
$ norikra-client query add status_count.all.3xx.api_restful \
  "SELECT COUNT(status) AS count_3xx \
  FROM api_restful.win:time_batch(60 sec) \
  WHERE status REGEXP '^3\d\d$'"
# 400 
# 4xx
# 503
# 5xx
```

but, I felt preparing many queries make difficult to manage them. So, I prefered the former `COUNT(1, status REGEXP '^2\d\d$')` way.

**in_norikra**

We configure Fluentd to retrieve results of Norikra in each 60 seconds. 
Here, we set `tag query_name` so that the tag of retrieved messages to be the query name which we specified on `query add` commands. 

```apache
<source>
  type norikra
  norikra localhost:26571
  <fetch>
    method event
    target status_count.all.api_restful # query name
    tag query_name
    # tag string api_restful.status_count.all
    # tag field field_name
    interval 60s
  </fetch>
  <fetch>
    method event
    target status_count.host.api_restful # query name
    tag query_name
    interval 60s
  </fetch>
</source>
```

Done!

The output wil be as belows:

```js
status_count.host.api_restful:
{"host":"host001","count_2xx":5,"count_3xx":0,"count_400":0,"count_4xx":0,"count_503":0,"count_5xx":0}
```

## Conclusion

I explained how to replace fluent-plugin-datacounter with Norikra. Try it out!


