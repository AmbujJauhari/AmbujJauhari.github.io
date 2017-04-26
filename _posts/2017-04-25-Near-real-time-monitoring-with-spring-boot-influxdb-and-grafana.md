---
layout: post
title: "Near real time monitoring with spring boot, influxdb and grafana"
date: 2017-04-25
---

## Introduction ##
The blog post will cover some basic aspects on real time monitoring of spring applications. We are going to use spring boot actuators to analyze different types of information about the running application – health, metrics, info, dump, env etc by dumping the data in a influx DB, which is basically a TSDB (time series database). The data dump can be then visualized in Grafana.

## Quick note on spring boot actuator ##
 Actuators provide different types of information about the application, including health, metrics, info, dump, status etc by exposing them via endpoints.

Here are some of the commonly used endpoints
* /health – Shows application health information
* /info – Displays arbitrary application info.
* /metrics – Shows ‘metrics’ information for the current application.

These are just some endpoints, all the endpoints are listed on the [official website]

We are not going to cover spring boot actuators in detail, there are numerous tutorials available and the official guide is very good as well.


## Quick note on InfluxDB ##
InfluxDB is an open-source time series database. It is optimized for fast, high-availability storage and retrieval of time series data in fields such as operations monitoring, application metrics, Internet of Things sensor data, and real-time analytics.

Here is some sample data in influxDB and i will define some common terminologies

name: Feeder

|time                      |free.mem     |counter.service| mem |
|--------------------------|-------------|---------------|-----|
|2015-08-18T00:00:00Z      |1000         |1              |1000 |    
|2015-08-18T00:00:00Z      |2000         |1              |15000|    
|2015-08-18T00:06:00Z      |1100         |1              |20000|     
|2015-08-18T00:06:00Z      |3323         |1              |20055|     
|2015-08-18T05:54:00Z      |2435         |1              |23423|    
|2015-08-18T06:00:00Z      |1245         |2              |45354|    
|2015-08-18T06:06:00Z      |8224         |2              |23564|    
|2015-08-18T06:12:00Z      |7562         |2              |96752|   

Above table shows the data from a sample table in influxdb. Every table in the influxDB has a time column in it. The next 3 columns are the fields, fields are made up of key-value pair. values are the data which can be of any format can be strings, floats, integers, or booleans, and, because InfluxDB is a time series database, a field value is always associated with a timestamp

Fields are not indexed so any queries done with fields will scan through the table. Indexed columns in influxdb are called tags, they are not compulsary but should be used for querying.

Tables in infulxdb are called as measurement. A single measurement can belong to different retention policies. A retention policy describes how long InfluxDB keeps data (DURATION) and how many copies of those data are stored in the cluster (REPLICATION). InfluxDB automatically creates the autogen retention policy which has an infinite duration and replication factor is set to 1

Further details can be found influx DB's [official documents]

[official website]:http://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#production-ready-endpoints
[official documents]:https://docs.influxdata.com/influxdb/v1.2/


