# Spark Notes
This document is incredibly incomplete.  It's meant as a rolling document for different ways to run Spark jobs on DC/OS.  It's mostly notes for myself, but I figure I can share with others as I learn.

Reminder: DC/OS essentially consists of many additional services packaged to interact with Apache Mesos, so all of these essentially boil down to different ways to run Spark on Apache Mesos, adjusted for the DC/OS environment.

This document currently focuses on using spark submit (i.e., not using an interactive shell.  Interactive shell may be added later).

This document indicates several different ways to run Spark jobs.

According to the Spark docs, there are two primary high-level ways of running Spark jobs against Mesos:

1. Client Mode: Start a Spark driver on the calling client machine.  We'll look at:

    1. Calling from outside cluster
    2. Calling from inside the cluster, using hdfs
    3. Calling from a Docker container running in the cluster
    4. Calling from Marathon

2. Cluster mode: Start a Spark driver as a task hosted in the cluster.  This requires a dispatcher.  We'll look at:

    1. Calling from dcos spark command line
    2. Calling from outside cluster
    3. Calling from inside cluster, using hdfs


First, let's look at running jobs without Mesos.

***

## Mode: Run using internal spark (no Mesos integration)

Setup (run as root):

```
# Download the Mesos-specific Spark binaries
curl -LO https://downloads.mesosphere.com/spark/assets/spark-2.2.0-bin-2.6.tgz
tar -xzvf spark-2.2.0-bin-2.6.tgz
mv spark-2.2.0-bin-2.6 /opt/spark
```

Set up path:
```
echo 'PATH=$PATH:/opt/spark/bin' >> ~/.bash_profile
sed -i '/export PATH/d' ~/.bash_profile 
echo "export PATH" >> ~/.bash_profile
```

Download examples jar file:
```
curl -LO https://downloads.mesosphere.com/spark/assets/spark-examples_2.11-2.0.1.jar
```

Run it:
```
spark-submit \
    --master local \
    --class org.apache.spark.examples.SparkPi \
    spark-examples_2.11-2.0.1.jar \
    30
```

So what is this actually doing?

1. We're downloading the Spark binaries, and extracting them
2. We're setting up our local path to point to the spark bin directory
3. We're downloading the spark example jar file
4. We're running spark-submit, with the following settings:
    * Use 'local' as our master (don't send jobs elsewhere)
    * Run class `org.apache.spark.examples.SparkPi`
    * Provide the class with the local jar file
    * Provide the parameter '30' to our job (indicating that we should run the example with 30 slices)

What could we do differently, for local run / what errors may we run into?

* We can add additional parameters to the executor configuration
* We could try to auto-fetch the url from https, but Spark by default only supports local, S3, and hdfs:
* We can change the parameters (this is trivial and boring)

***

## Mode: Client Mode with Mesos from Outside Cluster
#### This requires the following:

Download and extract spark binaries
```
# Download the Mesos-specific Spark binaries
curl -LO https://downloads.mesosphere.com/spark/assets/spark-2.2.0-bin-2.6.tgz
tar -xzvf spark-2.2.0-bin-2.6.tgz
mv spark-2.2.0-bin-2.6 /opt/spark
```

Set up path:
```
echo 'PATH=$PATH:/opt/spark/bin' >> ~/.bash_profile
sed -i '/export PATH/d' ~/.bash_profile 
echo "export PATH" >> ~/.bash_profile
```

Download examples jar file:
```
curl -LO https://downloads.mesosphere.com/spark/assets/spark-examples_2.11-2.0.1.jar
```

**New: Add required libraries**

You must obtain these files from somewhere (perhaps from an existing Mesos node):

* libaprutil-1.so.0
* libcrypto.so.1.0.0
* libssl.so.1.0.0
* libapr-1.so.0
* libsasl2.so.2
* libsvn_subr-1.so.1
* libsvn_delta-1.so.1
* libmesos.so

Put them all in one directory (e.g. /usr/lib/mesos), then export the path to that directory as an environment variable:

```
export LD_LIBRARY_PATH=/usr/lib/mesos
```

Then, compose your command like this (you must specify the correct IP addresses for your mesos zookeeper instances):

```
spark-submit \
  --class org.apache.spark.examples.SparkPi \
  --master mesos://zk://10.10.0.136:2181,10.10.0.50:2181,10.10.0.50:2181/mesos \
  --deploy-mode client \
  spark-examples_2.11-2.0.1.jar 40
```