# Implementing Redis Enterprise TimeSeries Module (#3)

In this post we will cover the details and implementation of our problem area identified in the last post.  

We will walk through:
* Creating the proper key topology with labels and rules
* Ingesting Data into the Redis TimeSeries Database
* Visualizing and Extracting
  - Grafana
  - Node API
* Recap
  - What worked as expected
  - What surprises did we encounter (Good or Bad)

