---
layout: post
title: Automatically ingest Event Hub Data to Azure Data Explorer using Python
subtitle: learned the hard way!
thumbnail-img: "/assets/img/20210820/thumb.png"
tags: [Python, ADX, Azure Event Hub, ingest, pipeline, Azure]
comments: true
---

# ### LINKS, GITHUB REFERENCE - MAKE SURE TO INGEST BEFORE COMMIT


Recently i came across a preview feature which allows customers to ingest data automatically into azure data explorer (adx) by using Azure Event Hub.
As this feature is to the time of this blog post still in preview, i just wanted to know how it works and what the usecase as well as the benefit customers get by using it. 
Therefore i have used python to send batches of simulated IoT Temprature Sensor Data to Event Hub. This Data i would like to use in ADX to be automatically ingested.

# Design

In this test scenario i have used the following components:
* Python script to send batches of "raw" telemetry data (producer client)
* Python script to check transmitted values to the event hub (consumer client)
* Event Hub Namespace & Event Hub Instance (to send and read data from)
* a storage account to store checkpoint data from the consumer client
* Azure Data Explorer to ingest Data

# Event Hub data transmission

## send data to event hub (producer client)

First, i decided to send a batch of raw sensor data to the event hub. For this particular case i decided to use python.


```python
async def create_batch(producer):
    event_data_batch = await producer.create_batch()
    i = 0
    while i <= 100:
        device_id = random.randint(0,2)
        json_obj = {"timestamp": str(datetime.utcnow()),"temperature": round(random.uniform(16.8,37.5),3), "iotdevice": str((device_id))}
        string = json.dumps(json_obj)
        Event_data = EventData(body=string)
        Event_data.properties = {"Table":"TestTable", "IngestionMappingReference":"TestMapping", "Format":"JSON", "Encoding":"UTF-8"}
        event_data_batch.add(Event_data)
        i += 1
    print(event_data_batch)
    return event_data_batch
```
The whole script is published in my offical github account!

Event Properties in general are used for passing metadata associated with the event body during Event Hubs operations. Furthermore, we are using those Properties later for ingesting Data into the Azure Data Explorer Accordingly. Before doing so, we have to create a Table in ADX and configure a table Mapping where those additional properties come to play. Later in the post you get more information to this.

```python
Event_data.properties = {"Table":"TestTable", "IngestionMappingReference":"TestMapping", "Format":"JSON", "Encoding":"UTF-8"}
```


Events get send to the Event Hub correctly as this screenhot demonstrates.

![](/assets/img/20210820/eventhubmessages.png) 

To further check if the body as well as the properties are send correctly, let´s take a look and learn how to check this.

## check data and examine file format (consumer client)

As we have ensured data in being sent correctly to event hubs, lets check if we can consume those events.
For this purpose i have written a small script with the Azure Event Hubs consumer clients to get those events which are send to event hub.

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

To be able to automatically ingest data from azure event hub, we have to prepare the Azure Data Explorer environment. With this we make sure, than ADX know in which table we want to ingest data as well as it knows which data format our data has. This is a mandatory step to auto ingest data.

## creation of an ADX table

first step to be made is the creation of a table in which our data is ingested. We use KUSTO to create a table.

```kusto
.create table TestTable (TimeStamp: string, Temperature: float, IoTDevice: string)
```
![](/assets/img/20210820/adxcreatetable.png)

## creation of a data mapping

Once this table is created, we can have to map the JSON data and values to the appropriate columns we have created. Also for this step we have to use KUSTO.

```kusto
.create table TestTable ingestion json mapping 'TestMapping' '[{"column":"TimeStamp", "Properties": {"Path": "$.timestamp"}} ,{"column":"Temperature", "Properties": {"Path":"$.temperature"}} ,{"column":"IoTDevice", "Properties": {"Path":"$.iotdevice"}}]'
```

![](/assets/img/20210820/adxcreatemapping.png)


>For the creation of the mapping as well as the tablename in which data should be ingested, take a note about the custom event properties we send with each event!


## creation of an adx data connection

Within the portal we can use the *Data connections* function to create a event hub pipeline connection to automatically ingest data into the table we have created. In order to tell ADX in which columns our ingested data should be imported, we just reference our mapping we have created before!

For the creation of the mapping as well as the tablename in which data should be ingested, take a note about the custom event properties we send with each event!

![](/assets/img/20210820/adxdataconnectioncreate.png)

Especially take a not of the target table and default routing settings in the lower part of the screenshot. 

![](/assets/img/20210820/adxdataconnectioncreatedetail.png)


# References

|Link|Description| 
|---|---|
|Azure Event Hub|https://docs.microsoft.com/en-us/azure/event-hubs/event-hubs-about|
|Azure Data Explorer|https://docs.microsoft.com/en-us/azure/data-explorer/data-explorer-overview|
|Azure Event Hub Properties|https://docs.microsoft.com/en-us/dotnet/api/azure.messaging.eventhubs.eventdata.properties?view=azure-dotnet|
|ADX ingest Data from Azure Event Hub|https://docs.microsoft.com/en-us/azure/data-explorer/ingest-data-event-hub-overview|
|Python Event Hub Samples| - https://docs.microsoft.com/en-us/azure/data-explorer/ingest-data-event-hub-overview <br> - https://docs.microsoft.com/en-us/samples/azure/azure-sdk-for-python/eventhub-samples/ <br> - https://pypi.org/project/azure-eventhub/|


