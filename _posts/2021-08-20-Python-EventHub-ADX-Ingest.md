---
layout: post
title: Automatically ingest Event Hub Data to Azure Data Explorer using Python
subtitle: learned the hard way!
thumbnail-img: /assets/img/20210820/thumb.png
cover-img: /assets/img/20210820/banner.png
share-img: /assets/img/20210820/banner.png
tags: [Python, ADX, Azure Event Hub, ingest, pipeline, Azure]
comments: true
---

Recently I came across a feature which allows customers to ingest data automatically into azure data explorer (adx) by using Azure Event Hub.
As this feature is to the time of this blog post still relatively new, I wanted to find out what benefits customers get by using it as well as how it works. The subtitle of this post mentions *learned the hard way!* and you might ask yourself, what does this mean? Actually during my testing and implementation I have found a bug which PG first needed to fix! I thought it was on me why it didn´t work the way I thought it would, but "luckily" it wasn´t my bad. Even though, bug is fixed and you can start testing the same way I did. I have used python to send batches of simulated IoT Sensor Data to Event Hub. This Data I would like to use in ADX to be automatically ingested.

# Design

In this test scenario I have used the following components:
* Python script to send batches of "raw" telemetry data (producer client)
* Python script to check transmitted values to the event hub (consumer client)
* Event Hub Namespace & Event Hub Instance (to send and read data from)
* a storage account to store checkpoint data from the consumer client (optional just for testing the consumer client)
* preperation of the Azure Data Explorer to ingest Data
* configuration of a ingestionbatching policy

# Event Hub data transmission

## send data to event hub (producer client)

First, I decided to send a batch of raw sensor data to the event hub. For this particular case I decided to use python.


```python
async def create_batch(producer):
    event_data_batch = await producer.create_batch()
    i = 0
    while i <= 1000:
        device_id = random.randint(0,2)
        json_obj = {
            "timestamp": str(datetime.utcnow()),
            "temperature": round(random.uniform(16.8,37.5),3),
            "humidity": round(random.uniform(40.0,62.4),2),
            "iotdevice": "device"+str(device_id)
            }
        string = json.dumps(json_obj)
        Event_data = EventData(body=string)
        Event_data.properties = {
            "Table":"TestTable",
            "IngestionMappingReference":"TestMapping", 
            "Format":"json"
            }
        print(Event_data)
        event_data_batch.add(Event_data)
        i += 1
    print(event_data_batch)
    return event_data_batch
```
The whole script is published in my offical github account!

Event Properties in general are used for passing metadata associated with the event body during Event Hubs operations. Furthermore, we are using those Properties later for ingesting Data into the Azure Data Explorer accordingly. Before doing so, we have to create a Table in ADX and configure a table Mapping where those additional properties come to play. Later in the post you get more information to this.

```python
Event_data.properties = {"Table":"TestTable", "IngestionMappingReference":"TestMapping", "Format":"JSON", "Encoding":"UTF-8"}
```


EventHub Properties are case sensitive! *Table* works, *table* does not!
{: .alert .alert-warning}

Events get send to the Event Hub correctly as this screenhot demonstrates.

![](/assets/img/20210820/eventhubmessages.png) 

Lets take a look on how to analyze the event body as well as the event custom properties.

## check data and examine file format (consumer client)

As we have ensured data in being sent correctly to event hubs, lets check if we can consume those events. For this purpose i have written a small script with the Azure Event Hubs consumer client to get those events which are send to event hub.

```python
async def on_event(partition_context, event):
    print("{}".format(event.body_as_json(encoding='UTF-8')))
    print({k.decode("utf-8"):v.decode("utf-8") for k,v in event.properties.items()})
    # Update the checkpoint so that the program doesn't read the events
    # that it has already read when you run it next time.
    await partition_context.update_checkpoint(event)
```
The whole script is published in my offical github account!

The script simply outputs two lines per event. The first one is the event body, the second one are the custom event properties.

![](/assets/img/20210820/resultconsumerclientjson.png)

