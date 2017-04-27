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

Tables in influxdb are called as measurement. A single measurement can belong to different retention policies. A retention policy describes how long InfluxDB keeps data (DURATION) and how many copies of those data are stored in the cluster (REPLICATION). InfluxDB automatically creates the autogen retention policy which has an infinite duration and replication factor is set to 1

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
![](https://raw.githubusercontent.com/AmbujJauhari/AmbujJauhari.github.io/master/_images/influxdb.JPG)

Double click influxdb.exe to start the influxdb server by default it run on port 8086, but it can be configured using file influxdb.conf. Click on influx.exe to start a CLI interface. Now lets create a database first.

![](https://raw.githubusercontent.com/AmbujJauhari/AmbujJauhari.github.io/master/_images/creating_db.JPG)

We have created a database SampleAppMetric and we will be using this database

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

    private static final int THRESHOLD_LIMIT = 50;
    InfluxDB influxDB;

    String dbName;
    BatchPoints batchPoints;

    public InfluxDBGaugeWriter() {
        influxDB = InfluxDBFactory.connect("http://localhost:8086", "root", "root");
        dbName = "sampleAppMetric";
        batchPoints = BatchPoints
                .database(dbName)
                .tag("componentName", "SampleApp")
                .retentionPolicy("autogen")
                .consistency(InfluxDB.ConsistencyLevel.ALL)
                .build();
    }

    @Override
    public void set(Metric<?> metric) {
        Point point = Point.measurement(MetricNameMapping.valueOf(metric.getName())).time(System.currentTimeMillis(), TimeUnit.MILLISECONDS)
                .addField("metricvalue", metric.getValue()).build();


        batchPoints.point(point);

        if (batchPoints.getPoints().size() > THRESHOLD_LIMIT) {
            influxDB.write(batchPoints);
            batchPoints.getPoints().clear();
        }
    }
}
```

spring boot provides MetricWriter interface or GaugeWriter for simple use cases. We have used GaugeWriter and overriden the set method.
In the constructor we are just establishing the connection with InfluxDB and created a batchPoints, in the set method we are creating a point and adding to the batchpoints and writing to influxdb when the batch breaches the threshold limit. 

If you see the set method here, you can see that we have not defined how our measurement or table will look like i.e we have not defined any particular schema, it is created dynamically and hence influxdb offers a schema less protocol.

It seems that grafana is not able to read measurements if it has '.' in its name so i have created a class MetricNameMapping whose valueOf function will replace the '.' from the metric's name.


Please note that we can do all other sorts of optimization here which involves batching and buffered flushing of data. This is just for learning purpose.

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

This is all good, now there are numerous situations where we would need to have custom metrics i.e. our own metrics information. So lets try creating a custom metric

```
@Service
public class ScheduledMetricsExample {

    private final CounterService counterService;

    public ScheduledMetricsExample(CounterService counterService){
        this.counterService = counterService;
    }

    @Scheduled(fixedRate = 60000L)
    public void scheduledMethod() {
        this.counterService.increment("custom.metrics.value");
    }

}
```

So we have created a service class ScheduledMetricsExample and declared a field of type CounterService which is instantiated by spring using the constructor. Spring provides 2 classes CounterService and GaugeService. The CounterService exposes increment, decrement and reset methods; the GaugeService provides a submit method.

We are using a counterservice here for incrementing custom.metrics.value every 1min.


Now lets create the application.properties file in resources which wil define the portno. for our application and the management port for our application.

```
server.port=8090
management.port=8081
management.security.enabled=false
service.name=Ambuj
```

You see we have defined service.name which will be injected the HelloWorldProperties file name field. we defined server.port as 8090 which means our application will be running on port 8090 and management.port is 8081 i.e. we will be able to see application metrics on port 8081

Now lets deploy your spring boot application, like any other application. Since i have the project setup in IDE, i will just run my main class. Once your application is up 

Lets browse to the application url first [http://localhost:8090]
![](https://raw.githubusercontent.com/AmbujJauhari/AmbujJauhari.github.io/master/_images/applicationurl.JPG)


Now once you can see the application lets go and check the management port url [http://localhost:8081/metrics]
![](https://raw.githubusercontent.com/AmbujJauhari/AmbujJauhari.github.io/master/_images/metricurl.JPG)

[http://localhost:8090]:http://localhost:8090
[http://localhost:8081/metrics]:http://localhost:8081/metrics

Now once we have confirmed that our application is working and our metrics can be seen, lets take a look at the influxdb 


Lets first take a look at all the measurements created
![](https://raw.githubusercontent.com/AmbujJauhari/AmbujJauhari.github.io/master/_images/measurements.JPG)

you can see that we have all the measurements and each measurement is nothing just the name of each metric

Now lets view some of the measurements like uptime and heap
![](https://raw.githubusercontent.com/AmbujJauhari/AmbujJauhari.github.io/master/_images/selectquery.JPG)

Since there were too many records, i have limited the no. of rows to 5.

So significance of the above data can be interpreted as at a particular given time what is the heap size of SampleApp.

All this is good and fine, we have the application metrics data stored in a db but now lets see how we can visualize the data in grafana

### Setting up Grafana ###

We are using the grafana version 4.1.2
once you have downloaded grafana unzip it and there is a file defaults.ini and sample.ini. Never ever change the defaults.ini to customize copy the sample.ini to custom.ini and override all the defaults in custom.ini file. For example we have modified the grafana running port from 3000 to 8080

![](https://raw.githubusercontent.com/AmbujJauhari/AmbujJauhari.github.io/master/_images/grafanaserverport.JPG)

let bring up grafan using grafan-server.exe and lets access [http://localhost:8080]

[http://localhost:8080]:http://localhost:8080
![](https://raw.githubusercontent.com/AmbujJauhari/AmbujJauhari.github.io/master/_images/grafanahomepage.JPG)

admin use for grafana is admin and password is admin as well, So lets login now

![](https://raw.githubusercontent.com/AmbujJauhari/AmbujJauhari.github.io/master/_images/grafanahomedashboard.JPG)

Now before we can do anything, we need to setup influxdb datasource first

![](https://raw.githubusercontent.com/AmbujJauhari/AmbujJauhari.github.io/master/_images/grafanahomedashboarddatasource.JPG)

![](https://raw.githubusercontent.com/AmbujJauhari/AmbujJauhari.github.io/master/_images/grafanadatasource.JPG)


Click on Add Data Source to add a new data source

![](https://raw.githubusercontent.com/AmbujJauhari/AmbujJauhari.github.io/master/_images/grafanaadddatasource.JPG)

Lets fill in the details now

![](https://raw.githubusercontent.com/AmbujJauhari/AmbujJauhari.github.io/master/_images/grafandatasourcedetails.JPG)

Once that is done you can click Add and then save & Test to test the connection, it should be all success

Now lets create a new dashboard

![](https://raw.githubusercontent.com/AmbujJauhari/AmbujJauhari.github.io/master/_images/grafananewdashboard.JPG)


Grafana allows you to create multiple rows, and each row can have multiple panels

We will be displaying heapused, uptime and our custom metrics that we have created custom.metrics.value 

we will disply heap and uptime in one row in separate panels and custome metrics in a separate row

So lets start with heap in panel-1, we will be choosing graph as our visualizer for heap

Once you click on graph it will create a blank panel, we will edit this panel now.

![](https://raw.githubusercontent.com/AmbujJauhari/AmbujJauhari.github.io/master/_images/grafanapaneledit.JPG)

 
Let modify the panel settings by changing the datasource and naming the fields and measurements

![](https://raw.githubusercontent.com/AmbujJauhari/AmbujJauhari.github.io/master/_images/grafanaheapusedpanelJPG.JPG)

Click on save button on the top now

![](https://raw.githubusercontent.com/AmbujJauhari/AmbujJauhari.github.io/master/_images/SampleAppDashboard.JPG)


Similarly lets add the panel for uptime in the same row, for that we will have to reduce the size of heapused panel 

![](https://raw.githubusercontent.com/AmbujJauhari/AmbujJauhari.github.io/master/_images/reduceheapusedpanel.JPG)

Now click on left side corner a popup will open there click on add panel
![](https://raw.githubusercontent.com/AmbujJauhari/AmbujJauhari.github.io/master/_images/grafanaaddnewpanel.JPG)

Now similar to what we have done for heapused, we do it for uptime and save the dashboard. Similarly we will add a new row and inside add a new panel and display our custom metrics but  additionally we will choose bars to display this time

![](https://raw.githubusercontent.com/AmbujJauhari/AmbujJauhari.github.io/master/_images/grafananewrownewpanel.JPG)

![](https://raw.githubusercontent.com/AmbujJauhari/AmbujJauhari.github.io/master/_images/grafanadisplaybar.JPG)


Now lets see how our complete dashboard looks like

![](https://raw.githubusercontent.com/AmbujJauhari/AmbujJauhari.github.io/master/_images/CompleteDashboard.JP)


This is a just a small post on how we can integrate spring-boot, influxdb and grafana for new real time monitoring. Hope this blog has been useful. You can find the source code at [git repo]

[git repo]:https://github.com/AmbujJauhari/spring-boot-influx-grafana
