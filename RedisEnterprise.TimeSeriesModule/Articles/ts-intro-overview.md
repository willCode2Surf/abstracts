# Getting to know Redis Enterprise TimeSeries Module Version (#2)

In this post we will cover all the details and considerations taken to make for a successful deployment of a product or service leveraging the Redis Enterprise TimeSeries Module.

We will walk through:

* Identifying a problem area that TimeSeries can fix
* Laying out the path to success by
  * Setting Up the proper instance in Azure
  * Understanding the TimeSeries Module features
  * Key/Data Topology
  * Planning for creating data sets that are downsampled for quick aggregation
  
## Where TimeSeries Module Fits In

Leveraging the TimeSeries Module you are likely engaged in some level of analytics.  Analytics is the power food for intelligent decision-making.  Finding a place where the TimeSeries Module can streamline operations in your enterprise will depend on existing infrastructure, pipelines, and legacy processes.  Never fear, working this technology into existing stacks is well-supported across various programming languages (Java, Go, Python, JavaScript/TypeScript, Rust, and Ruby).

For this scenario we will look at replacing some long-standing processes that create fixed cost and ongoing development costs.  We will also leverage Node with 
[RedisTimeSeries-JS](https://www.npmjs.com/package/redistimeseries-js).  We will also be propping up our Redis TimeSeries instance in [Azure](https://redislabs.com/cloud-partners/microsoft-azure/)  

## Success with Redis Enterprise TimeSeries Module

----------------------------------------------------------------

### Getting started in Azure

Since the first post in this series Redis Labs has fully launched Redis Enterprise in Azure!  

1. In the Azure portal, search for Azure Cache for Redis.
2. Click + Add.
3. Complete the initial screen with your needed details, select the Enterprise plan that fits your immediate needs.
4. Select appropriate public/private endpoint.
5. On the advanced screen we will select the modules that we need to use in this instance of Redis Enterprise.
6. Review and create the instance!
7. Once the instance has created, configure your [RedisInsight](https://redislabs.com/redis-enterprise/redis-insight/) so that you can see the new cluster.

----------------------------------------------------------------

### Understanding the TimeSeries Module Features

The Redis Enterprise platform brings with it the standard expectation of a tried-and-true tool in any modern stack but with scale in mind.  It is important to cover the features that can be leveraged and combined within the TimeSeries Module so early planning can take these into consideration.

There are a few key concepts that we need to expand on so that you can get most of the implementation.  Let’s cover the concepts that need to be understood before ingesting data.  (We will talk consumption in depth in the next post.)

1. Keys, this is not a new concept and are the way to sink data into the instance.  There are some attributes of keys that you can fine-tune data policies.
    * retention, give your data a TTL
    * labels, give context and additional attributes to your data
    * duplicate policy, determine what happens when you have data bumping into each other
    * compression policy

2. Aggregation, there are many out of the box.
    * AVG
    * SUM
    * MIN
    * MAX
    * RANGE
    * FIRST
    * LAST
    * STD.P
    * STD.S
    * VAR.P
    * VAR.S

3. Rules, make the data you are storing reactive to data ingest.
    * are applied to a specific key.
    * are predefined aggregation procedures
      * a destination key for the reactive data must be present
      * a predefined aggregation option must be provided
      * the time interval for the sampled data must be provided

4. Labels, the metadata option to give your data more context.
    * Label/Value collection that represents broader context of the underlying data
    * valuable at the digest/consumption layer to provide aggregation across keys by other similar characteristics

5. Ingest

----------------------------------------------------------------

### Key/Data Topology

If migrating from another system or starting from the ground up there will be some data points that need to be mapped to appropriate key structures.  Data points can be ingested as singular readings but are usually accompanied by additional data that gives a full picture of what just occurred.  Data ingest could come in at extremely high sampling rates (health devices, IoT sensors, weather apparatus) for the same system providing the data point, and we need to plan on how to handle this at scale so that we can surface downsampled data concisely without impacting performance.

Let's take a look at a sample log reading for a weather sensor.

```json5
{
  time: 1617293819263,
  temp: 55.425,
  wind: 13,
  windDirection: 13,
  windChill: 51,
  precipitation: 0,
  dewPoint: 20,
  humidity: .24,
  visibility: 10,
  barometer: 30.52,
  lat: 33.67,
  lon: 101.82,
  elevation: 3281,
  city: 2232,
  deviceId: 47732234
}
```

This sample log has many constant attributes, and some interesting data points that can vary with every reading.  If the scientific equipment produces a sample each second, then over a period of 24 hours we will receive 86,400 readings.  If we had 100 devices in a particular region providing realtime weather data, we could easily ingest close to 9M logs.  To perform some level of analytics (either visually or with additional ETL/ML pipelines) we need to downsample the footprint.  

Traditionally, one might sink this log reading into Cosmos or Mongo and aggregate on demand while handling the downsampling in code.  This is problematic, does not scale well, and is destined to fail at some point.  The better approach is to sink this data into Redis Enterprise using the TimeSeries Module.  The first thing to planning for a successful TimeSeries implementation is flattening your logs by identifying both reading data and metadata.  

With our sample log we can build a pattern for key definitions that we can sink readings into while attaching, define the metadata we will attach with the keys and resolving a key path.

Let's evaluate again with some context. 

```json5
{
  time: 1617293819263,  // timestamp   
  temp: 55.425,         // reading
  wind: 13,             // reading
  windDirection: 13,    // reading
  windChill: 51,        // reading
  precipitation: 0,     // reading
  dewPoint: 20,         // reading
  humidity: .24,        // reading
  visibility: 10,       // reading
  barometer: 30.52,     // reading
  lat: 33.67,           // meta-data
  lon: 101.82,          // meta-data
  elevation: 3281,      // meta-data
  city: 2232,           // meta-data
  deviceId: 47732234    // identification
}
```

With this context we can outline the keys we will need to represent each attribute in our log.  If it is a reading, it needs a key.  We will leverage the identifier combined with the readings to create the following keys based on this pattern.

```console
sensors:{deviceId}:{attribute}
```

With this topology defined we need to create keys for the attributes. 

| Attribute     | Key Definition                 |
|---------------|--------------------------------|
| temp          | sensors:47732234:temp          |
| wind          | sensors:47732234:wind          |
| windDirection | sensors:47732234:windDirection |
| windChill     | sensors:47732234:windChill     |
| precipitation | sensors:47732234:precipitation |
| dewPoint      | sensors:47732234:dewPoint      |
| humidity      | sensors:47732234:humidity      |
| visibility    | sensors:47732234:visibility    |
| barometer     | sensors:47732234:barometer     |

Before we create the keys, we need to identify the meta-data that we are going to associate with them as well as the TTL for series data and our duplicate policy.  TTL will default to 0 by default but in our case, we want to keep the data for 30 days and take the last reading if there was contention on a series. This

Based on our sample log we need to add the following labels to our keys.

| Attribute | Label          |
|-----------|----------------|
| lat       | lat 33.67      |
| lon       | lon 101.82     |
| elevation | elevation 3281 |
| city      | city 2232      |

Now let's create the keys.  You can always create the keys from the CLI in RedisInsight like so.

```console
>> TS.CREATE sensors:47732234:temp RETENTION 2592000000 DUPLICATE_POLICY LAST LABELS lat 33.67 lon 101.82 elevation 3281 city 2232
```

This works but to complete this for many sensors across many attributes we should lean in on some code to streamline this generation.  Using the redistimeseries-js library we can handle this programmatically.

```javascript
// set up our Redis Enterprise instance
const RedisTimeSeries = require('redistimeseries-js');
const {  Aggregation } = require('redistimeseries-js');
const options = {
  host: '{ENTER-YOUR-REDIS-ENTERPRISE-INSTANCE}',
  port: 10000,
  password: '{ENTER-YOUR-PASSWORD}'
}
const rtsClient = new RedisTimeSeries(options);
const retentionTime = 86400000 * 30;

// key generation
 let labels = {
      lat: 33.67,
      lon: 101.82,
      elevation: 3281,
      city: 2232
};
  
await rtsClient.create(key = `sensors:47732234:temp`, duplicatePolicy = 'LAST', labels = labels)
              .retention(retentionTime)
              .send()
              .catch((error) => {
                console.error(error);
              })
              .then(async (results) => {
                return true;
              });

```

We can validate that our key has been created by using the RedisInsight CLI command TS.INFO.

```console
>> TS.INFO sensors:47732234:temp
```
![TS.INFO](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/niqeczff6adzqv4sn53k.png)

----------------------------------------------------------------

### Making Data Reactive with rules

Aggregation out of the Redis Enterprise TimeSeries Module is very fast but can be optimized to provide even better performance if you know what kind of data summaries your data scientist, analyst, or systems need to make correct decisions.  Using our singular device that is producing real time weather information we have determined that there are a few intervals that will be extracted regularly.

* 5 minutes
* 1 hour
* 1 day

This is where we can define rules that react to data as it is ingested and downsampled using the built-in aggregation techniques to provide concise representation of many raw logs into a real time data point. 

NOTE:  there are a couple of things about rules that need to be made understood upfront.

* rules need an aggregation type.
* rules need a time interval to retain samples for.
* rules need downstream keys to sink the reactive data into.
* rules will not resample destination key data if rule is added after ingesting has begun, that is why we are defining these before ingesting data.

For this post we are going to focus on creating rules for the temp attribute.  We can create our 3 destination keys and 3 rules from the CLI as well.  We will define what aggregation type we are scoping to in the key structure with the interval it is concerned with.

Destination Keys

```console
>> TS.CREATE sensors:47732234:temp:avg:5min RETENTION 2592000000 DUPLICATE_POLICY LAST LABELS lat 33.67 lon 101.82 elevation 3281 city 2232
>> TS.CREATE sensors:47732234:temp:avg:1hr RETENTION 2592000000 DUPLICATE_POLICY LAST LABELS lat 33.67 lon 101.82 elevation 3281 city 2232
>> TS.CREATE sensors:47732234:temp:avg:1day RETENTION 2592000000 DUPLICATE_POLICY LAST LABELS lat 33.67 lon 101.82 elevation 3281 city 2232
```

Rules

```console
>> TS.CREATERULE sensors:47732234:temp sensors:47732234:temp:avg:5min AGGREGATION AVG 300000
>> TS.CREATERULE sensors:47732234:temp sensors:47732234:temp:avg:1hr AGGREGATION AVG 3600000
>> TS.CREATERULE sensors:47732234:temp sensors:47732234:temp:avg:1day AGGREGATION AVG 86400000
```

This can also be completed via code!

```javascript
 await rtsClient
      .createRule(`sensors:47732234:temp`, `pumps:47732234:temp:avg:5min`)
      .aggregation(Aggregation.AVG, 300000)
      .send().catch((error) => {
        console.error(error);
      }).then(async (results) => {
        return true;
      });
 await rtsClient
      .createRule(`sensors:47732234:temp`, `pumps:47732234:temp:avg:1hr`)
      .aggregation(Aggregation.AVG, 3600000)
      .send().catch((error) => {
        console.error(error);
      }).then(async (results) => {
        return true;
      });
await rtsClient
      .createRule(`sensors:47732234:temp`, `pumps:47732234:temp:avg:1day`)
      .aggregation(Aggregation.AVG, 86400000)
      .send().catch((error) => {
        console.error(error);
      }).then(async (results) => {
        return true;
      });
```

We can use TS.INFO to see our new rules just the same and now running it against our base key we will see the rules attached.

```console
>> TS.INFO sensors:47732234:temp
>> TS.INFO sensors:47732234:temp:avg:5min
>> TS.INFO sensors:47732234:temp:avg:1hr
>> TS.INFO sensors:47732234:temp:avg:1day
```
![TS.INFO.WITH.RULES](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/zs3c8ijcl8fwjrq1w7p7.png)

| 1 Day                                     | 1 Hour                                   | 5 Min                                     |
|-------------------------------------------|------------------------------------------|-------------------------------------------|
| ![TS.INFO.WITH.RULES](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/shi45nz3y2zd23c8jld3.png) | ![TS.INFO.WITH.RULES](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ukrzanjvh0f58ncxp1wm.png) | ![TS.INFO.WITH.RULES](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xmf90zpo9xvcvnphkdsc.png) |

Now that we have all of our keys and rules created, we can look into ingesting some sample data. 

----------------------------------------------------------------

### Ingest

For the purpose of this post, we will continue to focus on ingesting a single attribute from our sample log.  As data readings are coming from the system, we would have an API on the edge that would parse the log and update the TimeSeries instance.  We are going to simply ingest a few days worth of sample data for demo purposes around the temp attribute.  

Using the base code and redistimeseries-js module that we set up earlier we can add some functionality to simulate this process.  You can tweak the setInterval so that you can ingest different attributes or intervals. I ingested Jan 1 12AM through Jan 2 3:40AM with one minute interval logs.

```javascript
const moment = require('moment'); // add this to the project

let startTime = moment("2021-01-01");

const insertTS = async (key, timestamp, value, labels) => {
   await rtsClient.add(key, timestamp, value).labels(labels).send().catch((err) => { console.error(err);});
}
const randomTemp = (min, max) => {
  return Math.random() * (max - min) + min;
}

let labels = {
  lat: 33.67,
  lon: 101.82,
  elevation: 3281,
  city: 2232
};

setInterval(async () => {
  let timeStamp = startTime.add(1, 'minutes');
  console.info("⏰", timeStamp.format("LLLL"),  Date.parse(timeStamp.format()));
  return await insertTS('sensors:47732234:temp',Date.parse(timeStamp.format()), randomTemp(45, 77), labels);
}, 200);

```

Once ingest is complete we can return to RedisInsight and use TS.INFO to retrieve our base key details as well as take a peak at the destination keys that have the groomed data represented in real time!

```console
>> TS.INFO sensors:47732234:temp
>> TS.INFO sensors:47732234:temp:avg:5min
>> TS.INFO sensors:47732234:temp:avg:1hr
>> TS.INFO sensors:47732234:temp:avg:1day
```
![TS.INFO.WITH.RULES](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/3kgtbqw76wr1huxu9ge3.PNG)

| 1 Day - 1 Sample                                 | 1 Hour - 27 Samples                             | 5 Min - 332 Samples                              |
|--------------------------------------------------|-------------------------------------------------|--------------------------------------------------|
| ![TS.INFO.WITH.RULES](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/k4si3x48oz5ie7myevmc.png) | ![TS.INFO.WITH.RULES](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/axg866j0824daqqvi7f3.png) | ![TS.INFO.WITH.RULES](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/972a4bygihxvcbasd1lx.PNG) |

A quick look over the base key entries count (highlighted), and the destination keys gives us good context.  All the down sampled data is accurately represented by the time buckets we allocated for them to produce.

----------------------------------------------------------------

## Wrapping Up

Awesome! Hopefully at this point you have a grasp on the following concepts:

* Setting up an instance of Redis Enterprise in Azure with the TimeSeries Module 
* Data Topology and flattening logs to linear attributes as keys
* Rules for reactive data and destination keys
* Using RedisInsight and some basic commands for checking series state (TS.INFO, TS.CREATE, TS.CREATERULE)
* How to programmatically interact with the TimeSeries instance to ingest and process data to scale

In the next post we will cover the exciting parts that really show off the power of extracting and working with the data in the Redis Enterprise TimeSeries instance.  What to look forward to:

* Querying Data Out
* Exploring the destination key samples in detail
* Filtering with Labels
* API Code for composing complex objects across multiple keys
* Visualizing with Grafana
