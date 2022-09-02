---
title: "Create dashboards in Azure Data Explorer using azure IoT Hub data"
date: 2022-09-01
categories:
- Arduino
- azure
author: Hugo Alves
resources:
- name: "featured-image"
  src: "images/banner.jpg"
- name: featured-image-preview
  src: images/banner.jpg
subtitle: Build a room temperature monitor using Azure and Arduino Nano - part 2
tags:
- Arduino
- azure
- iot
---

On the last [tutorial](https://www.hugomalves.com/posts/2022/08/upload-sensors-data-from-arduino-nano-rp2040-to-azure-iot-hub/) we configured an Arduino to measure the room temperature, and upload the measurements to Azure IoT Hub. In this tutorial we are going to create a Azure stream analytics job to get the raw data from IoT Hub and transform it. Then, the transformed data will be sent to Azure Data Explorer where it will be permanently stored in a database. With all the processed data stored in the data explorer database we can start creating dashboards from it.

Let's see again the diagram of the complete flow.
{{< image src="images/flow_diagram.jpg" alt="upload temperature sensor data to azure" height="auto"  >}}

We have already implemented the first 3 steps, now we just need to read and transform the data according with our needs.

## IoT Hub configurations

Before continuing we should create a consumer group in the IoT Hub. This consumer group is going to be used later by the stream analytics job to read the events.
{{< image src="images/iothub_consumer_group.jpg" alt="IoT Hub consumer group" height="auto"  caption="IoT Hub consumer group">}}

## Azure Data Explorer cluster

{{< admonition type=tip title="" >}}
Create all the resources under the same resource group. It will be easier to delete everything later.
{{< /admonition >}}

### Create the cluster

Before creating and configuring the Stream analytics job we must create the data explorer cluster. To create it you can follow the following steps:
{{< admonition type=tip title="This is a tip" >}}
Don't forget to choose the developer tier. The data explorer cluster cost can be quite high!
{{< /admonition >}}

1. Navigate to the Data explorer cluster page
{{< image src="images/data_explorer_cluster_create_1.jpg" alt="Data explorer cluster create front page" height="auto"  caption="Data explorer cluster page">}}


2. Create the cluster with the following configurations
{{< image src="images/data_explorer_cluster_create_2.jpg" alt="Data explorer cluster create front page" height="auto"  caption="Data explorer cluster page">}}
Note that the cluster creation can take some time, in my case it took around 15 minutes.

### Create the database to store the telemetry data

To store the telemetry data we have to create a database in the cluster.

- Open the databases tab inside your cluster configuration page
{{< image src="images/data_explorer_database_1.jpg" alt="Data explorer Create database tab" height="auto"  caption="Data explorer create database tab">}}

- Create the database with the following configs
{{< image src="images/data_explorer_database_2.jpg" alt="Data explorer Create database" height="auto"  caption="Data explorer create database">}}


### Create the database tables and schema

Let's now create the table and schema that will store the telemetry data exported from IoT Hub. First navigate to the recently created database page, and then follow the following steps:

- Right click on the database name and then on the "create table". You will be redirect to the azure data explorer page
{{< image src="images/database_schema_1.jpg" alt="Azure data explorer" height="auto"  caption="Navigate to azure data explorer">}}

- Create a new table
{{< image src="images/database_schema_2.jpg" alt="Azure data explorer new table" height="auto"  caption="Create a new table">}}

- Define the source for the table - none
{{< image src="images/database_schema_3.jpg" alt="Azure data explorer new table source" height="auto"  caption="Specify source none">}}

- Create the columns  
{{< image src="images/database_schema_4.jpg" alt="Azure data explorer new table columns" height="auto"  caption="Create the columns to store the telemetry data">}}
Instead of manually typing each column name and type you can use the following command:
```sql
.create table ['telemetry']  (['temperature']:real, ['deviceId']:string, ['EventEnqueuedUtcTime']:datetime)
```
\
And that's all, now we have a table ready to store the telemetry data. The next step is to populate this table with the data from IoT Hub.

## Stream Analytics Job

The stream analytics job is going to be responsible to get the raw data from IoT Hub, transform it and then send it to azure data explorer database to be persisted.

### Create the Job

- Navigate to the Stream analytics Job page and click create
{{< image src="images/stream_analytics_job_1.jpg" alt="Stream analytics job front page" height="auto"  caption="Stream analytics job page">}}

- Create a job with the following configuration
{{< image src="images/stream_analytics_job_2.jpg" alt="Stream analytics job configuration" height="auto"  caption="Stream analytics job configuration">}}


### Configure the job inputs

Now we are going to define the inputs of this job. As said before, the input of this job is the data that is arriving from the IoT Hub, we have to configure an input specifying the origin and the format of the data.

- Navigate to the Inputs tab on the Stream analytics Job page  
Select the "IoT Hub" type
{{< image src="images/stream_analytics_job_input_1.jpg" alt="Stream analytics job input" height="auto"  caption="Stream analytics job input menu">}}

- Create an input job with the following configurations
{{< image src="images/stream_analytics_job_input_2.jpg" alt="Input job configurations" height="auto"  caption="Input job configurations">}}


### Configure the job outputs

In this step we are going to create the output for the data. The transformed data from the input will be sent to whichever system we configure in the output, in this case we are going to use azure data explorer, but we could also export the data to Cosmos db for example.

- Navigate to the Outputs tab on the Stream analytics Job page  
Select the "Azure data explorer" type
{{< image src="images/stream_analytics_job_output_1.jpg" alt="Stream analytics job output" height="auto"  caption="Stream analytics job output menu">}}

- Create an output job with the following configurations
{{< image src="images/stream_analytics_job_output_2.jpg" alt="output job configurations" height="auto"  caption="Output job configurations">}}


### Create a query to connect the input to the output

Having defined the input and output data is not enough to complete the flow from IoT Hub to data explorer. As we saw before we still have to do some transformations to the data before being ready to be used by data explorer. This is the job of que query that we will create now.

On the Stream analytics job page navigate to the query tab an paste the following query:

{{< image src="images/stream_analytics_query.jpg" alt="Stream analytics query" height="auto"  caption="Stream analytics query">}}


```sql
SELECT
    temperature,
    deviceId,
    EventEnqueuedUtcTime
INTO
    [dataExplorerOutput]
FROM
    [iotHubInput]
```

This query will read data from the input(IoTHub), and write it to the output (data explorer database). Since we don't need all the data to create the dashboards, we will only select the temperature, deviceId and EventEnqueuedUtcTime values. All the other values sent to Azure Iot Hub will not be propagated to the data explorer database.

After clicking **save query** you can test the query before starting the job. If you click on "Input preview" you can see the raw data that is arriving to the stream job from the IoT Hub. If you don't see any data just click on "refresh", it can take some seconds until you see some records appearing. 
The records shown on this tab are the input of the previous query.

{{< image src="images/stream_analytics_query_input_preview.jpg" alt="Stream analytics input preview" height="auto"  caption="Stream analytics input preview">}}


On the "Test results" tab you can see the result of the query. As expected, you will only see three columns, the temperature, the deviceId and the EventEnqueuedUtcTime.
This is the data that will be stored in the data explorer database.
{{< image src="images/stream_analytics_query_test_results.jpg" alt="Stream analytics test results" height="auto"  caption="Stream analytics test results">}}


### Start the stream analytics job

The last step is to start the stream analytics job, all of the configurations we did can't be done while the job is running.
While the job is stopped nothing is being sent to data explorer. To start the job you should open the overview page and click start.

{{< image src="images/stream_analytics_job_start.jpg" alt="Stream analytics job start" height="auto"  caption="Stream analytics job start">}}


## Azure data explorer dashboards

If all of the previous configurations were done correctly we now have everything we need to start creating the dashboards. The temperature measurements should be flowing from the arduino to IoT Hub, then to stream analytics job and finally to the data explorer database.

Let's now explore the Azure data explorer application. You can find the azure explore URL in the data explorer cluster overview page. You should open it on a new tab.
{{< image src="images/data_explorer_url.jpg" alt="Data explorer Url" height="auto"  caption="Data explorer Url">}}


### Verify that the stream analytics job is storing the transformed data in the telemetry table

On the azure data explorer application you should already see the database and table you created before. This table will contain the filtered data (temperature, deviceId and time).  
To validate the data you can run the following KQL query:

```sql
telemetry
| take 100 | order by EventEnqueuedUtcTime  desc 
```
{{< image src="images/data_explorer_transformed_data.jpg" alt="Data stored in the data explorer database" height="auto"  caption="Data stored in the data explorer database">}}

**Note**, before running the KQL query you should select the "telemetry table"
{{< admonition type=tip title="" >}}
There can be a delay of 5 minutes from the moment you start the stream analytics job until you start seeing data on the data explorer. If you don't see any data on the table please wait some minutes.
{{< /admonition >}}

You can also check the analytics job metrics to see events the number of events that are being processed.
On the next image, we see on the events count chart that the stream events job received 73 events and sent 73 to the data explorer database in the last 30 minutes.
{{< image src="images/analytics_jobs_metrics.jpg" alt="Analytics Job metrics" height="auto"  caption="Analytics Job metrics">}}



### Create a dashboard

Let's now create a simple dashboard that shows us the evolution of the temperature over the time.
I believe that the steps required to create the dashboards are quite simple and intuitive. However there are a lot of steps for creating a screenshot for each, you can get the big picture by watching the following movie.

{{< youtube 52qta7OZPKU >}}

The  most "technical" part is the one to create the datasource. Here I just copied the URI from the data explorer cluster, then the data explorer can automatically fetch the existing databases for you to use.

And I used the following KQL query to populate the dashboard:

```sql
telemetry | where EventEnqueuedUtcTime  between (['_startTime'] .. ['_endTime'])
```

**Note** This is not the exact same query you see on the video, the filter for the EventEnqueuedUtcTime wasn't present. However you should add the filter to only fetch the strictly necessary data for the plot

{{< image src="images/result.jpg" alt="Final result" height="auto"  caption="Final result">}}

## Clean the resources

{{< admonition type=warning title="Cleaning" >}}
Don't forget to delete all the resources used. The data explorer cluster cost can be quite high!
{{< /admonition >}}

If you followed the instructions of this guide you should have created all of your resources on a specific resource group. To delete everything you can just navigate to the resource group and delete everything.

{{< image src="images/resources_clean.jpg" alt="Delete all the resources" height="auto"  caption="Delete all the resources">}}

## References

- [Azure database explorer][1]
- [Azure data explorer access control][2]
- [Data explorer product page][3]
- [Azure Stream analytics job intro][4]
- [What is Azure Data Explorer?][5]
- [Kusto Query Language (KQL) overview][6]

---
[1]: https://docs.microsoft.com/en-us/azure/stream-analytics/azure-database-explorer-output
[2]: https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/access-control/how-to-authenticate-with-aad
[3]: https://azure.microsoft.com/en-us/pricing/details/data-explorer/
[4]: https://docs.microsoft.com/en-us/azure/stream-analytics/stream-analytics-introduction
[5]: https://docs.microsoft.com/en-us/azure/data-explorer/data-explorer-overview
[6]: https://docs.microsoft.com/en-us/azure/data-explorer/kusto/query/

