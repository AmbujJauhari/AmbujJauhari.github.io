---
layout: post
title: "Near real time monitoring with spring boot, influxdb and grafana"
date: 2017-04-25
---

The blog post will cover some basic aspects on real time monitoring of spring applications. We are going to use spring boot actuators to analyze different types of information about the running application – health, metrics, info, dump, env etc by dumping the data in a influx DB, which is basically a TSDB (time series database). The data dump can be then visualized in Grafana.


