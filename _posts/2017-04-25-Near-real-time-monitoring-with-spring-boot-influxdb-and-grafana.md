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
* /health – Shows application health information (a simple ‘status’ when accessed over an unauthenticated connection or full message details when authenticated). It is not sensitive by default.
* /info – Displays arbitrary application info. Not sensitive by default.
* /metrics – Shows ‘metrics’ information for the current application. It is also sensitive by default.
* /trace – Displays trace information (by default the last few HTTP requests).

These are just some endpoints, all the endpoints are listed on the [official website]

[official website]:http://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#production-ready-endpoints