Another method without scripting is the use of [Azure Service Bus Explorer](https://github.com/paolosalvatori/ServiceBusExplorer).
This tool can visualize the consumed events and offers much more insights into your service bus / event hub.

![](/assets/img/20210820/servicebusexploreroverview.png)

To check the event body as well as the custom event properties, you can just click on events and analyze each event separately.

![](/assets/img/20210820/servicebusexplorerevent.png)

After checking the compliance of our transmitted data, we can continue to prepare the ADX environment for the automatic ingestion of our data.

# Azure Data Explorer

To be able to automatically ingest data from azure event hub, we have to prepare the Azure Data Explorer environment. With this we make sure, that ADX knowx in which table we want to ingest data as well as it knows which data format our data has. This is a mandatory step to auto ingest data.

## creation of an ADX table

first step to be made is the creation of a table in which our data is ingested. We use KUSTO to create a table.

```kusto
.create table TestTable (TimeStamp: datetime, Temperature: float, Humidity: float, IoTDevice: string)
```
![](/assets/img/20210820/adxcreatetable.png)

## creation of a data mapping

Once this table is created, we can have to map the JSON data and values to the appropriate columns we have created. Also for this step we have to use KUSTO.

```kusto
.create table TestTable ingestion json mapping 'TestMapping' '[{"column":"TimeStamp", "Properties": {"Path": "$.timestamp"}} ,{"column":"Temperature", "Properties": {"Path":"$.temperature"}} ,{"column":"Humidity", "Properties": {"Path":"$.humidity"}} ,{"column":"IoTDevice", "Properties": {"Path":"$.iotdevice"}}]'
```
![](/assets/img/20210820/adxcreatemapping2.png)

For the creation of the mapping as well as the tablename in which data should be ingested, take a note about the custom event properties we send with each event!
{: .alert .alert-warning}

## creation of a ingestionbatching policy (optional)

Kusto attempts optimize throughput by batching small ingress data chunks together as they await ingestion. This sort of batching reduces the consumed resources by the ingestion process and doesn´t require post-ingestion resources to optimize those small data shards.

The downside of this batching before ingestion, which is the introduction of a forced delay, will cause a delay until data is ready to be queried.

With the ingestionbatching policy, you can configure parameters which provides the maximum forced delay to allow when batching small blobs together.

Batches are sealed when the first condition is met:

1. The total size of the batched data reaches the size set by the IngestionBatching policy.
2. The maximum delay time is reached
3. The number of blobs set by the IngestionBatching policy is reached

To speed-up the batching and thefore take effect on the ingestion time, I have decided for those settings. Batching with this settings happen every 30 secs, or 500 files or 1GB of data. (whatever comes first)<br>
This is just for my test environment and not recommended for productive use. 

```
.alter table TestTable policy ingestionbatching @'{"MaximumBatchingTimeSpan":"00:00:30", "MaximumNumberOfItems": 500, "MaximumRawDataSizeMB": 1024}'
```

To learn more about ingestionbatching policies, please check the references at the end of this post.

## creation of a user assigned managed identity

In the beginning of this blogpost I have mentioned that I have encountered a bug while working with the event hub ingestion. This bug was related to the system-assinged as well as user-assigned managed identities. In simple words, I was unable to ingest data when a system-managed identity was used! Meanwhile Microsoft has fixed this issue and you can use a user-assigned managed identity to do the authorization. Microsoft is working on the fix for the system-assigned managed identity as well. I will keep this post up-to-date to reflect the current state!
In this test scenario I have used a user assigned managed identity as Microsoft is still working on a fix.

## creation of an adx data connection

Within the portal we can use the *Data connections* function to create a event hub pipeline connection to automatically ingest data into the table we have created.<br>
In case you send one sort of events to just one event hub, it is not required to send custom eventhub properties containing the tablename, the mapping reference as well as the format.
In case you send multiple events from the same source and would like to ingest data into different tables, with different formats, than you would need to specify those custom events and send them along with the event itself!

![](/assets/img/20210820/adxdataconnectioncreate.png)

Especially take a note of the target table and default routing settings in the lower part of the screenshot in case you need them.

![](/assets/img/20210820/adxdataconnectioncreatedetail.png)

## check if events are ingested as supposed to

After you have created the data connection and sent some events, Azure Data Explorer should automatically pickup any new events and ingest them automatically into teh defined Table in your ADX. To check if data is being stored correctly, you can simply check this by browsing your table in ADX.

```KUSTO
TestTable
```

![](/assets/img/20210820/adxquerydata.png)

If you can see your simulated IoT Sensor Data or any other kind of data your automatic ingestion into ADX using Event Hub works!

# References

|Link|Description| 
|---|---|
|Azure Event Hub|[https://docs.microsoft.com/en-us/azure/event-hubs/event-hubs-about](https://docs.microsoft.com/en-us/azure/event-hubs/event-hubs-about)|
|Azure Data Explorer|[https://docs.microsoft.com/en-us/azure/data-explorer/data-explorer-overview](https://docs.microsoft.com/en-us/azure/data-explorer/data-explorer-overview)|
|Azure Event Hub Properties|- [https://docs.microsoft.com/en-us/dotnet/api/azure.messaging.eventhubs.eventdata.properties?view=azure-dotnet](https://docs.microsoft.com/en-us/dotnet/api/azure.messaging.eventhubs.eventdata.properties?view=azure-dotnet) <br> - [https://docs.microsoft.com/en-us/azure/data-explorer/ingest-data-event-hub-overview#ingestion-properties](https://docs.microsoft.com/en-us/azure/data-explorer/ingest-data-event-hub-overview#ingestion-properties)|
|ADX ingest Data from Azure Event Hub|[https://docs.microsoft.com/en-us/azure/data-explorer/ingest-data-event-hub-overview](https://docs.microsoft.com/en-us/azure/data-explorer/ingest-data-event-hub-overview)|
|Python Event Hub Samples| - [https://docs.microsoft.com/en-us/azure/data-explorer/ingest-data-event-hub-overview](https://docs.microsoft.com/en-us/azure/data-explorer/ingest-data-event-hub-overview) <br> - [https://docs.microsoft.com/en-us/samples/azure/azure-sdk-for-python/eventhub-samples/](https://docs.microsoft.com/en-us/samples/azure/azure-sdk-for-python/eventhub-samples/) <br> - [https://pypi.org/project/azure-eventhub/](https://pypi.org/project/azure-eventhub/)|
|Azure Batching Policy|[https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/batchingpolicy](https://docs.microsoft.com/en-us/azure/data-explorer/kusto/management/batchingpolicy)|
|GitHub Samples|[https://github.com/chwunder/blogsamples/tree/master/20210820_python_eventhub_adx](https://github.com/chwunder/blogsamples/tree/master/20210820_python_eventhub_adx)|