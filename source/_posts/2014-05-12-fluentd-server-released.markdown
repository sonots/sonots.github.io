---
layout: post
title: "fluentd-server released!"
date: 2014-05-12 19:52:23 +0900
comments: true
categories: 
---

[fluentd-server](https://github.com/sonots/fluentd-server), a Fluentd config distribution server, is released!

## What is Fluentd Server

I heard opinions as some users of Fluentd want something like chef-server for Fluentd, so I created the fluentd-server. 

With Fluentd Server, you can manage fluentd configuration files centrally with erb.

For example, you may create a config post whose name is worker as:

```apache
<source>
  type forward
  port <%= port %>
</source>

<match **>
  type stdout
</match>
```

Then you can download the config via an API whose uri is like `/api/worker?port=24224` where its query parameters are replaced with variables in the erb. The downloaded contents should become as follows:

```apache
<source>
  type forward
  port 24224
</source>

<match **>
  type stdout
</match>
```

## How To Use From Fluentd

The include directive of fluentd config supports http, so write just one line on your fluentd.conf as:

```
# /etc/fluentd.conf
include http://fqdn.to.fluentd-server/api/:name?port=24224
```

where :name is the name of your config post, so that it will download the real configuration from the Fluentd Server.

You can see the specification detail of API at [API.md](https://github.com/sonots/fluentd-server/blob/master/API.md).

## Demo

I deployed on heroku, so you can see the demo at [http://fluentd-server.herokuapp.com](http://fluentd-server.herokuapp.com).

## All the end

Fluentd server is yet under development. Patches are welcome!

