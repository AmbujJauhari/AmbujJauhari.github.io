---
layout: post
title: "Near real time monitoring with spring boot, influxdb and grafana {Under DEV}"
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


## Quick note on Grafana ##
Grafana is an open source metric analytics & visualization suite. It is most commonly used for visualizing time series data for infrastructure and application analytics

More on grafana [here]

[official website]:http://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#production-ready-endpoints
[official documents]:https://docs.influxdata.com/influxdb/v1.2/
[here]:http://docs.grafana.org/

## Project setup ##

### Setting up influx db ###
Lets first bring up influxdb on my local machine. We are using version v1.2.2
![](../_images/influxdb.JPG?raw=true)

Double click influxdb.exe to start the influxdb server by default it run on port 8086, but it can be configured using file influxdb.conf. Click on influx.exe to start a CLI interface. Now lets create a database first.

![](../_images/creating_db.JPG?raw=true)

We have created a database SampleAppMetric and we will be using thie database

### Spring boot application setup ###

We will setup a small spring boot application with a simple HelloWorld controller. 
First lets create a simple configuration properties file

```
@ConfigurationProperties(prefix = "service", ignoreUnknownFields = false)
public class HelloWorldProperties {


    private String name = "World";

    public String getName() {
        return this.name;
    }

    public void setName(String name) {
        this.name = name;
    }


}
```

@ConfigurationProperties is a very neat way to inject properties in an application. All you need to do is add a simple property in application.properties service.name and instead of 'World', your value from application.properties will be injected.

Now lets create a controller

```
@Controller
@Description("A controller for handling requests for hello messages")
public class HelloController {

    private final HelloWorldProperties helloWorldProperties;

    public HelloController(HelloWorldProperties helloWorldProperties) {
        this.helloWorldProperties = helloWorldProperties;
    }

    @GetMapping("/")
    @ResponseBody
    public Map<String, String> hello() {
        return Collections.singletonMap("message",
                "Hello " + this.helloWorldProperties.getName());
    }
}
```

It a simple controller, which defaults to "Hello world".


Next we will create InfluxDBWriter which will actually responsible for putting data in InfluxDB

```
public class InfluxDBGaugeWriter implements GaugeWriter {

    InfluxDB influxDB;

    String dbName;
    BatchPoints batchPoints;

    public InfluxDBGaugeWriter() {
        influxDB = InfluxDBFactory.connect("http://localhost:8086", "root", "root");
        dbName = "sampleAppMetric";
        batchPoints = BatchPoints
                .database(dbName)
                .tag("async", "true")
                .retentionPolicy("autogen")
                .consistency(InfluxDB.ConsistencyLevel.ALL)
                .build();
    }

    @Override
    public void set(Metric<?> metric) {

        Point point = Point.measurement("FeederMetric").time(System.currentTimeMillis(), TimeUnit.MILLISECONDS)
                .addField(metric.getName(), metric.getValue()).build();


        batchPoints.point(point);
        influxDB.write(batchPoints);
    }
}
```

spring boot provides MetricWriter interface or GaugeWriter for simple use cases. We have used GaugeWriter and overriden the set method.
In the constructor we are just establishing the connection with InfluxDB and created a batchPoints, in the set method we are creating a point and adding to the batchpoints and writing to influxdb. 

Please note that we can do all sorts of optimization here which involves batching and buffered flushing of data. This is just for learning purpose.

Now we will create the application's main class

```
@SpringBootApplication
@EnableConfigurationProperties(HelloWorldProperties.class)
public class Application {

    @Bean
    @ExportMetricWriter
    public GaugeWriter fileGaugeWriter() {
        InfluxDBGaugeWriter writer = new InfluxDBGaugeWriter();
        return writer;
    }

    @Bean
    public MetricsEndpointMetricReader metricsEndpointMetricReader(MetricsEndpoint metricsEndpoint) {
        return new MetricsEndpointMetricReader(metricsEndpoint);
    }

    public static void main(String... args) {
        SpringApplication.run(Application.class, args);
    }
}
```
