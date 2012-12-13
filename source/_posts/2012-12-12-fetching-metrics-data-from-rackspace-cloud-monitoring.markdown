---
layout: post
title: "Fetching Metrics Data From Rackspace Cloud Monitoring"
date: 2012-12-12 15:02
comments: true
author: Chris Snell
categories: 
- Cloud Monitoring
---
*Guest post contributed by Chris Snell, from the Rackspace Enterprise Technical Sales team. Chris can be found at <http://output.chrissnell.com>

In August, Rackspace launched a cloud-based monitoring service that gave its users the ability to monitor infrastructure without having to invest time and servers in an on-premesis monitoring environment.  The initial release of Cloud Monitoring was limited to service and server availability checks, run centrally, that could trigger scriptable alert actions when a fault is detected.  Last week, the Rackspace Monitoring team unveiled a [new software agent](http://www.rackspace.com/knowledge_center/article/install-the-cloud-monitoring-agent) that can gather performance metrics directly on the server.  The administrator installs the agent on their servers and the agent begins reporting metrics like CPU, disk, and RAM utilization back to the Rackspace Monitoring cloud.  The current metrics data then becomes visible on the MyCloud control panel:

<center>
![Server Metrics](/images/servermetrics.png)
</center>

I recently wrote a patch for Rackspace Cloud Monitoring's [Python binding](https://github.com/racker/rackspace-monitoring) that adds the ability to fetch current and historical metrics data collected by the agent from the Cloud Monitoring API.   Using the library to fetch this data is simple.  You'll need the IDs of the entity (the host being monitored), the check (CPU, memory, etc.), and the metric name, which is specific to the type of check.

Here is some sample code that traverses through your entities, checks, and metrics, printing 30 minutes of data points as captured by the agent. 

{% codeblock lang:python %}
#!/usr/bin/python
 
import os
import datetime

from rackspace_monitoring.providers import get_driver
from rackspace_monitoring.types import Provider
 
#
# Configuration
#
RACKSPACE_USER=os.environ['OS_USERNAME']
RACKSPACE_KEY=os.environ['OS_PASSWORD']

start = datetime.datetime.now() - datetime.timedelta(minutes=30)
starttime = int(start.strftime("%s")) * 1000
endtime = int(datetime.datetime.now().strftime("%s")) * 1000

 
Cls = get_driver(Provider.RACKSPACE)
driver = Cls(RACKSPACE_USER, RACKSPACE_KEY)

# Get a list of entities for this account
entities = driver.list_entities()

# Iterate through the entities
for entity in entities:
    print entity.id + " ---> " + entity.label

    # Get a list of checks for this entity
    checks = driver.list_checks(entity=entity)

    # Iterate through the checks
    for check in checks:
        print "    check --->  " + check.id + "  [" + check.label + "]"

        # Get a list of metrics for this check
        metrics = driver.list_metrics(entity_id=entity.id, check_id=check.id)

        # ...and iterate through it...
        for metric in metrics:
           print "        metric --->  " + metric.name 

           # Fetch five datapoints from a 30-minute span
           datapoints = driver.fetch_data_point(entity_id=entity.id, check_id=check.id, metric_name=metric.name, from_timestamp = starttime, to_timestamp = endtime, points=5)

           # ...and iterate through them
           for point in datapoints:
               print "                datapoint: " + str(point.average)
{% endcodeblock %}


NOTE: This script makes many API calls to fetch this data.  The Cloud Monitoring API limits you to 50,000 API requests every 24 hours for this type of data.  This is generous enough to pull data points for a large number of metrics but for your production use, I would recommend storing things like entity IDs, check IDs, and metric names in a local database since they are unlikely to change, as opposed to calling the API every time you need to lookup one of them.
