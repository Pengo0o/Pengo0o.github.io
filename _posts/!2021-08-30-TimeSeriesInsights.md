---
layout: post
title: Time Series Insights - Event Hub Data Source
subtitle: exploration, usage, 
thumbnail-img: /assets/img/20210820/thumb.png
cover-img: /assets/img/20210820/banner.png
share-img: /assets/img/20210820/banner.png
tags: [Python, ADX, Azure Event Hub, ingest, pipeline, Azure]
comments: true
---


Based on my last Blogpost [Automatically ingest Event Hub Data to Azure Data Explorer using Python](https://christianwunderlich.com/2021-08-20-Python-EventHub-ADX-Ingest/) I wanted to use those simulated IoT Data to explore Azure Time Series Insights (TSI) capabilities. Time Series Insights collects, processes, stores, queries and visualizes data at Internet of Things (IoT) scale! With this service, you can analyze your operational data and unveil hidden trends and spotting anomalies.

With this post I would like to provide you insights into: <br>
- connecting a data source to TSI
- explore the data model in TSI
- configuration of custom data types
- using the REST API to modify the data model
- understanding interpolation

Of course this post does not use enterprise scale IoT data but to provide a common understanding how different configurations work, it is best to look at this first in a smaller scale. You can adopt accordingly!

# Azure Time Series Insights Environment

the setup of TSI will not be covered in this post as i assume either your TSI environment is already setup or in case it isn´t please refer to the official documentation: [Setup a TSI Environment using the Portal](https://docs.microsoft.com/en-us/azure/time-series-insights/how-to-create-environment-using-portal)

## selection of the Time Series ID

selecting an appropriate Time Series ID is a critical step while creating the environment. A Time Series ID is almost similar like choosing a Database partition key. It is an unique identifier for your time series model.

Also in this case, please check the offical documentation to see what possibilities you have by choosing the best-fitting TSID. [Choosing an appropriate TSID](https://docs.microsoft.com/en-us/azure/time-series-insights/how-to-select-tsid)

Once the environment is created with the specified TSID, you are not able to change it afterwards! choose wisely!
{: .alert .alert-warning}

# connecting a data source to TSI

TSI contains an ingestion engine to collect, process and store streaming time series data. TSI consume and stores data in near real time. Azure Time Series can currently have to streaming event sources: <br>
- Azure IoT Hub
- Azure Event Hub

As my python script in the previous blogpost have used Event Hub to send simulated sensor data, I will connect Event Hub to TSI.

While creating a time series insights environment, you can create an event source during the setup process or you can skip it and create it later. The screenshot shows how you could configure this at a later point in time.

![](/assets/img/20210830/tsiconnectdatasource.png) 

I would recommend you create a dedicated Event Hub Instance SAS policy as well as a dedicated consumer group for the TSI environment. Can be quite useful in case you have other event hub listeners configured. Keep attention especially to the *Start Time as well as the Timestamp property name* in the wizard. If you are sending a custom timestamp on your events to Event Hub or IoT Hub, you can use this custom data field. If you don´t specify it, event enqueue time is used.

Once you have created your event source, you should see data in TSI.

# Data Model in Azure Time Series Insights

In most cases, Events and Data you send to IoT Hub or Event Hub need to be transformed to gain valuable insights. This data computation can be done by using Azure TSI. <br>
The Azure TSI Data Model consist of three core components:
- Time Series Model instances
- Time Series Model hierarchies
- Time Series Model types

## Time Series Model instances

TSI instances are usually uniquely identified by deviceID or assetID, which are saved as time series IDs. In our case i have used a couple of simulated IoT Devices which are sending sensor data for temperature and humidity. Each simulated IoT device is represented as an instance within TSI.

Remember from the previous blogpost, my simulation is sending those information:

```json
{ body: '{"timestamp": "2021-08-20 23:36:22.937446", "temperature": 27.4, "humidity": 47.64, "iotdevice": "device1"}', properties: {'Table': 'TestTable', 'IngestionMappingReference': 'TestMapping', 'Format': 'json'} }<br>
{ body: '{"timestamp": "2021-08-20 23:02:53.748482", "temperature": 13.0, "humidity": 50.0, "iotdevice": "device2"}', properties: {'Table': 'TestTable', 'IngestionMappingReference': 'TestMapping', 'Format': 'json'} }
```

As I have set *iotdevice* as my Time Series ID, each IoT Device like *device0, device1, device2, ...* is a unique instance and therefore available in TSI.

![](/assets/img/20210830/tsiinstances.png)

After an Event is discovered and collected by the ingestion pipeline, an instance based on the TSID is automatically created.

## Time Series Model hierarchies

Hierarchies are being used to group and logically order instances. Think of having a bunch of sensors in your e.g house. Those do have continues unique identifiers like device0, device1, device2, device3, ...<br>
After installing them you might want to build a layout based on your floors and rooms.

<ul>
    <li>First Floor
        <ul>
            <li>Bathroom
                <ul>
                    <li> device0
                </ul>
            </li>
            <li> Bedroom
                <ul>
                    <li> device1</li>
                    <li> device2</li>
                </ul>
            </li>
        </ul>
    </li>
    <li>Ground Floor
        <ul>
        <li>Living Room
            <ul>
                <li> device3 </li>
            </ul>
        </li>
        <li> Kitchen
            <ul>
                <li> device4</li>
                <li> ....</li>
            </ul>
        </li>
    </li>
</ul>
<br>
You can create a hierarchy model called House which has the Floor as Level 1 and e.g Room as Level 2:

![](/assets/img/20210830/tsihierarchies.png)

After you have created a hierarchy, you can assign an instance a hierarchy object and fill out the instance fields!

![](/assets/img/20210830/tsiinstancefields.png)

Furthermore you can assign a name and a description to each instance.

![](/assets/img/20210830/tsiinstancenamedesc.png)

Once assigned, your data model will be applied accordingly! 

![](/assets/img/20210830/completehierarchy.png)

## Time Series Model types

Now as we have ordered our sensor data, remember we are still looking at raw non-transformed data. Types are used to computate data and add relations between different data to gain insights!

As types are quite complex and offer a lot of different methods for calculations, I would like to encourage you to take a look on the offical docs [Time Series Insights Types](https://docs.microsoft.com/en-us/rest/api/time-series-insights/dataaccessgen2/time-series-types/execute-batch#definitions).

In my case i would like to categorize the temperature in three zones based on its value!

- green (<= 20)
- yellow (> 20 but <= 30)
- red (> 30)

Furthermore i still want to see the collected values for temperature and humidity.

Start by creating a new type and give it a name as well as a short description.

![](/assets/img/20210830/createtypezonenamedesc.png)

In the next step, we create variables. Variables are something you can render on the timechart! If you had previously used the default type which for device0 as an example, you could choose to display temperature as well as humidity data on the chart. Once you create a new type only with our zone model, you can´t display the collected data for temperature as well as humidity anymore. Therefore we create those non-computated variables as well to still be able to display the raw data.

![](/assets/img/20210830/tsiinstancetempzoneraw.png)

Do the same for the humidity data as well.

Next is to create as an example the temeprature zones shown above! Create a new variable, give it the name *TempZone* and select Kind *Categorial*. If we would have just raw values read from the sensor like 1,2,3 - we could simply provide the values and specify a appropriate color. In our case, we would like to do something a bit more complex to assign temp ranges, therefore switch for the value configuration to "custom" and provide use something like this.

```kusto
iff(tolong($event.temperature.Double) <= 30, iff(tolong($event.temperature.Double) <= 20, 'cold', 'normal'), iff(tolong($event.temperature.Double) > 30, 'hot', 'normal'))
```

![](/assets/img/20210830/typecreation1.png)
![](/assets/img/20210830/typecreation2.png)

Last but not least, let´s change the Type for device1 from *DefaultType* to the one we have created earlier.

![](/assets/img/20210830/tsiinstancechangetype.png)

once applied, you are now able to not only present our newly created variable, but also the "default" ones.

![](/assets/img/20210830/presentvariables.png)



|Link|Description| 
|---|---|
|Azure Time Series event sources|[https://docs.microsoft.com/en-us/azure/time-series-insights/concepts-streaming-ingestion-event-sources](https://docs.microsoft.com/en-us/azure/time-series-insights/concepts-streaming-ingestion-event-sources)|
|Azure Time Series instances|[https://docs.microsoft.com/en-us/azure/time-series-insights/concepts-model-overview#time-series-model-instances](https://docs.microsoft.com/en-us/azure/time-series-insights/concepts-model-overview#time-series-model-instances)|
|Azure Time Series hierarchies|[https://docs.microsoft.com/en-us/azure/time-series-insights/concepts-model-overview#time-series-model-hierarchies](https://docs.microsoft.com/en-us/azure/time-series-insights/concepts-model-overview#time-series-model-hierarchies)|
|Azure Time Series Types|[https://docs.microsoft.com/en-us/rest/api/time-series-insights/dataaccessgen2/time-series-types/execute-batch#definitions](https://docs.microsoft.com/en-us/rest/api/time-series-insights/dataaccessgen2/time-series-types/execute-batch#definitions)|