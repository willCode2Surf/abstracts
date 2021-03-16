# Get excited for Redis Enterprise on Azure (#1)

Everyone that I have worked with loves Redis.  It is a highly effective tool used as a database, cache, pubsub, and many other things.  It is highly optimized and enjoyed success for 10 years in its space.  Redis's ease of use and intense scalability make it a must use in my opinion.  So how could Redis Labs improve Redis?  Take on a strategic partnership with Microsoft Azure and roll out the Redis Enterprise tier!

[Redis Labs and Microsoft Azure Partnership](https://azure.microsoft.com/en-us/blog/microsoft-and-redis-labs-collaborate-to-give-developers-new-azure-cache-for-redis-capabilities/)


## Redis Enterprise Better on Azure
Redis Labs thinks latency is the new downtime and they mean business with Redis Enterprise!  Redis Enterprise is about to exit preview and go mainstream on Azure for the masses.
* High Availability (99.99%)
* Geographically Distributed
* Module Extension
    * RedisTimeSeries
    * RedisSearch
    * RedisBloom
* Redis on Flash (RoF) over Azure NVMe
* Scaling
    * up to 15TB
    * 50K to 500K connections
* Secure
    * Azure Virtual Network, Private Link

## Usage
Currently in the preview version we have made heavy use of the TimeSeries Module, to replace organic processes.  There will be subsequent posts and code samples detailing our work around the TimeSeries Module but the others can be leveraged either in conjunction or independently.  Our use needs to support hundreds of thousands of IoT sensors providing real-time data. 
*   [Redis TimeSeries Module](https://redislabs.com/modules/redis-timeseries/)
*   [Redis Search](https://oss.redislabs.com/redisearch/)
*   [Redis Bloom](https://redislabs.com/modules/redis-bloom/)

## Resources
The developer ecosystem around Redis is fantastic.  Redis also has some tools that you should have already installed. 

* [RedisInsight](https://redislabs.com/redis-enterprise/redis-insight/ "The best tool for managing Redis and Redis Enterprise")

Node resources
* [Redis Modules (TypeScript)](https://github.com/danitseitlin/redis-modules-sdk/)
* [RedisTimeSeries-JS](https://www.npmjs.com/package/redistimeseries-js)

Planning
* [Redis TimeSeries Sizing Calculator](https://redislabs.com/modules/redis-timeseries/time-series-sizing-calculator/)

## Conclusion
As Redis Enterprise becomes generally available you owe it to yourself and organization to see if this improves your current Redis topology or if one of the new modules for Redis Enterprise can solve some business problems.

In the next post we will dive into TimeSeries and showcase some concepts including data topology, down sampling, covering the steps needed to be successful when planning for leveraging the Redis TimeSeries module to scale!

Thank You!